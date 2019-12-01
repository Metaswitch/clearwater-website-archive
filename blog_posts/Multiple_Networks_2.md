Multiple Networks Support (Part 2)
----------------------------------
This is part 2 of a series of blog posts discussing the design process we went through when enabling multiple network support in the Ultima release of Project Clearwater. Part 1 of this series can be found [here](Multiple_Networks_1.md). As always, if you're looking for instructions to enable this feature on your existing Clearwater deployment, please refer to [our documentation](https://clearwater.readthedocs.org/en/latest/Multiple_Network_Support/) In the last post, we discussed the problem space and listed some possible approaches to implementing a solution. In this post, we'll look at some of these tools and discuss the issues we had with them.

## Linux Routing Tables

The obvious first candidate was Linux's outbound routing tables. These routing tables allow a system administrator to configure rules that select a network interface to use to reach a given target address (and optionally configure a gateway to contact to reach unrecognised addresses). A simple example routing table might look like:

    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         10.0.0.1        0.0.0.0         UG    100    0        0 eth0
    192.168.0.0     0.0.0.0         255.255.255.0   U     0      0        0 eth1
    10.0.0.0        0.0.0.0         255.0.0.0       U     0      0        0 eth0

This table indicates that packets destined for the `10.0.0.0/8` subnet should be sent over `eth0`, those for the `192.168.1.0/24` subnet should be sent over `eth1` and everything else should be sent to `10.0.0.1` over `eth0` (this is the default gateway). Unfortunately, this routing can only look at the destination address to determine where to send a packet and, since we don't want to require that two networks to have non-overlapping address spaces, this is insufficient to guarantee that packets are routed correctly. As an example of this limitation, if the two networks are both `10.0.0.0/8` then:

*   If a process sent a packet to `10.0.0.3` we'd not know which network (and hence interface) to use.
*   If the process sent a packet to `192.168.0.3`, we'd want to use the signalling network's default gateway for signalling traffic and the management network's one for management traffic, but again we can't tell which is in use.

Given these limitations, we couldn't use Linux routing tables on their own.

## iptables

Having determined that routing tables on their own were not sufficient for our needs, we looked at `iptables`. We hoped that we could use some carefully crafted rules to separate the traffic onto the two networks and to police that traffic leaves and arrives on the correct interface. Sadly, although `iptables` do a good job of policing incoming packets, they are not as good at controlling outgoing ones. The `iptables` rules are run after the previously described Linux routing tables and cannot change which interface is chosen for a given packet so we're left with the same limitations that Linux routing tables have already. There is a mechanism that allows `iptables` to force a packet to be passed back through the routing tables after modification and we could possibly have leveraged this by:

1.  modifying (say) the management traffic packets to reference a fake network
2.  force the modified packets to be re-routed
3.  register routing rules for that fake network sending the packets to the management interface
4.  fix up the modifications on the packets before sending them out

This plan would have required that the `iptables` rules be able to tell whether a given packet was management or signaling. The only reliable way to do this would be to inspect the source IP address and port (and know that, for example, port 22, used for SSH, belongs to the management network). Some of third party libraries we were using would not bind locally-established connections to a local address/port so we could not see a way to make this proposal work. `iptables` are powerful packet filters/editors and can achieve almost any low-level network control you desire, but even they cannot meet our requirements.

## Next Time

[Next time](Multiple_Networks_3.md) we'll look at the solution we finally settled on, and the follow on problems we met with before finalising our design.
