---
title: spark_dagscheduler
date: 2017-10-16 20:01:20
tags:
  - spark
  - stage
  - rdd
  - partition
  - dagscheduler
  - source_code
toc: true
---

> 基于 spark 1.6.2

面向 Stage 的调度器，负责计算每个 job 的 DAG，并将 DAG 图划分为不同的 stage，对哪些 RDD 以及相应输出进行记录，寻找一个运行相应 job 所需要的最小 stage。然后将 `stage` 以 `TaskSet` 的形式提交到下层的 `TaskScheduler` 进行具体的 task 调度。每个 `TaskSet` 包含整个可以独立运行的 task，这些 task 能够利用集群上已有的数据立即运行起来，如果集群上已有的数据已经不存在了，那么当前 task 就会失败。

Spark 的 stage 以 RDD 的 shuffle 为界进行划分。窄依赖的 RDD 操作会被穿起来放到一个 task 中，比如 `map()`, `filter()` 这样的操作。但是需要使用到 shuffle 依赖的操作，需要多个 stage（至少一个将中间文件写到特定的地方，另外一个从特定的地方进行读取）。每个 Stage，只会对其他 Stage 有 shuffle 依赖，在同一个 stage 中会进行很多计算。实际的将计算串起来的操作在 RDD.compute 中完成。

`DAGScheduler` 同样会基于缓存状态决定 task 希望运行在那（preferred location），如果 shuffle 输出文件丢失造成的 Stage 失败，会重新被提交。在 **Stage 内部** 的不是由 shuffle 文件丢失造成的失败，由 `TaskScheduler` 来完成，`TaskScheduler` 会在取消整个 stage 前进行小部分重试。
<!-- more -->
下面的几个核心概念：

- Jobs (通过 `ActiveJob` 来表示）是提交给调度器中最上次工作单元。比如，用户触发一次 action，比如 `count()`，的时候，就会提交一个 Job。每个 Job 可能会包含多个 stage
- Stages 是 Job 中产生中间结果的一系列 Task 的集合，同一个 Stage 的每个 Task 运行着同样的逻辑，只是处理同一个 RDD 的不同分区。Stage 以 Shuffle 为界（后面的 Stage 必须等前面的 Stage 运行完才能继续往下进行）。现在有两种 Stage：`ResultStage`，Job 的最终执行 action 的 Stage，`ShuffleMapState` 产生中间文件的 shuffle。如果多 Job 公用同一个 RDD 的话，Stage 可能会在多个 Job 间共享。
- Tasks 是组小的独立工作单元，每个 Task 会被独立的分发到具体的机器上运行
- Cache tracking: `DAGScheduler` 会记录哪些 RDD 以及被缓存过，从而避免重复计算，也会记录哪些 Stage 已经生成过输出文件从而避免重复操作
- Preferred locations: `DAGScheduler` 会根据计算所依赖的 RDD 数据、缓存的地址以及 shuffle 输出结果对 Task 进行调度
- Cleanup: 如果上游依赖的 Job 处理完成后，下游的数据会被清理掉，防止长驻服务的内存泄漏

为了能够从错误中进行恢复，同一个 Stage 可能会被运行多次，每一次就是一个 "attempts"。如果 `TaskScheduler` 汇报某个 task 失败的原因是因为依赖的前一个 Stage 的输出文件已经不见了，那么 `DAGScheduler` 会对前一个 Stage 重新进行提交。这通过 `CompletionEveent` 以及 `FetchFailed` 或者 `ExecutorLost` 事件来完成。`DAGScheduler` 会等待一小段时间来判断是否还有其他 task 需要重试，然后将所有失败的 task 进行重试。在这一过程中，我们需要重新对之前清理过的 Stage 进行计算。由于之前的 “attempt" 可能还在运行，所以需要特别注意

![processure](/images/procesure.jpg)

上面的图是 DAGScheduler.scala 的主题脉络，当然还包括其他诸如 `taskStarted`, `taskGettingResult`, `taskEnded`, `executorZHeatbeatReceived`, `executorLost`, `executorAdded`, `taskSetFailed`, 等函数

> 其中带箭头的虚线为消息调用；带箭头的实线为直接调用，无箭头的实线表示方法申明和内部的主要实现（比如 `getShuffleMapStage` 中包括了 `getAncestorShuffleDependencies` 和 `newOrUsedShuffleStage`)

