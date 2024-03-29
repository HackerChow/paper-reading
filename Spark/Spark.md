# RDDs导读
## Abstract
RDDs对分布式内存进行抽象，使得可以进行内存中的计算。这个特点使得它很适合做迭代式的计算和交互式的数据挖掘。

fault-tolerant，RDDs对共享内存提供了严格的形式。基于粗粒度的转换而不是细粒度的更新。
### 关键词：
1. in-memory computation
2. fault-tolerant
3. coarse-grained transformations

## Introduction
现行的一些分布式计算框架比如，mapreduce，缺乏对分布式内存的抽象。这使得它们很难复用中间结果，比如mapreduce把中间结果存在磁盘上，每次读取都很慢。所以使得他们在做，迭代式的任务，比如机器学习，图算法的时候很难受，像训练网络的时候肯定希望参数一直驻留在内存里。同时，由于磁盘IO什么的很慢，所以做一些交互式的任务的时候延迟会很大。  

当然现在有很多框架，比如Pregel为了进行交互式的图计算，把中间结果存在内存中。还有HaLoop是一个迭代式的mapreduce框架。他们做的确实不错，但是他们都是为了进行某个特定的应用而被构建出来的，缺乏通用性。  

所以本文构建了resilient distributed datasets来解决上述的问题。
1. RDDs是容错的，并行化的数据结构
2. 用户可以明确的将中间结果存在内存中
3. 控制它们的划分从而优化数据放置
4. 使用一系列的“算子”操作它们  

设计RDDs最大的难点在于如何高效地做到容错，现行的集群内存共享策略比如distributed shared memory都是基于细粒度的更新（我觉得可能就是随机写吧，比如我要改一张表里的一个cell，本文没展开，但我觉得蛮有趣的，有空再看），所以它们都是用数据备份和写日志的方式来容错的，但是这样的话在数据密集的任务中会十分耗时。（写日志容错一般都要写metadata和data）

所以为了解决上述的问题，RDDs放弃了细粒度更新，转而变为粗粒度的数据转换。这样的话就不用在日志里记录具体的数据了，而只用记录操作（原文里称为“血缘”）。这样的话，如果我丢了一个RDD，那我还是知道这个RDD是怎么来的，那重新计算得到这个RDD就很容易。比如有一个动物数据集，然后我对这个数据集进行了一个filter的操作，我想得到所有狗的图片作为一个RDD，这样我就把这个操作记录下来。然后电脑挂了，我丢了这个RDD，那我只要再过滤一遍就可以恢复这个RDD了。

本文作者认为，虽然这个粗粒度的数据转换看起来很局限，但实际上它的表达能力是很强的，可以用来实现很多框架比如mapreduce。同时他认为RDDs的强大的适应能力(可以实现很多不同功能的应用)是它最有价值的地方。

## Resilient Distributed Datasets
### RDDs的特征有：
1. 只读
2. 分区
3. RDD只能用特定的operation来创造（比如map，filter，join），数据来源只能是稳定存储介质中的数据（比如磁盘）或者其他RDDs。
4. RDDs不用任何时候被具象化实现（我理解下来就是有真实存在的数据），但是RDDs有足够的信息（log中记录的operation，也就是它的“血缘”）来知道这个RDD是怎么被构造出来的。所以说一个RDD要被程序使用的话，它必须能够被重建。（“血缘”保存完整）。
5. 用户可以控制RDDs的persistence和分区，也就是用户可以指明我要复用哪个RDD，还有这个RDD要怎么存（比如存在内存里)  用户也可以指明, 这个RDD要怎么分区到不同的机器上,通过指定每条记录中的keyword(我对这个的理解就是比如动物数据集,我要不同动物的图片存在不同机器上,所以我的keyword就是species)

### Spark 编程接口
在Spark框架下,每个数据集都是一个对象,然后数据转换是通过调用这些对象的方法来完成的.  

