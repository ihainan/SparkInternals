# 分区器

## 分区器类别
分区器我们在前面章节中或多或少有所提及。RDD 中，分区器主要有三个作用，而这三个影响在本质上是相互关联的。1. 决定 RDD 的分区数量。例如执行操作 `groupByKey(new HashPartitioner(2))` 所生成的 `ShuffledRDD` 中，分区的数目等于 2。2. 决定 Shuffle 过程中 Reducer 的个数（实际上是子 RDD 的分区个数）以及 Map 端的一条数据记录应该分配给哪一个 Reducer。3. 决定 `CoGroupedRDD` 与父 RDD 之间的依赖关系。由于分区器能够间接决定 RDD 中分区的数量和分区内部数据记录的个数，因此选择合适的分区器能够有效提高并行计算的性能。Apache Spark 内置了两种分区器，分别是__哈希分区器（Hash Partitioner）__和__范围分区器（Range Partitioner）__。

开发者还可以根据实际需求，编写自己的分区器。分区器对应的源码实现是 `Partitioner` 抽象类，`Partitioner` 的子类（包括自定义分区器）需要实现自己的 `getPartition` 函数，用于确定对于某一特定键值的键值对记录，会被分配到子RDD中的哪一个分区。``` scala
/**
 * An object that defines how the elements in a key-value pair RDD are partitioned by key.
 * Maps each key to a partition ID, from 0 to `numPartitions - 1`.
 */
abstract class Partitioner extends Serializable {
  def numPartitions: Int
  def getPartition(key: Any): Int
}
```## 哈希分区器
哈希分区器的实现在 `HashPartitioner` 中，其 `getPartition` 方法的实现很简单，取键值的hashCode，除以子 RDD 的分区个数，取余即可。 下图展示了 `HashPartitioner` 在 Shuffle 过程中的一个示例，注意此例中，整数的 hashCode 即其本身。

![哈希分区器](../media/images/section1/RDDPartitioner/HashPartitioner.png)


## 范围分区器

哈希分析器的实现简单，运行速度快，但其本身有一明显的缺点：由于不关心键值的分布情况，其散列到不同分区的概率会因数据而异，个别情况下会导致一部分分区分配得到的数据多，一部分则比较少。范围分区器则在一定程度上避免这个问题，范围分区器争取将所有的分区尽可能分配得到相同多的数据。
范围分区器的原理与 Apache Hadoop 上的 TeraSort 算法有些类似，范围分区器希望能够将所有键值划分成几个数据块（数目等于子 RDD 的分区个数），找出每个数据块的边界，每个数据块边界范围内，数据的个数应该是基本相等的，根据边界可以将键值对记录指派给特定的分区。这时候就需要考虑两个问题：一个是如何确定数据块的边界，一个是如何如何快速确定某个键值对记录属于哪个数据块（分区）。

### 确定边界
对于第一个问题，范围分区器给的方法与 TeraSort 算法一致，即采样。范围分区器对父 RDD 中的前 N - 1 个分区，进行采样，其中 N 指子 RDD 的分区个数，对采样的结果进行排序，然后按照权重，将其划分成 N 个数据块，找出每个数据块的上限与下限。

在采样算法上，范围分区器采用的是鱼塘采样法（Reservoir Sampling）。该方法能够从一个元素数量为 n 的集合 S 中，抽取出 k 个样本，其中 n 是一个很大，或者未知的值。受条件限制，不能把数据全部放到内存中，而该算法则能保证每个样本抽取的概率是一致的。

具体关于确定边界算法的细节，我还在研究当中，有个别细节还没完全弄懂，因此在此不做展开，以免误导。感兴趣的可以先参考 `RangePartitioner` 类中 `rangeBounds` 方法、`sketch` 方法以及 `determineBounds` 方法的相关实现。
### 快速定位对于第二个问题，范围分区器给的方法是使用二分查找法。若边界个数少于或等于 128，则分区器直接采用穷举法，从头扫描，找到键值所落在的数据块，从而确定分区编号；若边界个数大于 128，则对边界进行二分查找，判断当前键值是大于等于或者小于中间的边界值，根据情况继续查找左右两侧的边界值，直到分区刚好落在中间位置。具体关于快速定位算法的细节，读者可以参考 `RangePartitioner` 类中 `getPartition` 方法的实现。

## 参考资料
1. [Reservoir sampling - Wikipedia, the free encyclopedia](https://en.wikipedia.org/wiki/Reservoir_sampling)
2. [董的博客 &raquo; Hadoop中TeraSort算法分析](http://dongxicheng.org/mapreduce/hadoop-terasort-analyse/)
