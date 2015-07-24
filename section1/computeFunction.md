# 计算函数
## 计算函数
`RDD` 抽象类要求所有子类都必须实现 `def compute(split: Partition, context: TaskContext): Iterator[T]` 方法，该方法接受的参数之一是一个 Partition 对象，目的是计算该分区中的数据。我们以 `MappedRDD` 类为例，观察其对于 `compute` 方法的实现。

```scala
  override def compute(split: Partition, context: TaskContext) =
      firstParent[T].iterator(split, context).map(f)
```

`MappedRDD` 类的 `compute` 方法调用当前 RDD 内的第一个父 RDD 的 `iterator` 方法，`iterator` 方法会返回一个包含父 RDD 相同编号分区内数据的 `Iterator[T]` 迭代器对象，其中 `T` 是分区内数据的类型。调用迭代器的 `map` 方法，将每个元素都输入到 `f` 函数中进行转化，返回包含所有转化结果的新迭代器。

对于所有的 RDD，`compute` 函数负责父 RDD 的分区数据到当前 RDD 分区数据的计算逻辑，而父 RDD 分区数据则通过调用其 `iterator` 方法得到。

## iterator 方法

`iterator` 方法的实现在 `RDD` 类中，该方法被用于__拉取一个 RDD 中的分区数据__。

```scala
  /**
   * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
   * This should ''not'' be called by users directly, but is available for implementors of custom
   * subclasses of RDD.
   */
  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```

`iterator` 方法首先检查 RDD 的存储级别 `StorageLevel`，如果存储级别不等于 `None`，说明该 RDD 曾经执行过 `cache` 或者 `persist` 操作，这时候调用 `cacheManager` 对象的 `getOrCompute` 方法拉取数据，获得数据的步骤如下：

- 根据 RDD 的编号和分区编号获取得到存储层相应的块编号，块编号 = "rdd\_" + RDD 编号 + "\_" + 分区编号。
- 通过存储层提供的数据读取接口，提取出块中的数据，数据可能是从本地或者远程节点中提取，存储层提供的数据接口以及数据提取的具体步骤将会在后面章节中细述。
- 如果成功提取得到数据，则直接返回；如果是 RDD 缓存后该分区数据第一次被计算，此时数据还尚未被写入到存储介质当中，提取得到 `None`，这时候需要计算分区数据（调用 `RDD` 的 `computeOrReadCheckpoint` 方法，见下），并将得到的数据写入到存储介质中（本地为何直接返回？）。

如果父 RDD 的存储级别为 `None`，执行 `computeOrReadCheckpoint` 方法，步骤如下：
- 如果 RDD 被标记为检查点，则执行其父 RDD 的 `iterator` 方法。在这里，RDD 的父 RDD 不再是原先的父 RDD，而是一个 `CheckpointRDD` 对象，该对象的 `iterator` 方法最终调用自身的 `compute` 方法，`compute` 方法负责从文件系统中读取并返回数据。我们会在 1.8 小节中解释 RDD 的检查点机制。
- 如果 RDD 未被标记成检查点，则执行自身的 `compute` 函数。

## 参考资料
1. [Apache Spark 的设计与实现 - Cache 和 Checkpoint](http://spark-internals.books.yourtion.com/markdown/6-CacheAndCheckpoint.html)
2. [Spark源码分析之-scheduler模块](http://jerryshao.me/architecture/2013/04/21/Spark%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B-scheduler%E6%A8%A1%E5%9D%97/)
