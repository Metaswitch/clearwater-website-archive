Using start-stop-daemon with the Python Interpreter
------------------------------------------------------------

[start-stop-daemon](http://manpages.ubuntu.com/manpages/trusty/man8/start-stop-daemon.8.html) is, as the name suggests, a tool for starting and stopping daemons (long-running Unix server processes). It has a couple of nice features:

*   It can start a process running, put it into the background and write a pidfile for it – this means we don’t have to write our own code to fork and daemonize our processes
*   It can handle a lot of the security configuration and other setup needed when starting a process – such as switching to an unprivileged user, chrooting the process, and setting nice/ionice levels
*   It can do a phased stop of a program – for example, attempting to send SIGTERM initially, then sending SIGKILL if the program hasn’t exited within 30 seconds
*   It offers several options for stopping the right process (matching by process ID, by name, etc.)

It’s this last bullet that we’ve found tricky to get right, particularly with our Python processes. We’ve recently seen and tackled a range of bugs in this area (e.g. [https://github.com/Metaswitch/ellis/pull/106](https://github.com/Metaswitch/ellis/pull/106), [https://github.com/Metaswitch/clearwater-etcd/issues/218](https://github.com/Metaswitch/clearwater-etcd/issues/218)) – and we’ve now worked out the right tricks to make start-stop-daemon do the right thing. In this blog post, I’ll talk about configuring start-stop-daemon to accurately kill Python daemons, especially when using [Python multiprocessing](http://toastdriven.com/blog/2008/nov/11/brief-introduction-multiprocessing/). In a later blog post, I’ll talk about the best way to use pidfiles to prevent start-stop-daemon from losing track of a process.

## Identifying your processes

start-stop-daemon has four options for identifying a process (which is useful both when stopping a process, to send SIGTERM to the right one, and also when starting a process – start-stop-daemon will check if a matching process is already running, and not start a conflicting one if so):

*   match by pidfile (--pidfile) – if your process has written its process ID to a file, start-stop-daemon can read that file and use it to target a specific process
*   match by executable name (--exec)
*   match by process name (--name)
*   match by the user ID running the process (--user)

## The problem with interpreters

To quote the [start-stop-daemon manpage](http://manpages.ubuntu.com/manpages/trusty/man8/start-stop-daemon.8.html), when talking about the --exec option: “Note: this might not work as intended with interpreted scripts, as the executable will point to the interpreter”. The problem here is that for one of our C++ programs, the executable (e.g. /usr/share/clearwater/bin/sprout) uniquely identifies its process. For our Python processes, however, we run them as “/usr/share/clearwater/ellis/env/bin/python –m metaswitch.ellis.main” – which means that the executable is “/usr/share/clearwater/ellis/env/bin/python”. But “/usr/share/clearwater/ellis/env/bin/python” is a general-purpose Python interpreter, which might be running other scripts, and if start-stop-daemon tries to kill every process with the executable “/usr/share/clearwater/ellis/env/bin/python”, it will kill those scripts too. In particular, one bug we’ve seen is that we ran “/usr/share/clearwater/ellis/env/bin/python create\_numbers.py” to allocate numbers on the system – but this sometimes got killed and left the system without any phone numbers, because restarting the Ellis process also killed the “create\_numbers.py” script. (The problem is mitigated slightly because we use Python [virtual environments](https://virtualenv.readthedocs.org/en/latest/), with their own copy of the Python interpreter. If we were running “/usr/bin/python -m metaswitch.ellis.main”, it would be even worse – start-stop-daemon would potentially kill **all** the Python scripts running on the system.) At first glance, none of the other options completely solve this problem:

*   matching on a user would work if we were running as a specialist user – but some of our processes currently run as the root user (such as clearwater-cluster-manager, which needs the ability to edit system-wide Cassandra config files)
*   matching on a pidfile would be OK if we just had a single process, but to use all our CPU cores, we often use [Python multiprocessing](http://toastdriven.com/blog/2008/nov/11/brief-introduction-multiprocessing/) – so starting our Crest or Ellis daemons can actually create several Python processes, each with separate PIDs
*   the process name is typically just the last part of the executable name – e.g. for “/usr/share/clearwater/ellis/env/bin/python” it’s “python”. You can check this by looking at /proc/PID/status

Luckily, there’s a little-known system call called prctl, which makes the process name and the pidfile approaches much more usable for these purposes.

## The secrets of prctl

[prctl](http://manpages.ubuntu.com/manpages/trusty/man2/prctl.2.html) is the process control system call, which lets you query and set various low-level properties of the process. (This is also available through a Python module wrapper - [https://pypi.python.org/pypi/prctl](https://pypi.python.org/pypi/prctl)). Examples of properties you can check and modify include:

*   PR\_GET\_DUMPABLE – if this process crashes, does it write out its memory to a core file?
*   PR\_CAPBSET\_READ – does this process have a particular privileged capability?
*   PR\_GET\_TSC – can this process read the timestamp counter?

The key ones we’re interested in, though, are PR\_SET\_NAME and PR\_SET\_PDEATHSIG. PR\_SET\_NAME allows you to change the current name of the process. This means that we can switch the Ellis process’s name from being “python” to something unique like “ellis” – which is much better for start-stop-daemon’s --name matching. PR\_SET\_PDEATHSIG lets you tell the kernel to send a UNIX signal (e.g. SIGTERM, SIGHUP, etc.) to your process when its parent process dies. Calling this in our child processes solves the problem with using start-stop-daemon’s --pidfile option – while the pidfile only contains the PID of your first Python process, and so start-stop-daemon will only terminate that, the kernel will then send SIGTERM to the other Python processes and clean them up.

## Putting it all together

The two prctl-based approaches are mutually exclusive – you can either kill all the processes which have renamed themselves to “ellis”, or kill just the one process recorded in the pidfile and rely on the kernel to send SIGTERM to all its child processes – so we’ve had to pick one to use. We’ve gone with the pidfile-based approach, just for consistency – we use pidfiles to monitor our C++ processes (where we can use threads for parallelism, and don’t suffer the same issues as with multiprocessing), and we need to create them anyway to integrate with monit for monitoring. That said, we do also change the process name – this makes it simpler to identify our Python processes in diagnostics. So [in Ellis’ main.py file](https://github.com/Metaswitch/ellis/blob/29cb4816a4177f8ea138151d13c56ca4afc143a1/src/metaswitch/ellis/main.py#L111), we use the Python prctl module to call through to the prctl system call:

    prctl.prctl(prctl.NAME, "ellis")
    prctl.prctl(prctl.PDEATHSIG, signal.SIGTERM)

And in our init.d script, we call start-stop-daemon and pass a pidfile:

    start-stop-daemon --stop --quiet --retry=TERM/30/KILL/5 --pidfile $PIDFILE --user $USER --exec $DAEMON

This kills all the Ellis processes we started for Python multiprocessing, but (because of the specific process name check) won’t kill other scripts being run by the /usr/share/clearwater/ellis/env/bin/python interpreter, like create\_numbers.py. We’ve found this to be a useful process management strategy, which we’re applying to all our processes. If you’d like more detail on this, you can see our full init.d script [here](https://github.com/Metaswitch/ellis/blob/29cb4816a4177f8ea138151d13c56ca4afc143a1/debian/ellis.init.d).
