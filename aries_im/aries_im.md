ARIES/IM:基于WAL的一种高效高并发的索引管理方法  
C. MOHAN
Data Base Technology Institute, IBM Almaden Research Center, San Jose, CA 95120, USA
wharr@alnwden, tbm. com
FRANK LEVINE
IBM, 11400 Burnet Road, Austin, TX 78758, USA  
摘要：本文提供了一种对于事务系统中索引管理的综合处理方案。这种方案叫做ARIES/IM(Algorlthm for Recovery and isolation Exploiting Semantics for Index Management),可以进行并发控制和B+树的恢复。ARIES/IM确保串行性并使用WAL来恢复。它支持高并发并且有性能优异：（1）将某个key上的锁视为数据页中相应记录的锁（比如在记录级别）（2）为了支持高并发，在提交时不获取索引页锁，即使这时候索引结构发生了变更（SMO）比如页分裂，页删除。（3）支持在遍历，插入，删除同时，进行SMO。在重启恢复时，对于索引变更的redo是以面向页的方式执行（比如，不需要遍历整个索引树），并且在正常执行和重启恢复时，不管有没有可能，undo都是以面向页的方式执行的。ARIES/IM使用多种级别的锁来保证灵活性。ARIES/IM的一个子集已经应用到0S/2 Extended Edition Database Manager。由于ARIES/IM的锁设计很通用，所以它们也被应用到SQL/DS和VM Shared File System，尽管这些系统使用影子页技术来恢复。  
**1、介绍**  
对B树及其变种的并发访问协议已经研究了好长时间(参见 [BaSc77, LeYa81, Mino84, Sagi86, ShGo88]及其它们所引用的一些文章）。这些论文中都没有考虑的如何保证事务的原子性和串行性，该事务包含了对B+树的多种操作（比如获取，插入，删除等）,当事务，系统，或者介质崩溃，并且被不同事务同时访问。[FuKa89]描述了一种错误的（比如：在not found的时候不完全锁，对于范围扫描时进行完全加锁）并且昂贵（使用嵌套事务）的方案来解决此类问题（详情参考[MoLe89]）。数据库管理系统（DBMS）比如[DB21,the 0S12 Extended Edition Database Managerl, System R,NonStop SQLt and SQUDS]的索引管理都支持串行化（重复读（RR）或者第三级别的一致性[Gray78]）。在恢复时，DB2, NonStop SQL 和  0S/2 Extended Edition Database Manager 使用日志先行（WAL）[Gray78,MHLPS92],而System R 和 SQL/DS使用影子页技术[GM BLL81]。不幸的是，上述系统所使用算法细节并没不公开。本文中，我们会描述一种并发控制及恢复策略，称为ARIES/IM（A/gorithm for Recovery and Isolation Exploiting Semantics forindex Management),来构建B+树索引。我们使用ARIES/IM作为0S/2 Extended Edition Database Manager设计的一部分。  
首先，System R如何对索引加锁的大部分细节已经在[Moha90a]中详细描述，并且作为我们ARIES/KVL的一部分，用来改进该方法的并行度和加锁开销特性。除了提供低颗粒度锁（通过数据的记录锁和索引的key值锁），System R系统（起源于IBM的SQL/DS产品）锁提供的并发级别，客户并不满意的。由于，ARIES/KVL是在key值上加锁而不是在单独的key上加锁，所以ARIES/KVL对于增强并发度仍然不够。后者在非唯一性索引上有重大改进。此外，在System R中，即使是对单条记录的插入或删除所要获取的锁数量也是相当多的。因此在设计ARIES/IM是，我们的首要目标是修改System R算法，使其使用WAL，并且彻底提升了它的并发度，性能和功能特性。需要通过高效的恢复和存储以及高并发来支持串行执行。ARIES/IM可以满足所有这些要求。  
ARIES/IM是基于ARIES的恢复和并行控制策略，这在[MH LPS92]中有介绍，并且在众多产品中不同程度的实现了它，比如：IBM的产品0S/2 Extended Edition Database Manager,Workstation Data Save FacilityA/M 和 DB2 V2,IBM的研究原型系统Starburst 和 QuickSilver，以及Transarc的Encinai产品套装，Wisconsin大学的 Gamma data base machine 和 EXODUS 可扩展DBMS。在ARIES/IM中，以极少的锁提供了高度的并发性。通过高效的执行undo和redo以及在undo时避免死锁，来提升重启恢复和正常执行时的性能。我们的并发度测量在[KuPa79]中有明确的定义，它陈述：一系列事务运行相交的事务越多，它们的并行度越高。我们的性能测量就是所需要获取的锁数量，在redo,undo过程中访问的页数量，以及正常操作数，在介质恢复时所需要遍历日志的次数，所需要同步的数据页数，以及日志IO。  
![](./img/fig1.png)  
本文剩余的部分如下组织。本节接下来，我们首先介绍树的结构，并列出在索引并发控制和恢复中的一些问题。