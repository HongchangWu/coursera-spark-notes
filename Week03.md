# Week 03

## Shuffling

Some operations, e.g. `groupBy` or `groupByKey`, returns `ShuffledRDD`:

```scala
val pairs = sc.parallelize(List((1, "one"), (2, "two"), (3, "three")))
pairs.groupByKey()
// res2 : org.apache.spark.rdd.RDD[(Int, Iterable[String])]
//   = ShuffledRDD[16] at groupByKey at <console>:37
```

### Grouping vs Reducing

Given

```scala
case class CFFPurchase(customerId: Int, destination: String, price: Double)
```

Assume we have an RDD of the purchases that users of the CFF's (Swiss train
company), we want to calculate how many trips, and how much money was spent by
each individual customer over the course of the month.

- **Grouping**: `groupByKey` moves a lot of data from one node to another to
  ge "grouped" with its key. Doing so is called "shuffling" and is expensive.
  ```scala
  val purchasesRdd: RDD[CFFPurchase] = sc.textFile(...)

  val purchasesPerMonth =
    purchasesRdd.map(p => (p.customerId, p.price)) // Pair RDD
                .groupByKey() // groupByKey returns RDD[(K, Iterable[V])]
                .map(p -> (p._1, (p._2.size, p._2.sum)))
                .collect()
  ```
- **Reducing**: `reduceByKey` reduces dataset locally first, therefore the
  amount of data sent over the network during the shuffle is greatly reduced.
  ```scala
  val purchasesRdd: RDD[CFFPurchase] = sc.textFile(...)

  val purchasesPerMonth =
    purchasesRdd.map(p => (p.customerId, p.price)) // Pair RDD
                .reduceByKey((v1, v2) => (v1._1 + v2._1, v1._2 + v2._2))
                .collect()
  ```

## Partitioning

### Partitions

**Properties of partitions:**
- Partitions never span multiple machines.
- Each machine in the cluster contains one or more partitions.
- The number of partitions to use is configurable. By default, it equals to
  the _total number of cores on all executor nodes._

**Two kinds of partitioning available in Spark:**
- Hash partitioning
- Range partitioning

###  Hash partitioning

Hash partitioning attempts to spread data evenly across partitions _based on
the key._

For example, `groupByKey` first computes per tuple `(k, v)` its partition `p`:

```scala
p = k.hashCode() % numPartitions
```

### Range partitioning

Pair RDDs may contain keys that have an _ordering_ defined (e.g. `Int`,
`Char`, `String`).

For such RDDs, _range partitioning_ may be more efficient. Using a range
partitioner, keys are partitioned according to:
1. an _ordering_ for keys
2. a set of _sorted ranges_ of keys

### Partitioning Data

Two ways to create RDDs with specific partitionings:

1. Call `partitionBy` on an RDD, providing an explicit `Partitioner`.
   Example:
   
   ```scala
   val pairs = purchasesRdd.map(p => (p.customerId, p.price))

   val tunedPartitioner = new RangePartitioner(8, pairs)
   val partitioned = paris.partitionBy(tunedPartitioner).persist()
   ```
   
   Notice the `persist` call. This is to prevent data being shuffled across
   the network over and again.
2. Using transformations that returns RDDs with specific partitions.

   **Partitioner from parent RDD:**
   Pair RDDs that are the rsult of a transformation of a _partitioned_ Pair
   RDD typically is configured to use the has partitioner that was used to
   construct the parent.

   **Automatically-set partitioners:**
   e.g.
   - When using `sortByKey`, a `RangePartitioner` is used.
   - When using `groupByKey`, a `HashPartitioner` is used by default.

Operations on Pair RDDs that hold to (and propagate) a partitioner:

- `cogroup`
- `groupWith`
- `join`
- `leftOuterJoin`
- `rightOuterJoin`
- `groupByKey`
- `reduceByKey`
- `foldByKey`
- `combineByKey`
- `partitionBy`
- `sort`
- `mapValues` (if parent has a partitioner)
- `flatMapValues` (if parent has a partitioner)
- `filter` (if parent has a partitioner)

**All other operations will produce a result without a partitioner.**