首先用户通过对存在稳定介质中的数据进行转换(比如map或者filter)来定义RDDs.之后用户就可以对这些RDDs使用各种动作(action)了,比如count(返回一个RDD中的数据个数),collect(返回一个RDD中的数据),save(保存数据集)。

文章在这里提到了Spark的一个关键特性"延迟计算",下面解释一下。上面已经说到了对RDD有两类操作，一类是数据转换operation（比如map，filter），第二类是动作action（比如count，collect）。延迟计算的意思是，比如现在有一个动物数据集animals，我写了两行代码

dogs=animals.filter（species=dog）  
maledogs=dogs.filter（gender=male）

写完两行代码之后，我想得到所有公狗的信息，但其实此时由于我只是调用了两个operation，spark还没有真正计算得到这个数据集。它只是记录了要得到这个数据集需要进行哪些操作，在这里就是简单的两次过滤操作，为了得到真正的数据，我需要再加一行代码。  

maledogs.collect（）

这时候我调用了一个action，所以spark会真正的计算出这个数据集。这个懒惰计算的好处就是它可以对连续的几个operation进行流水操作，提高效率。

除此之外，用户还可以调用persist方法（persist属于一个transform operation，也会被懒惰执行）来指明他们想复用哪一个RDD，spark的默认存储是内存，但是用户可以指定（存在磁盘，或者存多个副本在不同机器上）。当然内存不够的时候，spark会进行换出操作，用户可以指定RDD的换出优先级。

### RDD模型的优势
1. RDD只支持粗粒度的转换，所以它只能用作bulk write应用，但它能十分高效地进行容错（通过使用“血缘”）  
2. 此外，RDDs不可改变的特性使得spark可以通过对慢任务进行backup copies来加快执行。但是像DSM这种，就很难做到，因为同一个任务的不同备份会访问同一个全局的位置，那么他们之间就会影响彼此，比如交替update（我猜DSM就是把不同机器的内存拼在一起变成一大块内存，然后每一个内存块都有一个全局的地址）。
3. 最后RDDs比起DSM还有两大优势，首先，对于RDDs的bulk operation，spark runtime程序可以基于数据局部性来调度任务。（DSM做不到，我猜因为在DSM看起来，就只有一整块内存）其次，RDDs在没有足够内存的时候可以优雅地进行降级，只要操作是scan-based operation，就是简单的把存不下的数据放到磁盘上。
4. spark适合做批处理任务，也就是对一堆数据采用同一个操作的任务，不适合做那些对一个共享状态（我觉得就是指共享数据吧）进行精细化异步更新的任务。需要牢记，spark是一个批处理系统！

## spark编程接口

开发者首先写一个驱动程序（master），来连接各台计算机。驱动程序定义一个或多个RDDs，并唤醒在RDDs上的action。master上的spark代码会跟踪各个RDDs的血缘，各个worker是长时间保持的进程，它们可以使得RDD片段驻留在内存中，即使操作变了。

## 实例
1. 逻辑回归
2. pagerank  
这两个任务都是迭代算法，很适合使用spark。原始的输入数据都可以驻留在内存中。值得注意的是，如果迭代的次数很多的话，可以适当的记录中间结果，比如逻辑回归的权重，pagerank的rank值，从而加快故障恢复。

可以从pagerank出发，分析一下通过控制RDDs的切分进行优化。

1. 如果我们将links和初始ranks，对url按照同样的规则进行切分，那么join的时候不同机器就不会有通讯开销了。
2. 我们也可以把相互有link的pages分到一起

join操作之后，我们会得到  
（url，（links，rank））  
然后我们就把url对每个link的贡献值，传到link所在的机器，然后在那个机器上计算link新的贡献值，继续迭代。

## 表达 RDDs
表达每个RDDs，一共需要五部分信息
1. a set of partitions，是它的原子划分
2. 对父RDDs的依赖
3. 一个函数，告诉我们它是如何从父RDDs计算得到的
4. 元数据，记录RDDs是如何划分的
5. 是如何放置的。比如代表HDFS的文件的RDDs就要记录每一块在哪台机器上。  

