# 计算函数
## 分区数据计算过程
在依赖小节中，我提及了 RDD 通过计算链来实现容错机制和并行计算，本节进一步研究整个计算过程，建议配合依赖章节最后两小节来看，会有更深入的理解。

RDD 内部，数据的计算是惰性的，一系列转换操作只有在遇到动作操作时候，才会真的去计算 RDD 分区内的数据。在调度章节中，我们会发现，实际计算过程大概是这样的：

1. 根据动作操作来将一个应用程序划分成多个作业。
2. 一个作业经历 DAG 调度和任务调度之后，被划分成一个一个的任务，对应 `Task` 类。
3. 任务被分配到不同核心去执行，执行 `Task.run`。
4. `Task.run` 会调用阶段末 RDD 的 `iterator` 方法，获取该 RDD 某个分区内的数据记录，而 `iterator` 方法有可能会调用 RDD 类的 `compute` 方法来负责父 RDD 与子 RDD 之间的计算逻辑。

整个过程会在调度章节中细述，在此不进行展开，我们只需要知道 Apache Spark 最终会调用 RDD 的 `iterator` 和 `compute` 方法来计算分区数据即可。下面我会分别介绍这两个方法的具体实现。

## compute 方法
`RDD` 抽象类要求其所有子类都必须实现 `compute` 方法，该方法接受的参数之一是一个`Partition` 对象，目的是计算该分区中的数据。

以 `map` 操作生成得到的 `MapPartitionsRDD` 类为例，观察其内部 `compute` 方法的实现。

```scala
override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))
```

其中，`firstParent` 在 `RDD` 抽象类中定义。

```scala
  /** Returns the first parent RDD */
  protected[spark] def firstParent[U: ClassTag] = {
    dependencies.head.rdd.asInstanceOf[RDD[U]]
  }
```

`MapPartitionsRDD ` 类的 `compute` 方法调用当前 RDD 内的第一个父 RDD 的 `iterator` 方法，该方的目的是拉取父 `RDD` 对应分区内的数据。`iterator` 方法会返回一个迭代器对象，迭代器内部存储的每个元素即父 RDD 对应分区内__已经计算完毕的数据记录__。得到的迭代器作为 `f` 方法的一个参数。`f` 在 `RDD` 类的 `map` 方法中指定，如下所示。

```
  /**
   * Return a new RDD by applying a function to all elements of this RDD.
   */
  def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
```

`compute` 方法会将迭代器中的记录一一输入 `f` 方法，得到的新迭代器即为所求分区中的数据。
其他 `RDD` 子类的 `compute` 方法与之类似，在需要用到父 RDD 的分区数据时候，就会调用 `iterator` 方法，然后根据需求在得到的数据之上执行粗粒度的操作。__换句话说，`compute` 函数负责的是父 `RDD` 分区数据到子 `RDD` 分区数据的变换逻辑。__
## iterator方法
`iterator` 方法的实现在 `RDD` 抽象类中。

```scala
  /**   * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.   * This should ''not'' be called by users directly, but is available for implementors of custom   * subclasses of RDD.   */  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {    if (storageLevel != StorageLevel.NONE) {      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)    } else {      computeOrReadCheckpoint(split, context)    }  }
```

`iterator` 方法首先检查当前 `RDD` 的存储级别，如果存储级别不为 `None`，说明分区的数据要么已经存储在文件系统当中，要么当前 RDD 曾经执行过 `cache`、 `persise` 等持久化操作，因此需要想办法把数据从存储介质中提取出来。

`Iterator` 方法继续调用 `CacheManager` 的 `getOrCompute` 方法。```scala
 /** Gets or computes an RDD partition. Used by RDD.iterator() when an RDD is cached. */  def getOrCompute[T](      rdd: RDD[T],      partition: Partition,      context: TaskContext,      storageLevel: StorageLevel): Iterator[T] = {    val key = RDDBlockId(rdd.id, partition.index)    blockManager.get(key) match {       case Some(blockResult) =>         // Partition is already materialized, so just return its values         context.taskMetrics.inputMetrics = Some(blockResult.inputMetrics)         new InterruptibleIterator(context, blockResult.data.asInstanceOf[Iterator[T]])	       case None =>         // 省略部分源码         val computedValues = rdd.computeOrReadCheckpoint(partition, context)         val cachedValues = putInBlockManager(key, computedValues, storageLevel, updatedBlocks)         new InterruptibleIterator(context, cachedValues)    }    // 省略部分源码 }
```

`getOrCompute` 方法会根据 RDD 编号与分区编号计算得到当前分区在存储层对应的块编号，通过存储层提供的数据读取接口提取出块的数据。这时候会有两种可能情况发生：- 数据之前已经存储在存储介质当中，可能是数据本身就在存储介质（如读取 HDFS 中的文件创建得到的 RDD）当中，也可能是 RDD 经过持久化操作并经历了一次计算过程。这时候就能成功提取得到数据并将其返回。- 数据不在存储介质当中，可能是数据已经丢失，或者 RDD 经过持久化操作，但是是当前分区数据是第一次被计算，因此会出现拉取得到数据为 `None` 的情况。这就意味着我们需要计算分区数据，继续调用 `RDD` 类 `computeOrReadCheckpoint` 方法来计算数据，并将计算得到的数据缓存到存储介质中，下次就无需再重复计算。

如果当前RDD的存储级别为 `None`，说明为未经持久化的 `RDD`，需要重新计算 RDD 内的数据，这时候调用 `RDD` 类的 `computeOrReadCheckpoint` 方法，该方法也在持久化 RDD 的分区获取数据失败时被调用。``` scala
  /**   * Compute an RDD partition or read it from a checkpoint if the RDD is checkpointing.   */  private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] = {    if (isCheckpointed) firstParent[T].iterator(split, context) else compute(split, context)  }
```

`computeOrReadCheckpoint` 方法会检查当前 RDD 是否已经被标记成检查点，如果未被标记成检查点，则执行自身的 `compute` 方法来计算分区数据，否则就直接拉取父 RDD 分区内的数据。

需要注意的是，对于标记成检查点的情况，当前 RDD 的父 RDD 不再是原先转换操作中提供数据的父 RDD，而是被 Apache Spark 替换成一个 `CheckpointRDD` 对象，该对象中的数据存放在文件系统中，因此最终该对象会从文件系统中读取数据并返回给 `computeOrReadCheckpoint` 方法，在 1.8 节我会解释这样做的原因。## 参考资料
1. [Cache 和 Checkpoint 功能 |  Apache Spark 的设计与实现](http://spark-internals.books.yourtion.com/markdown/6-CacheAndCheckpoint.html)
