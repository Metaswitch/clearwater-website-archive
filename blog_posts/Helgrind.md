Analysing Threading Issues with Helgrind
----------------------------------------
Clearwater forms part of an IMS core network. As such it is important that it is stable and does not crash at high levels of load. We regularly test Clearwater with lots of call and registration traffic to flush out bugs that are only hit at high load. However, this is not always the most efficient way of finding and analysing these bugs. Some bugs are only hit incredibly rarely, and/or under certain conditions (such as when a TCP connection is congested). When problems are hit, they are often difficult to diagnose as only limited diagnostics are available.

One of the most common categories of bug that is only hit at load is “threading bugs”. Recently the Clearwater team has started using [Helgrind](http://valgrind.org/docs/manual/hg-manual.html) – a part of the [Valgrind](http://valgrind.org/) toolset – to identify potential threading bugs in our code and, if necessary, fix them. This blog post will explain what Valgrind and Helgrind are, how to run Clearwater under Helgrind, and our results so far.

##What is Valgrind?
To quote from the Valgrind website: “Valgrind is an instrumentation framework for building dynamic analysis tools”. It works by instrumenting the executable’s machine code before it is run on the CPU. Valgrind comes with a variety of different tools, and the instrumentation that is added depends on the tool you have selected. Because of how it works, Valgrind can analyse production binaries - there is no need to create a special debug build of your code, or preload any special analysis libraries at runtime. Furthermore, Valgrind instruments _all_ code that is used as part of your executable, including code from 3rd party libraries.

The disadvantage of Valgrind is that it makes your program run slower – between 5x – 100x. However, this is usually not a problem when testing as you can just reduce the amount of load Clearwater is subjected to.

##What is Helgrind?
Helgrind is one of the tools that comes with Valgrind. It analyses a program’s threading behaviour, and can identify three types of bug:

* Incorrect uses of threading APIs.
* Potential deadlocks.
* Data races.

Incorrect API use is noteworthy but not particularly interesting, so I’ll focus on the other two categories in the rest of this blog post.

## Potential Deadlocks
A [deadlock](https://en.wikipedia.org/wiki/Deadlock) normally occurs when two threads are waiting for the other to finish some work, and thus neither ever does. The classic example is when two threads try to acquire two locks, but in different orders:

* Thread 1 holds lock A
* Thread 2 holds lock B
* Thread 1 tries to acquire lock B, but can’t because thread 2 already holds it
* Thread 2 tries to acquire lock A, but can’t because thread 1 already holds it.

At this point neither thread can make any progress, and they both stop. This is a simple example of a more general case, and deadlocks can occur when there are more than 2 threads and/or more than 2 locks.

Helgrind spots potential deadlocks by tracking when any lock is acquired or released. It builds up a picture of which locks are taken in which order and reports on any inconsistent orderings. This allows Helgrind to spot potential deadlocks without the program having to actually hit the deadlock.

When Helgrind spots a potential deadlock:

* If only two locks were involved it prints four call stacks. Two of them show the locks being taken in one order on one thread and the other two show the locks being taken in the other order on a different thread. From this it’s pretty easy to determine what the problem is.
* If three or more locks were involved it prints out two call stacks showing two of the locks being taken by one thread. A fair amount of code reading is normally needed to identify the cause of the potential deadlock.

An example of Helgrind’s output can be found at the end of this blog post.

## Data Races
A data race occurs when two or more threads could potentially access a memory location at the same time, and where at least one of them is modifying the contents of the memory. Data races may cause the program’s objects to become temporarily or permanently inconsistent, which can lead to crashes.

To avoid a data-race the code must ensure only one thread can access the object at a time. For example, you might protect it with a lock (and have a rule that the lock must be held when accessing the object), or pass it between threads when necessary (and have a rule that only the owning thread may access it). However these rules are entirely a convention and most languages (including C++) are not able to enforce them.

To spot data races, Helgrind tracks all memory access the program makes (!), plus all CPU-level thread synchronization primitives. If it sees two memory accesses (where one or both is a write) from two different threads without a synchronization between those two threads in-between, Helgrind flags this as a potential data race.

When Helgrind spots a data race it reports two call stacks. Both call stacks show the same memory location being accessed. The stacks are from threads which accessed the memory without any intervening synchronization. See the end of this blog post for an example.

##Running Clearwater Under Helgrind
Valgrind is most useful for analysing native binaries. For Clearwater this means the core sprout, bono, homestead and ralf executables (homer, homestead-prov and ellis are all written in python). So far we have focussed our testing on [Sprout](https://github.com/Metaswitch/sprout/). Sprout is the only production-ready part of Clearwater that uses SIP which, due to the complex nature of [SIP](https://en.wikipedia.org/wiki/Session_Initiation_Protocol), makes it more likely to have threading bugs.

We used the Clearwater SIP stress package to drive registration and call traffic through Sprout, while running the sprout process under Helgrind. Our public documentation has instructions for:

* [Stress testing Clearwater](http://clearwater.readthedocs.io/en/latest/Clearwater_stress_testing.html). Bear in mind that Valgrind slows sprout down, so be sure to reduce the number of subscribers the SIPp node simulates. I’d recommend starting at 1000 subscribers and only increasing the load if Sprout’s CPU utilization is below 60%.
* [Running Clearwater under Valgrind](http://clearwater.readthedocs.io/en/latest/Debugging_Bono_Sprout_and_Homestead_with_GDB_and_Valgrind.html). The example shows running sprout under massif (another Valgrind tool), but if you want to use Helgrind you just need to change the tool specified on the command line, and also pass any additional general [Valgrind options](http://valgrind.org/docs/manual/manual-core.html#manual-core.options) or [Helgrind-specific options](http://valgrind.org/docs/manual/hg-manual.html#hg-manual.options) you want.

Although Helgrind can spot thread errors without actually hitting them, you do need to at least execute some code for Helgrind to check it. For example it will only spot errors in application server processing if your test setup includes some application servers and causes the AS processing to be invoked!

## Results
 When we ran Sprout under Helgrind we found a total of 5 potential deadlocks and 4 data races. None had these had been observed before (either by the Clearwater team or any of our users) so the analysis was certainly useful! So far we’ve fixed some of the bugs, and have plans on how to address the rest, but it’s a work in progress.

One noteworthy point is that some of our solutions to potential deadlocks involve leveraging sprout’s threading model. Sprout has one “transport thread” that is responsible for receiving incoming messages and parsing them, but it quickly hands messages off to a pool of “worker threads” that do the rest of the processing. The transport thread also processes timer pops. We’re eliminating some of the potential deadlocks by moving the some processing onto the transport thread so that only one thread ever does this processing. To check that this is safe we’ve replaced the locks with error logs that are triggered if the code is executed on a worker thread. We’re also planning to continue running tests with Helgrind to check we haven’t turned a potential deadlock into a data race by removing a lock.

Overall we’ve been really impressed with Helgrind – it’s easy to set up and run, and is an efficient way of spotting threading bugs. It makes data races and simple deadlocks much easier to diagnose (though deadlocks involving 3+ locks are still difficult to debug seeing as Helgrind doesn’t output full information about the lock orderings). Unfortunately Helgrind doesn’t make fixing them any easier!

### Example Helgrind Output

Here is some example output from running Helgrind against Sprout, so you can see what the diagnostics look like.

#### Potential Deadlock
    ==17288== Thread #118: lock order "0x14BDCED0 before 0x16E4BD00" violated
    ==17288==
    ==17288== Observed (incorrect) order is: acquisition of lock at 0x16E4BD00
    ==17288==    at 0x4C32536: pthread_mutex_lock (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6D482D: pj_mutex_lock (os_core_unix.c:1243)
    ==17288==    by 0x6DBE0F: pj_lock_acquire (lock.c:180)
    ==17288==    by 0x65F82E: destroy_transport (sip_transport.c:1087)
    ==17288==    by 0x65FADF: pjsip_transport_destroy (sip_transport.c:1179)
    ==17288==    by 0x65F437: transport_idle_callback (sip_transport.c:967)
    ==17288==    by 0x6E3E43: pj_timer_heap_poll (timer.c:643)
    ==17288==    by 0x656B4E: pjsip_endpt_handle_events2 (sip_endpoint.c:711)
    ==17288==    by 0x656CBE: pjsip_endpt_handle_events (sip_endpoint.c:768)
    ==17288==    by 0x4E0F2E: pjsip_thread_func(void*) (stack.cpp:163)
    ==17288==    by 0x6D3B8F: thread_main (os_core_unix.c:523)
    ==17288==    by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==
    ==17288==  followed by a later acquisition of lock at 0x14BDCED0
    ==17288==    at 0x4C32536: pthread_mutex_lock (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x59E706: ConnectionTracker::connection_state_update(pjsip_transport*, pjsip_transport_state) (connection_tracker.cpp:93)
    ==17288==    by 0x661864: tp_state_callback (sip_transport.c:2113)
    ==17288==    by 0x65F944: destroy_transport (sip_transport.c:1118)
    ==17288==    by 0x65FADF: pjsip_transport_destroy (sip_transport.c:1179)
    ==17288==    by 0x65F437: transport_idle_callback (sip_transport.c:967)
    ==17288==    by 0x6E3E43: pj_timer_heap_poll (timer.c:643)
    ==17288==    by 0x656B4E: pjsip_endpt_handle_events2 (sip_endpoint.c:711)
    ==17288==    by 0x656CBE: pjsip_endpt_handle_events (sip_endpoint.c:768)
    ==17288==    by 0x4E0F2E: pjsip_thread_func(void*) (stack.cpp:163)
    ==17288==    by 0x6D3B8F: thread_main (os_core_unix.c:523)
    ==17288==    by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==
    ==17288== Required order was established by acquisition of lock at 0x14BDCED0
    ==17288==    at 0x4C32536: pthread_mutex_lock (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x59E8FF: ConnectionTracker::connection_active(pjsip_transport*) (connection_tracker.cpp:120)
    ==17288==    by 0x4E0EF3: on_rx_msg(pjsip_rx_data*) (stack.cpp:115)
    ==17288==    by 0x656F38: pjsip_endpt_process_rx_data (sip_endpoint.c:885)
    ==17288==    by 0x657245: endpt_on_rx_msg (sip_endpoint.c:1032)
    ==17288==    by 0x660F07: pjsip_tpmgr_receive_packet (sip_transport.c:1782)
    ==17288==    by 0x6664A6: on_data_read (sip_transport_tcp.c:1396)
    ==17288==    by 0x6D869B: ioqueue_on_read_complete (activesock.c:492)
    ==17288==    by 0x6D076A: ioqueue_dispatch_read_event (ioqueue_common_abs.c:589)
    ==17288==    by 0x6D2A1F: pj_ioqueue_poll (ioqueue_epoll.c:810)
    ==17288==    by 0x656C12: pjsip_endpt_handle_events2 (sip_endpoint.c:740)
    ==17288==    by 0x656CBE: pjsip_endpt_handle_events (sip_endpoint.c:768)
    ==17288==
    ==17288==  followed by a later acquisition of lock at 0x16E4BD00
    ==17288==    at 0x4C32536: pthread_mutex_lock (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6D482D: pj_mutex_lock (os_core_unix.c:1243)
    ==17288==    by 0x6DBE0F: pj_lock_acquire (lock.c:180)
    ==17288==    by 0x661911: pjsip_transport_add_state_listener (sip_transport.c:2138)
    ==17288==    by 0x59E96F: ConnectionTracker::connection_active(pjsip_transport*) (connection_tracker.cpp:130)
    ==17288==    by 0x4E0EF3: on_rx_msg(pjsip_rx_data*) (stack.cpp:115)
    ==17288==    by 0x656F38: pjsip_endpt_process_rx_data (sip_endpoint.c:885)
    ==17288==    by 0x657245: endpt_on_rx_msg (sip_endpoint.c:1032)
    ==17288==    by 0x660F07: pjsip_tpmgr_receive_packet (sip_transport.c:1782)
    ==17288==    by 0x6664A6: on_data_read (sip_transport_tcp.c:1396)
    ==17288==    by 0x6D869B: ioqueue_on_read_complete (activesock.c:492)
    ==17288==    by 0x6D076A: ioqueue_dispatch_read_event (ioqueue_common_abs.c:589)
    ==17288==
    ==17288==  Lock at 0x14BDCED0 was first observed
    ==17288==    at 0x4C321AA: pthread_mutex_init (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x59E485: ConnectionTracker::ConnectionTracker(ConnectionsQuiescedInterface*) (connection_tracker.cpp:54)
    ==17288==    by 0x4E249E: init_stack(std::string const&, std::string const&, int, int, int, int, std::string const&, std::string const&, std::string const&, std::string const&, std::string const&, std::string const&, SIPResolver*, int, int, int, int, int, int, QuiescingManager*, std::string const&) (stack.cpp:865)
    ==17288==    by 0x4D90B4: main (main.cpp:1790)
    ==17288==  Address 0x14bdced0 is 0 bytes inside a block of size 104 alloc'd
    ==17288==    at 0x4C2C500: operator new(unsigned long) (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x4E247F: init_stack(std::string const&, std::string const&, int, int, int, int, std::string const&, std::string const&, std::string const&, std::string const&, std::string const&, std::string const&, SIPResolver*, int, int, int, int, int, int, QuiescingManager*, std::string const&) (stack.cpp:865)
    ==17288==    by 0x4D90B4: main (main.cpp:1790)
    ==17288==  Block was alloc'd by thread #1
    ==17288==
    ==17288==  Lock at 0x16E4BD00 was first observed
    ==17288==    at 0x4C321AA: pthread_mutex_init (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6D4543: init_mutex (os_core_unix.c:1139)
    ==17288==    by 0x6D4747: pj_mutex_create (os_core_unix.c:1192)
    ==17288==    by 0x6DBBBC: create_mutex_lock (lock.c:75)
    ==17288==    by 0x6DBC49: pj_lock_create_recursive_mutex (lock.c:96)
    ==17288==    by 0x6649F3: tcp_create (sip_transport_tcp.c:625)
    ==17288==    by 0x665D93: on_accept_complete (sip_transport_tcp.c:1116)
    ==17288==    by 0x6D911B: ioqueue_on_accept_complete (activesock.c:866)
    ==17288==    by 0x6D0560: ioqueue_dispatch_read_event (ioqueue_common_abs.c:474)
    ==17288==    by 0x6D2A1F: pj_ioqueue_poll (ioqueue_epoll.c:810)
    ==17288==    by 0x656C12: pjsip_endpt_handle_events2 (sip_endpoint.c:740)
    ==17288==    by 0x656CBE: pjsip_endpt_handle_events (sip_endpoint.c:768)
    ==17288==  Address 0x16e4bd00 is 5,968 bytes inside a block of size 6,144 alloc'd
    ==17288==    at 0x4C2BFA0: malloc (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6D6896: default_block_alloc (pool_policy_malloc.c:46)
    ==17288==    by 0x6DDD2D: pj_pool_create_block (pool.c:60)
    ==17288==    by 0x6DDEDF: pj_pool_allocate_find (pool.c:138)
    ==17288==    by 0x6DDBC7: pj_pool_alloc (pool_i.h:62)
    ==17288==    by 0x6DDC00: pj_pool_calloc (pool_i.h:69)
    ==17288==    by 0x6639D0: pj_pool_zalloc (pool.h:476)
    ==17288==    by 0x664913: tcp_create (sip_transport_tcp.c:610)
    ==17288==    by 0x665D93: on_accept_complete (sip_transport_tcp.c:1116)
    ==17288==    by 0x6D911B: ioqueue_on_accept_complete (activesock.c:866)
    ==17288==    by 0x6D0560: ioqueue_dispatch_read_event (ioqueue_common_abs.c:474)
    ==17288==    by 0x6D2A1F: pj_ioqueue_poll (ioqueue_epoll.c:810)
    ==17288==  Block was alloc'd by thread #118

### Data Race

    ==17288== Possible data race during write of size 4 at 0x16FAB8B0 by thread #118
    ==17288== Locks held: 1, at address 0x121F6730
    ==17288==    at 0x666720: on_connect_complete (sip_transport_tcp.c:1493)
    ==17288==    by 0x6D925B: ioqueue_on_connect_complete (activesock.c:926)
    ==17288==    by 0x6D00D6: ioqueue_dispatch_write_event (ioqueue_common_abs.c:280)
    ==17288==    by 0x6D2A4E: pj_ioqueue_poll (ioqueue_epoll.c:813)
    ==17288==    by 0x656C12: pjsip_endpt_handle_events2 (sip_endpoint.c:740)
    ==17288==    by 0x656CBE: pjsip_endpt_handle_events (sip_endpoint.c:768)
    ==17288==    by 0x4E0F2E: pjsip_thread_func(void*) (stack.cpp:163)
    ==17288==    by 0x6D3B8F: thread_main (os_core_unix.c:523)
    ==17288==    by 0x4C30FA6: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6912181: start_thread (pthread_create.c:312)
    ==17288==    by 0x7DA347C: clone (clone.S:111)
    ==17288==
    ==17288== This conflicts with a previous read of size 4 by thread #18
    ==17288== Locks held: 1, at address 0x16F48688
    ==17288==    at 0x66610C: tcp_send_msg (sip_transport_tcp.c:1243)
    ==17288==    by 0x65F12A: pjsip_transport_send (sip_transport.c:849)
    ==17288==    by 0x65A66A: stateless_send_transport_cb (sip_util.c:1229)
    ==17288==    by 0x65A9D9: stateless_send_resolver_callback (sip_util.c:1334)
    ==17288==    by 0x65AB53: pjsip_endpt_send_request_stateless (sip_util.c:1384)
    ==17288==    by 0x67011B: tsx_send_msg (sip_transaction.c:2109)
    ==17288==    by 0x670976: tsx_on_state_null (sip_transaction.c:2348)
    ==17288==    by 0x66F269: pjsip_tsx_send_msg (sip_transaction.c:1716)
    ==17288==  Address 0x16fab8b0 is 400 bytes inside a block of size 6,144 alloc'd
    ==17288==    at 0x4C2BFA0: malloc (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
    ==17288==    by 0x6D6896: default_block_alloc (pool_policy_malloc.c:46)
    ==17288==    by 0x6DDD2D: pj_pool_create_block (pool.c:60)
    ==17288==    by 0x6DDEDF: pj_pool_allocate_find (pool.c:138)
    ==17288==    by 0x6DDBC7: pj_pool_alloc (pool_i.h:62)
    ==17288==    by 0x6DDC00: pj_pool_calloc (pool_i.h:69)
    ==17288==    by 0x6639D0: pj_pool_zalloc (pool.h:476)
    ==17288==    by 0x664913: tcp_create (sip_transport_tcp.c:610)
    ==17288==    by 0x6659A4: lis_create_transport (sip_transport_tcp.c:1005)
    ==17288==    by 0x6613D3: pjsip_tpmgr_acquire_transport2 (sip_transport.c:1977)
    ==17288==    by 0x6575EB: pjsip_endpt_acquire_transport2 (sip_endpoint.c:1189)
    ==17288==    by 0x65A365: stateless_send_transport_cb (sip_util.c:1158)
    ==17288==  Block was alloc'd by thread #18
