Scylla and Seastar
------------------
Clearwater is [architected](Architecture.md) as stateless processing nodes backed by several different clustered stores to store its state.

*   [Memcached](http://memcached.org/) for registration, subscription and dialog state.
*   [Chronos](https://github.com/Metaswitch/chronos) for timers.
*   [Cassandra](http://cassandra.apache.org/) for caching subscriber configuration learnt from the HSS.

Recently, we came across [Scylla](http://www.scylladb.com/), a reimplementation of Cassandra in C++ claiming up to 10x higher performance. Although Cassandra isn't a bottleneck for Clearwater, it can take around a third of the CPU, so a faster implementation would give us a measurable performance boost. Also, since Cassandra is written in Java, we have seen [occasional garbage collection pauses on single-vCPU VMs](https://issues.apache.org/jira/browse/CASSANDRA-9805) - since Scylla is implemented in C++, we might hope it wouldn't suffer from the same issue. Scylla is built on [Seastar](http://www.seastar-project.org/), a C++ framework for high-performance event-driven applications, such as data stores, web servers and (maybe) SIP servers. As well as providing a clean abstraction for asynchronous function, it also

*   provides APIs to schedule code on specific CPU cores - this improves performance because the memory that the code accesses can be kept in cache on that CPU, resulting in faster access times
*   uses [DPDK](http://dpdk.org/) for fast network access - DPDK works by using polling rather than interrupts to receive network packets - this reduces the overhead of handling network traffic, giving higher maximum performance, but also means that your process uses 100% CPU even when idle
*   runs both on Linux and on [OSv](http://osv.io/) - a bare-bones OS designed for applications running under virtualization.

After a bit of investigation, we found that Scylla wasn't quite ready for use in Clearwater yet - one of the key points being that it [doesn't yet support updating schema](http://www.scylladb.com/technology/status/#road-map), which can be required on upgrade. However, it remains a very exciting project, and we're hoping to take another look at it when it's matured further. Seastar might also form the basis for a high-performance SIP server - although we're pretty happy with Clearwater's performance, it might be interesting to experiment with porting part of it to Seastar and seeing how it behaves.
