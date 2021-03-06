#《Hadoop权威指南》note2-Hadoop分布式文件系统
_整理：leotse_

### 分布式文件系统
管理网络上跨平台多台计算机存储的文件系统。

**Note**：非常另类的定义。这是迄今为止本人见过最言简意赅的DFS定义。

### HDFS的设计

1.HDFS以流式数据访问模式来存储超大文件，运行于商用硬件集群上。

**Note**：这里的“超大文件”是指数百MB，数百GB以至于数百TB大小的文件；在选择DFS的时候，我们需要根据我们实际的文件大小来加以判断，比如淘宝开源的TFS适用于海量小文件（<1MB）的场景。HDFS一开始设计的时候就是为了存储大文件，而且是超大文件；  
流式数据是指：在HDFS上进行文件相关分析操作，一般都会读取数据集的大部分或者全部，即读取的文件非常大，因此读第一条记录的延迟就显得不那么重要。

2.HDFS不适用低延迟的数据访问。HDFS是为高吞吐应用设计和优化的，因此会损失一部分的时间性能。

3.大量的小文件。Namenode将HDFS的元数据存储在内存中，所以，HDFS集群的存储能力受限于Namenode的内存大小。

**Note**：根据《Hadoop权威指南》作者的介绍，一般情况下，每个文件、目录和数据块的存储信息大约占150 Byte；如果有100w个文件，且每个文件占一个数据块，则需要300MB的内存。  
HDFS的这种设计（事实上，不少文件系统都是这样设计，由中控节点负责管理所有的元数据）遭到一些人的诟病，因为存在单点故障，于是出现了去中心化的分布式文件系统，也出现了不管理元数据的分布式文件系统，比如[FastDFS](https://github.com/leotse90/blogs/blob/master/FastDFS%E6%A6%82%E8%A7%88.md)。

4.HDFS的文件可能只有一个writer，而且写操作总是将数据追加在文件末尾。HDFS当前不支持多个writer的操作，也不支持在文件的任意位置进行修改。

**Note**：不支持多个writer可以保证文件的一致性，防止同时修改某一文件造成冲突。不支持任意位置修改文件可能是因为HDFS会将文件进行分块存储（默认64MB）有关，数据块的重新分配与写入，文件的一致性等等导致任意修改的代价高昂且效率非常低。  

### HDFS的概念

1.**数据块**  
1）每个磁盘都有默认的数据块大小，这是磁盘进行读写的最小单位。文件系统中也存在块的概念，一般是磁盘数据块大小的整数倍。但是这些都是对用户透明的；

2）HDFS中数据块默认大小为64MB，我们称这些分块为chunk，是HDFS的独立存储单元。HDFS中，一个数据块没有写满是不会写另一个数据块的。

3）为什么HDFS的数据块的是64MB？  
这是由于以下两个原因：a.我们在[《Hadoop权威指南》note1-初识Hadoop](https://github.com/leotse90/SparkNotes/blob/master/%E3%80%8AHadoop%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97%E3%80%8Bnote1-%E5%88%9D%E8%AF%86Hadoop.md)中了解到，寻址时间对于大数据存储分析非常重要，因此，当块设置得足够大，就能降低寻址时间在整个处理过程（寻址+传输）中的占比，因而，传输一个由多个数据块组成的文件的时间取决于磁盘传输速率；换个说法就是，数据块越大，寻址次数越少！b.既然可以降低寻址时间，那么为什么不设置得更大一点呢？这是因为MR中map任务通常一次只处理一个块中的数据，如果数据块size太大，会减少处理map任务的机器数，从而影响计算效率。

**Note**：吐槽一下，为什么不是48MB，也不是72MB？

4）为什么分布式文件系统需要抽象出块的概念？  
a.最明显的好处是，文件大小可以大于一个磁盘的容量。而文件的所有块并不一定要放在同一磁盘上；b.抽象出块的概念，可以简化子系统的设计，所有的文件规范为块，便于管理（由于块的大小是固定的，因此很容易根据磁盘大小计算出该磁盘能容纳的块数），我们在存储文件的时候不仅仅是存储文件的内容本身，还要存储相关的文件元数据，而这些信息不需要和数据块在一起，可以交由系统的某一模块进行统一管理；c.抽象出块可以提高系统的容错能力和可用性，分布式文件系统一般都有数据的副本，丢失一个块，可以去其他机器读取该副本的数据。


**2.Namenode和Datanode**  
1）namenode管理文件系统的命名空间。它维护着整个文件系统树以及整棵树内所有的文件和目录，这些信息以两个文件形式**永远保存**在本地磁盘上：**命名空间镜像文件**和**编辑日志文件**。

2）namenode还记录每个文件的分块数据节点信息，但是，它不永久保存块的位置信息，因为这些信息会在系统启动时由数据节点重建。

**Note**：为什么不永久保存块的位置信息？我们知道，每个datanode知道它的各个数据块的位置信息，因此没有必要增加namenode的管理工作。

3）datanode的主要功能是存储和检索数据块，并定期向namenode发送所在块的存储列表。

4）namenode的容错机制：  
a.备份元数据信息到其他的网络文件系统。一般的做法是，将元数据的持久状态写入本地磁盘的同时，写入一个远程挂载的网络文件系统。  
b.运行一个辅助namenode，也就是我们熟悉的secondarynamenode，但是secondarynamenode本身不能用作namenode，它只是承担了定期通过编辑日志合并命名空间镜像的工作以减少namenode的工作负担，降低其出现问题的概率。secondarynamenode会保存合并后的命名空间镜像的副本，并在namenode发生故障时启用。它的状态总是落后于namenode。

**联邦HDFS**  
我们在前面就已经注意到，Namenode会将文件系统中的每个文件和每个数据块的引用关系保存在**内存**中，因此，Namenode的内存大小就成为系统横向扩展的瓶颈。在Hadoop 2.x版本中，引入了联邦HDFS的概念，通过增加NameNode的数量来实现扩展。每个Namenode管理文件系统命名空间的一部分。

在联邦环境下，每个NameNode负责维护一个命名空间卷（namespace volume），包括命名空间下的源数据和在该命名空间下的文件的所有数据块的数据块池。**命名空间卷之间相互独立，两两之间并不互相通信，甚至其中一个namenode失效也不影响其他namenode维护的命名空间卷**。

集群中的datanode需要注册到每个namenode，并且存储来自多个数据块池中的数据块。也就是，datanode可以存储任何一个namenode中命名空间关联的数据块。

**Note**：如果将HDFS的功能进行高度抽象，我们可以将HDFS划分为3层：低层是**物理存储层**，其上是**数据块管理层**，最高是**命名空间管理层**。在Hadoop 1.x的架构中，物理存储层由众多的datanode构成，上两层则由单一的namenode来承担。而在联邦HDFS中，它管理命名空间的核心思想是：将一个大的命名空间切割成若干子命名空间，每个子命名空间由单独的namenode来管理，namenode之间相互独立，也无需任何协调工作。所有的datanode被多个namenode共享，仍然充当实际数据块的存储场所。而子命名空间和datanode之间则由**数据块管理池**作为中介来建立映射关系。数据块管理层由若干的数据块管理池构成，每个数据块**唯一**属于某个固定的数据池块，而_一个子命名空间可以对应多个数据块池_。以上是Hadoop 2.x中联邦HDFS的实现思想，真正的实现中，子命名空间和其对应的数据块池管理功能都由对应的namenode承担，当然也可以将其独立出来构造一个真正的数据块管理层。  
（注：联邦HDFS只解决了namenode的扩展性问题，并没有解决其高可用性的问题，事实上，namenode联邦使情况变得更糟，因为单个namenode失效的概率更大。）

**HDFS的高可用性**  
Hadoop 2.x针对HDFS的高可用性问题，增加了HA的支持。在这一实现中，配置了一对活动-备用（active-standby）namenode，当活动namenode失效，备用namenode就会接管它的任务并开始服务于客户端的请求，不会有任何明显中断。架构上做了以下调整与修改：  
1）namenode之间需要通过高可用的共享存储实现编辑日志的共享；当备用namenode接管工作，它将通读共享编辑日志直至末尾，以实现与活动namenode的状态同步，并继续读取由活动namenode写入的新条目；  
2）datanode需要同时向两个namenode发送数据块处理报告，因为数据块的映射关系存储在内存中而不是磁盘中；  

一个称为故障转移控制器的系统中有一个新实体管理着将活动namenode转移为备用namenode的转换过程。