#### 上面是注释，下面是代码解释

入口在 `runJob`，`runJob` 会调用 `submitJob(rdd, func, partitions, callSite, resultHandler, properties)` 返回一个 waiter，等待处理完成

`submitJob` 核心代码如下，主要生成一个 waiter 对象，然后发送一个 `JobSubmitted` 信号


```
val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
eventProcessLoop.post(JobSubmitted(jobId, rdd, func2, partitions.toArray, callSite, waiter, SerializationUtils.clone(properties)))
```

`JobSubmitted` 方法会将整个 DAG 图进行 Stage 的划分，然后提交 finalStage（也就是 action 所在的 Stage），其中 `newResultStage` 会进行具体的 Stage 划分，

```
try {
	// New stage creation may throw an exception if, for example, jobs are run on a
	// HadoopRDD whose underlying HDFS files have been deleted.
	finalStage = newResultStage(finalRDD, func, partitions, jobId, callSite)
} catch {
	case e: Exception =>
		logWarning("Creating new stage failed due to exception - job: " + jobId, e)
		listener.jobFailed(e)
		return
}
```

`submitJob` 返回的 `JobWaiter`，JobWaiter 用于控制 Job，以及 Job 结束后进行相应的状态更新


`newResultStage` 会将所有的 Stage 划分出来（通过 `getParentStagesAndId` 函数，`getParentStages` 进行具体的 Stage 划分），其中 `getParentStages` 进行 BFS 进行查找（这个地方 BFS 和 DFS 有什么区别？）


```
private def getParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
	val parents = new HashSet[Stage]
	val visited = new HashSet[RDD[_]]
	// We are manually maintaining a stack here to prevent StackOverflowError
	// caused by recursively visiting
	val waitingForVisit = new Stack[RDD[_]]
	def visit(r: RDD[_]) {
		if (!visited(r)) {
			visited += r
			// Kind of ugly: need to register RDDs with the cache here since
			// we can't do it in its constructor because # of partitions is unknown
			for (dep <- r.dependencies) {
				dep match {
					case shufDep: ShuffleDependency[_, _, _] =>
						parents += getShuffleMapStage(shufDep, firstJobId)
					case _ =>
						waitingForVisit.push(dep.rdd)
				}
			}
		}
	}
	waitingForVisit.push(rdd)
	while (waitingForVisit.nonEmpty) {
		visit(waitingForVisit.pop())
	}
	parents.toList
}
```

getParentStages 返回某个 RDD 的所有依赖的 stage（直接和间接的），stage 以 getShuffleMapStage() 返回为准