此外，每个RDDs的map结果和他的划分策略是一样的。

最关键的问题是如何表达RDD之间的依赖，这里把依赖的种类分为两类。
1. narrow依赖，每个父RDD的每个分区，最多被一个子RDD使用，比如map操作。
2. wide依赖，多个子RDD使用，比如join操作。

上面的划分有很大的意义

首先narrow依赖使得同一节点上的流水化执行成为可能，比如一个map接着一个filter。

但是wide依赖就不行，计算子RDD的时候需要所有的父RDD都是可用的。

其次，narrow依赖在错误恢复的时候更加简单。因为只需要利用一个父RDD的分区就可以重新计算了，而且丢失的分区可以并行计算，不会有依赖。但是wide依赖就很烦了，丢失的分区可能要用到父RDD的所有分区，

## 关键实现技术点
### 作业调度
当用户对一个RDD调用了一次action操作之后（意味着不能继续懒惰了），调度器会检查这个RDD的血缘图，对这个图进行stages的划分，每个stage都尽可能包含narrow连接的transform。（就是对operation进行分块）

每个stage的边界，是那些wide依赖的操作或者是任何已经计算好的分块（可以节省父RDD的计算）。划分完stage之后，调度器会发起任务，计算各个阶段中缺失的RDD，直到计算出了目标RDD。

调度器基于数据局部性把任务分配到各台机器上，采用的调度算法是delay scheduling（就是数据在哪里，计算就在哪里）。
1. 如果一个任务需要处理的分区在一个节点的内存中可用，那么调度器会把这个任务分配到这个节点。
2. 如果RDD为它的分区提供了preferred location（比如HDFS），就是去哪些节点访问这个分区会比较快（一般都是基于数据局部性）。

对于wide依赖（父RDD->wide->子RDD），spark把中间记录在父RDD的分区所在的nodes上具象化。（这里的具象化指的是驻留在内存？还是写入磁盘？）

如果一个任务失败了，但是它的stage的父stage仍然可用的话，就在另一个node上重新运行这个任务。如果某些父stage的分区不可用了，那么重新提交计算这个分区的task，递归下去。

### 解释器集成
这里感觉是语言上的工作，先放一放

### 内存管理
spark提供三种存放RDDs的方式
1. 内存中反序列化的java对象
2. 内存中序列化的数据
3. 磁盘中的存储

RDD换入换出采用的算法是LRU，当内存不够，我们就牺牲一个最近最不常使用的RDD的分区，除非这个分区所在的RDD和新的分区是一样的。

在集群上运行的每个spark实例都有自己独立的内存空间。

### 断点保存支持
RDDs的基本恢复机制是血缘，但是如果血缘太长又有很多wide依赖的话恢复还是会很慢的，所以这个时候可以适当的把某些中间的RDD存入磁盘。比如在做pagerank的时候可以适当存一下中间的rank值。

举个例子，做pagerank的时候，我画一个简化图。

rank0 -> wide -> rank1 -> wide -> rank2 -> ...

注意的是做pagerank的时候，我们只会保留最新的rank在内存里，假设是rankn，他有p1，p2，p3，...pm个分区，就算只丢了一个，为了找回这个分区，我们得把整个rankn-1全部算出来，因为这个分区可能会用到全部的rankn-1的分区，以此类推，也就是说虽然大部分的rankn的分区都还在，我们还是需要把之前的计算过程原封不动地再执行一遍。但是checkpoint的话就可以优化一点。

但是对于那些只有narrow依赖的RDDs，比如逻辑回归什么的，丢了一个分区，就只要计算那个分区就可以了，比如如果上面的rank之间都是窄依赖（举个例子），那么如果没有用checkpoint，也只要计算先前的每个rank的一个分区。代价很小。

并且RDDs只读的特性，使得他在做checkpoints的时候比较简单（比DSM），因为不需要担心数据一致性。