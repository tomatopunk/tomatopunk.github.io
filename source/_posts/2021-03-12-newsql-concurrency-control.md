---
title: 一篇文悟透分布式DBMS的并发控制
date: 2021-03-12 09:55:51
tags:
---
NewSQL-并发控制原理解析
<!-- more -->

并发控制方案,是DBMS系统事务处理的设计中, 最突出、最重要的部分, 它几乎影响了系统的所有方面.

并发控制允许客户以多个程序的方式访问数据库. (SIGMOD Record, June 2016 (Vol. 45, No. 2) 49 med fashion), 同时保留了每个人都在专用的系统中单独执行事务的错觉.该方案保证了系统中的原子性与隔离性,因此影响了整个系统的行为.

除此以外, 分布式DBMS设计的另一重点在于, `集中式` 或 `分布式` 的分布式事务协调协议.

在采用了`集中式`协调器的系统中,所有的事务都必须通过协调器,由协调器决定是否允许事务进行. 这与1970-1980年代的TP监控器(如IBM CICS,Oracle Tuxedo)采用的方法相同. 在采用了`分布式`协调器的系统中,所有节点都维护访问其管理的数据的事务状态.节点之间必须相互协调,以确定并发事物是否冲突. 分布式协调器的可拓展性更好,但要求DBMS的时钟必须高度同步,才可以进行全局的事务排序

1970-80年的第一批DBMS使用了两段锁定(2PC)方案, SDD-1是第一个经过专门设计,由集中式协调器管理无共享节点集群上分布式事务处理的DBMS. IBM的`R*`与`SSD-1`类似,但主要区别在于`R*`中事务协调器采用了分布式的2PC协议, 事务直接锁定节点中的数据项. INGRES的分布式版本也使用了去中心化的2PC, 实现了采用集中式算法的死锁检测.


Almost all of the NewSQL systems based on new architectures eschew 2PL because the complexity of dealing with deadlocks. Instead, the current trend is to use variants of timestamp ordering (TO) concurrency control where the DBMS assumes that transactions will not execute interleaved operations that will violate serializable ordering. The most widely used protocol in NewSQL systems is decentralized multi-version concurrency control (MVCC) where the DBMS creates a new version of a tuple in the database when it is updated by a transaction. Maintaining multiple versions potentially allows transactions to still complete even if another transaction updates the same tuples. It also allows for long-running, read-only transactions to not block on writers. This protocol is used in almost all of the NewSQL systems based on new architectures, like MemSQL, HyPer, HANA, and CockroachDB. Although there are engineering optimizations and tweaks that these systems use in their MVCC implementations to improve performance, the basic concepts of the scheme are not new. The first known work describing MVCC is a MIT PhD dissertation from 1979 [3], while the first commercial DBMSs to use it were Digital’s VAX Rdb and InterBase in the early 1980s. We note that the architecture of InterBase was designed by Jim Starkey, who is also the original designer of NuoDB and the failed Falcon MySQL storage engine project.

Other systems use a combination of 2PL and MVCC together. With this approach, transactions still have to acquire locks under the 2PL scheme to modify the database. When a transaction modifies a record, the DBMS creates a new version of that record just as it would with MVCC. This scheme allows read-only queries to avoid having to acquire locks and therefore not block on writing transactions. The most famous implementation of this approach is MySQL’s InnoDB, but it
is also used in both Google’s Spanner, NuoDB, and Clustrix. NuoDB improves on the original MVCC by employing a gossip protocol to broadcast versioning information between nodes.

All of the middleware and DBaaS services inherit the concurrency control scheme of their underlying DBMS architecture; since most of them use MySQL, this makes them 2PL with MVCC systems.

We regard the concurrency control implementation in Spanner (along with its descendants F1 [4] and SpannerSQL) as one of the most novel of the NewSQL systems. The actual scheme itself is based on the 2PL and MVCC combination developed in previous decades. But what makes Spanner different is that it uses hardware devices (e.g., GPS, atomic clocks) for high-precision clock synchronization. The DBMS uses these clocks to assign timestamps to transactions to enforce consistent views of its multi-version database over wide-area networks. CockroachDB also purports to provide the same kind of consistency for transactions across data centers as Spanner but without the use of atomic clocks. They instead rely on a hybrid clock protocol that combines loosely synchronized hardware clocks and logical counters [41].

Spanner is also noteworthy because it heralds Google’s return to using transactions for its most critical services. The authors of Spanner even remark that it is better to have their application programmers deal with performance problems due to overuse of transactions, rather than writing code to deal with the lack of transactions as one does with a NoSQL DBMS [1].

Lastly, the only commercial NewSQL DBMS that is not using some MVCC variant is VoltDB. This system still uses TO concurrency control, but instead of interleaving transactions like in MVCC, it schedules transactions to execute one-at-atime at each partition. It also uses a hybrid architecture where single-partition transactions are scheduled in a decentralized manner but multi-partition transactions are scheduled with a centralized coordinator. VoltDB orders transactions based on logical timestamps and then schedules them for execution at a partition when it is their turn. When a transaction executes at a partition, it has exclusive access to all of the data at that partition and thus the system does not have to set fine-grained locks and latches on its data structures. This allows transactions that only have to access a single partition to execute efficiently because there is no contention from other transactions. The downside of partition-based concurrency control is that it does not work well if transactions span multiple partitions because the network communication delays cause nodes to sit idle while they wait for messages. This partition-based concurrency is not a new idea. An early variant of it was first proposed in a 1992 paper by Hector Garcia-Molina [2] and implemented in the kdb system in late 1990s [5] and in HStore (which is the academic predecessor of VoltDB). 

In general, we find that there is nothing significantly new about the core concurrency control schemes in NewSQL systems other than laudable engineering to make these algorithms work well in the context of modern hardware and distributed operating environments.

## References

> [1] J. C. Corbett, J. Dean, M. Epstein, A. Fikes, C. Frost,J. Furman, S. Ghemawat, A. Gubarev, C. Heiser,P. Hochschild, W. Hsieh, S. Kanthak, E. Kogan, H. Li,A. Lloyd, S. Melnik, D. Mwaura, D. Nagle, S. Quinlan,R. Rao, L. Rolig, Y. Saito, M. Szymaniak, C. Taylor,R. Wang, and D. Woodford. Spanner: Google’s Globally-Distributed Database. In OSDI, 2012
> [2] H. Garcia-Molina and K. Salem. Main memory database systems: An overview. IEEE Trans. on Knowl. and Data Eng., 4(6):509–516, Dec. 1992.
> [3] D. P. Reed. Naming and synchronization in a decentralized computer system. PhD thesis, MIT, 1979
> [4]  J. Shute, R. Vingralek, B. Samwel, B. Handy,C. Whipkey, E. Rollins, M. Oancea, K. Littlefield,D. Menestrina, S. Ellner, J. Cieslewicz, I. Rae,T. Stancescu,and H. Apte. F1: A distributed sql database that scales. Proc. VLDB Endow.,6(11):1068–1079, Aug. 2013.
> [5] A. Whitney, D. Shasha, and S. Apter. High Volume Transaction Processing Without Concurrency Control,Two Phase Commit, SQL or C++. In HPTS, 1997.