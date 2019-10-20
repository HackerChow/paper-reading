# GraphX 导读
## 概要
为了提高图处理的性能，许多系统社区放弃使用通用型的分布式数据流框架，而转向专门的图处理系统，从而获得设计好的编程抽象并且加速迭代式的图处理算法的执行。

作者提出了一个嵌入式的图处理系统GraphX，搭载在spark框架上。

为了和专门处理图的系统达到相同的性能，GraphX做了分布式join的优化和materialized view maintenance（后面一个不知道是个啥）

通过利用分布式流处理系统的优势，Graphx以低代价进行容错。

## 介绍
由于图数据日益增多，推动了很多图处理系统的产生，通过采用  
1. specialized abstrcation
2. graph-specific optimization  
这些系统能高效运行一些迭代式的图算法，比如pagerank和community detection。所以这些针对图处理的系统，在执行图处理的时候，会比一些通用的分布式数据流框架，比如MapReduce，在效率上高出一个数量级。