`getShuffleMapStage` 首先从 shuffleToMapStage(shuffleid 到 stage 的 map 结构）中查找，没有找到就以 shuffleDep.rdd 为起始点建立一个依赖关系，并且将整条依赖链上的东西都建立起来


`getAncestorShuffleDependencies` 会以一个 RDD 为起点，找到 RDD 直接&间接 依赖的所有 shuffleDependency

`getAncestorShuffleDependencies` 和  `getParentStages` 类似但又不一样，前者在搜索的时候，每次都需要把 rdd.dep 入队，而后者只需要将 narrowdependency 的入队。还有为什么两个函数一个结果保存为 Set，一个是 Stack？

`newShuffleMapStage` 中会更新 job 和 stage，以及 stage 和 job 的关系，每个 stage 属于哪些 job，每个 job 包含哪些 stage


整个类中有一个变量 `mapOutputTracker :MapOutputTracker` 用于记录 shuffle 的结果以及位置


`MapOutputTracker` 记录 shuffleMapStage 的 output location  ，为啥要两个 status map（一个 MapStatus，一个 CachedSerializedStatus，后者是前者的序列化之后的结果）,这些 Map 都是带 ttl 的


```
/* org.apache.spark.MapOutputTracker.scala
* Class that keeps track of the location of the map output of
* a stage. This is abstract because different versions of MapOutputTracker
* (driver and executor) use different HashMap to store its metadata.
```

Driver 使用 MapOutputTrackerMaster 跟踪 output location，只有所有 partition 的 output location 都就绪了，整个被依赖的 RDD 才是就绪的
			    

```
* MapOutputTracker for the driver. This uses TimeStampedHashMap to keep track of map
* output information, which allows old output information based on a TTL.
```

Executor 则使用 `MapOutputTrackerWorker` 从 Driver 获取 map output 的相应信息


`newOrUsedShuffleStage` 函数中首先查找该 shuffleMapStage 是否注册过 MapOutputTracker，如果注册过就直接获取，如果没有注册过就进行注册。
MapOutputTracker 有两个 Map 结构，一个是原始的 partition location，一个是序列化之后的（用于加速？），这两个 map 包含过期清理策略，用于节省空间


定期清理的 Meta 信息包括如下几种：

- MAP_OUTPUT_TRACKER, 
- SPARK_CONTEXT, 
- HTTP_BROADCAST, 
- BLOCK_MANAGER,
- SHUFFLE_BLOCK_MANAGER, 
- BROADCAST_VARS

### 上面是一条链路上的相关函数，下面包括一些其他的处理

#### `handleTaskCompletion` 处理 Task comple 的信息（不分成功和失败）
task complete 会分几种信息：

- Success：
  首先将 task 从 pendingTask 中去掉
    task 分为两种：
	- ResultTask：
	将 job 对应的当前 task 标记为 true（如果没有标记过的话），如果整个 job 都处理完成，就将 stage 标记为完成
	- ShuffleMapTask：
	更新 outputLocation （当前 shuffleMapTask 的输出）
	如果 ShuffleMapTask 的所有 partition 都处理完成，就将当前的 stage 标记为完成
	这里为了防止是重复进行计算（之前失败过），需要重新进行 outputLocation 的注册（主要是增加 epoch）
	如果当前当前的 Stage.isAvailable 为 true 就通知所有依赖该 stage 的 stage 可以继续工作了。否则重新提交失败的 task（注意这里available 和上面的 finish 不一样，available 是以 output 是否能够获取到为准）

	- Resubmitted：
	将 task 加到 pendingTask 中即可，等待后续的调度

	- FetchFailed：
	首先判断失败 task 的 attemp 是否是当前的 attemp，不是就忽略，然后判断当前 stage 是否正在运行，如果不是，忽略。
	否则将当前的 stage 标记为 finish，然后将进行 mapStage.removeOutputLoc 以及 mapOutputTracker.unregisterMapOutput。
	如果同一个 executor 上的 fetchFailed 很多（这个有调用方判断），则将所在的 executor 标记为 Failed

	- 其他信息，直接忽略


#### ExecutorLost
如果上报的 executor 之前没有上报过（会有一个 Map 记录所有上报过的 executor），或者之前上报的该 executor 对应的 epoch 小于 currentepoch， 则需要进行处理

首先从 blockManagerMaster 中将当前 executor 删除

如果没有启动 externalShuffleService（开启 `DynamicAllocation` 后需要开启，不然会出问题），或者 fetchFailed（由调用方设置），则进行下面的操作：

`if (!env.blockManager.externalShuffleServiceEnabled || fetchFailed)`

所该 executor 上所有的输出都进行标记删除，并且增加 `mapOutputTracker` 的 epoch

#### ExecutorAdded
  如果当前添加的 executor 是马上需要回收的，那么就从即将回收的 map 中删除，防止回收，否则不需要操作

#### StageCancellation
  如果有正在运行的 job 依赖当前 stage，则将所有的 job 标记为 cancel

#### JobCancellation
  将该 job 以及只由该 job 依赖的 stage 都标记为 failed

## 配置
  `spark.stage.maxConsecutiveAttempts` 表示一个 stage 尝试多少次之后会被标记为失败

  SparkContext 中会根据模式生成和注册相应的 backend 以及 taskscheduler


### 问题
1. 如果一个 Stage 有多个 RDD，那么这些 RDD 是在同一个 TaskSet 中吗
2. 如何模拟 `r2 = r0.reduceByKey; r3 = r1.reduceByKey; r4 = r2.map(xx); r5 = r4.union(r3); r6 = r5.map; r7 = r6.reduceByKey` 的 Stage 划分和生成（会有多少个 Stage，每个 Stage 分别包含哪些 RDD，以及整个 DAG 怎么整合起来的）
![](/images/spark_stage.jpg)
