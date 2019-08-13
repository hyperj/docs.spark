# ByKey

## RDD

### JavaPairRDD

#### sampleByKey/sampleByKeyExact

通过 Key 返回对 RDD 分层抽样的子集，math.ceil(numItems * samplingRate)，以上为近似和精确两种类型，其中，精确无放回需要遍历一次数据，精确有放回需要遍历两次数据

- [Spark入门：基本的统计工具（1） – spark.mllib](http://dblab.xmu.edu.cn/blog/1204/)
- [史上最全采样方法详细解读与代码实现](https://blog.csdn.net/bitcarmanlee/article/details/82795137)

#### combineByKey/aggregateByKey/foldByKey/reduceByKey

Map 端：mapSideCombine=true 时生效，按照 key 的不同，将 createCombiner 函数应用于该 key 在该 partition 内的第一条记录，并且对于在该 partition 中相同 key 的记录，应用 mergeValue 处理读入记录，并得到 C 类型的值，最终 map 端的每个分区得到一系列 RDD[(K, C)] 类型的对象（map 端不同partition 输出可能具有相同的 key）
    
Reduce 端：根据 partitioner 进行 shuffle，同一个 key 所有 map 输出结果存放到同一个 partition 内，将相同 key 的数据应用 mergeCombiners，最终得到 RDD[(K, C)] 类型的输出结果

- groupByKey：mapSideCombine = false，C = V，一般场景
- reduceByKey：mapSideCombine = true，C = V，一般场景，相比 groupByKey，在 Map 端聚合优化
- foldByKey：mapSideCombine = true，zeroValue，C = V，需要初始值场景
- aggregateByKey：mapSideCombine = true，zeroValue，需要初始值，输入输出不同类型场景
- combineByKey：按照需要控制场景

```scala
  /**
   * :: Experimental ::
   * Generic function to combine the elements for each key using a custom set of aggregation
   * functions. Turns an RDD[(K, V)] into a result of type RDD[(K, C)], for a "combined type" C
   *
   * Users provide three functions:
   *
   *  - `createCombiner`, which turns a V into a C (e.g., creates a one-element list)
   *  - `mergeValue`, to merge a V into a C (e.g., adds it to the end of a list)
   *  - `mergeCombiners`, to combine two C's into a single one.
   *
   * In addition, users can control the partitioning of the output RDD, and whether to perform
   * map-side aggregation (if a mapper can produce multiple items with the same key).
   *
   * @note V and C can be different -- for example, one might group an RDD of type
   * (Int, Int) into an RDD of type (Int, Seq[Int]).
   */
  @Experimental
  def combineByKeyWithClassTag[C](
      createCombiner: V => C, // 将 V 类型值转换成 C 类型值
      mergeValue: (C, V) => C, // 将一个 V 类型值与一个 C 类型值合并成 C 类型值
      mergeCombiners: (C, C) => C, // 将两个 C 类型值合并为一个 C 类型值
      partitioner: Partitioner, // 分区函数
      mapSideCombine: Boolean = true, // 是否在 Map 端进行 combine 操作
      serializer: Serializer = null // 指定序列化方式
      )(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
    require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
    if (keyClass.isArray) {
      if (mapSideCombine) {
        throw new SparkException("Cannot use map-side combining with array keys.")
      }
      if (partitioner.isInstanceOf[HashPartitioner]) {
        throw new SparkException("HashPartitioner cannot partition array keys.")
      }
    }
    val aggregator = new Aggregator[K, V, C](
      self.context.clean(createCombiner),
      self.context.clean(mergeValue),
      self.context.clean(mergeCombiners))
    if (self.partitioner == Some(partitioner)) {
      self.mapPartitions(iter => {
        val context = TaskContext.get()
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
    }
  }
```

#### sortByKey

```scala
/**
   * Sort the RDD by key, so that each partition contains a sorted range of the elements. Calling
   * `collect` or `save` on the resulting RDD will return or output an ordered list of records
   * (in the `save` case, they will be written to multiple `part-X` files in the filesystem, in
   * order of the keys).
   */
  // TODO: this currently doesn't work on P other than Tuple2!
  def sortByKey(ascending: Boolean = true, // 排序方式
                numPartitions: Int = self.partitions.length // 分区数
                ) : RDD[(K, V)] = self.withScope {
    val part = new RangePartitioner(numPartitions, self, ascending)
    new ShuffledRDD[K, V, V](self, part)
      .setKeyOrdering(if (ascending) ordering else ordering.reverse)
  }
```

#### subtractByKey

`subtractByKey`实现方式为，将`rdd1`数据保存在内存中，通过`rdd2`来过滤是否要删除`rdd1`的值，避免了 OOM 的风险；`subtractByKey`是`cogroup`的优化版本，`cogroup`实现效率较低，将`rdd1`中的匹配和不匹配`rdd2`的值的所有条目都保存在`JHashMap`中，直到匹配结束

## Reference