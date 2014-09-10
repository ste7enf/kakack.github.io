---

layout: post
categories: [Spark]
tags: [Spark,Big Data,Distributed System,Graphx]

---

#[GraphX Programming Guide](http://ampcamp.berkeley.edu/4/exercises/graph-analytics-with-graphx.html)

![Alt text](http://spark.apache.org/docs/latest/img/graphx_logo.png)

##Overview

Graphx是一个新的SparkAPI，用来做graphs和graph-parallel计算，从更高层面说，Graphx扩展了RDD，引入了[Resilient Distributed Property Graph](http://spark.apache.org/docs/latest/graphx-programming-guide.html#property_graph)，一个有向多重图，每个边和向量上都有相应地属性。其中包含多种图计算和优化算法。

####Background on Graph-Parallel Computation

新的图计算需求促进了新的graph-parallel系统的发展。

![Alt text](http://spark.apache.org/docs/latest/img/data_parallel_vs_graph_parallel.png)

![Alt text](http://spark.apache.org/docs/latest/img/tables_and_graphs.png)

![Alt text](http://spark.apache.org/docs/latest/img/graph_analytics_pipeline.png)

Graphx的目的是通过一个集合的API在一个系统上统一graph-parallel和data-parallel计算。Graphx的API可以让用户既可以用Graph的角度，也可以用collection的角度来查看数据，不需要额外的数据转移和备份。

##Getting Started

导入必要的库包


    import org.apache.spark._
    
    import org.apache.spark.graphx._
    
    // To make some of the examples work we will also need RDD
    
    import org.apache.spark.rdd.RDD

##The Property Graph

Property Graph是一种有向的多重图，有用户自定义的对象，并通过边和向量来连接。一个有向多重图首先是一个有向图，其次其潜在很多平行的边共享同样的起始点和终结点。每个向量由独立唯一的64位长id标注（VertexID）

![Alt text](http://ampcamp.berkeley.edu/4/exercises/img/social_graph.png)

我们首先储存这张property graph

```
val vertexArray = Array(
  (1L, ("Alice", 28)),
  (2L, ("Bob", 27)),
  (3L, ("Charlie", 65)),
  (4L, ("David", 42)),
  (5L, ("Ed", 55)),
  (6L, ("Fran", 50))
  )
val edgeArray = Array(
  Edge(2L, 1L, 7),
  Edge(2L, 4L, 2),
  Edge(3L, 2L, 4),
  Edge(3L, 6L, 3),
  Edge(4L, 1L, 1),
  Edge(5L, 2L, 2),
  Edge(5L, 3L, 8),
  Edge(5L, 6L, 3)
  )

```

在这用到了edge这个类，一个edge通过*scrId*和*dstId*来表达对应的起始点和终结点。另外还有一个*attr*成员来描述edge的属性。

然后使用`sc.parallelize`来构建*vertexArrayh*和*edgeArray*变量的RDD

```
val vertexRDD: RDD[(Long, (String, Int))] = sc.parallelize(vertexArray)

val edgeRDD: RDD[Edge[Int]] = sc.parallelize(edgeArray)
```

然后我们根据已有的顶点的RDD (with type RDD[(VertexId, V)]) 和边的RDD (with type RDD[Edge[E]]) 来构建我们需要的graph

```
val graph: Graph[(String, Int), Int] = Graph(vertexRDD, edgeRDD)
```

在这个property graph中的顶点是一个元组（String，Int）表示用户的名字和年纪，而边上的数字表示权重或者社交网络中的关联程度。

##Graph View

在很多情况下，我们希望能提取图中顶点和边的RDD视图，因此graph类所包含的成员都提供了能访问graph中顶点和边的途径。然而这些扩展RDD的成员[(VertexId, V)] and RDD[Edge[E]]，他们事实上是依靠GraphX所表示的图数据内部的影响关系所表现出来的。
使用`graph.vertices`来显示所有年龄大于30岁的人。

```
// Solution 1
graph.vertices.filter { case (id, (name, age)) => age > 30 }.collect.foreach {
  case (id, (name, age)) => println(s"$name is $age")
}

// Solution 2
graph.vertices.filter(v => v._2._2 > 30).collect.foreach(v => println(s"${v._2._1} is ${v._2._2}"))

// Solution 3
for ((id,(name,age)) <- graph.vertices.filter { case (id,(name,age)) => age > 30 }.collect) {
  println(s"$name is $age")
}

```

结果显示：

```
David is 42
Fran is 50
Ed is 55
Charlie is 65

```

除了属性图的顶点和边视图，Graphx还有一个三重态视图（triplet view），可以如图表示：

![Alt text](http://ampcamp.berkeley.edu/4/exercises/img/triplet.png)

类`EdgeTriplet`继承了`Edge`类，加上了`srcAttr`和`dstAttr`成员，表示起止点的性质,可以用`graph.triplets`来表示相互之间的一级关联关系。

```
for (triplet <- graph.triplets.collect) {
  println(s"${triplet.srcAttr._1} likes ${triplet.dstAttr._1}")
}

```

结果如：

```
Bob likes Alice
Bob likes David
Charlie likes Bob
Charlie likes Fran
David likes Alice
Ed likes Bob
Ed likes Charlie
Ed likes Fran
```

同样可以找出关联程度在5以上的组对，

```
for (triplet <- graph.triplets.filter(t => t.attr > 5).collect) {
  println(s"${triplet.srcAttr._1} loves ${triplet.dstAttr._1}")
}

```

##Graph Operation

作为RDD，其本身就自带有许多基础的操作，比如`count`, `map`, `filter`和`reduceByKe`等，对于属性图来说，同样也具有这一系列的基础操作集合，以下就罗列的一些通过Graphx API暴露出来的function：

```
/** Summary of the functionality in the property graph */
class Graph[VD, ED] {
  // Information about the Graph
  val numEdges: Long
  val numVertices: Long
  val inDegrees: VertexRDD[Int]
  val outDegrees: VertexRDD[Int]
  val degrees: VertexRDD[Int]

  // Views of the graph as collections
  val vertices: VertexRDD[VD]
  val edges: EdgeRDD[ED]
  val triplets: RDD[EdgeTriplet[VD, ED]]

  // Change the partitioning heuristic
  def partitionBy(partitionStrategy: PartitionStrategy): Graph[VD, ED]

  // Transform vertex and edge attributes
  def mapVertices[VD2](map: (VertexID, VD) => VD2): Graph[VD2, ED]
  def mapEdges[ED2](map: Edge[ED] => ED2): Graph[VD, ED2]
  def mapEdges[ED2](map: (PartitionID, Iterator[Edge[ED]]) => Iterator[ED2]): Graph[VD, ED2]
  def mapTriplets[ED2](map: EdgeTriplet[VD, ED] => ED2): Graph[VD, ED2]

  // Modify the graph structure
  def reverse: Graph[VD, ED]
  def subgraph(
      epred: EdgeTriplet[VD,ED] => Boolean = (x => true),
      vpred: (VertexID, VD) => Boolean = ((v, d) => true))
    : Graph[VD, ED]
  def groupEdges(merge: (ED, ED) => ED): Graph[VD, ED]

  // Join RDDs with the graph
  def joinVertices[U](table: RDD[(VertexID, U)])(mapFunc: (VertexID, VD, U) => VD): Graph[VD, ED]
  def outerJoinVertices[U, VD2](other: RDD[(VertexID, U)])
      (mapFunc: (VertexID, VD, Option[U]) => VD2)
    : Graph[VD2, ED]

  // Aggregate information about adjacent triplets
  def collectNeighbors(edgeDirection: EdgeDirection): VertexRDD[Array[(VertexID, VD)]]
  def mapReduceTriplets[A: ClassTag](
      mapFunc: EdgeTriplet[VD, ED] => Iterator[(VertexID, A)],
      reduceFunc: (A, A) => A)
    : VertexRDD[A]

  // Iterative graph-parallel computation
  def pregel[A](initialMsg: A, maxIterations: Int, activeDirection: EdgeDirection)(
      vprog: (VertexID, VD, A) => VD,
      sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexID,A)],
      mergeMsg: (A, A) => A)
    : Graph[VD, ED]

  // Basic graph algorithms
  def pageRank(tol: Double, resetProb: Double = 0.15): Graph[Double, Double]
  def connectedComponents(): Graph[VertexID, ED]
  def triangleCount(): Graph[Int, ED]
  def stronglyConnectedComponents(numIter: Int): Graph[VertexID, ED]
}
```

这些操作由Graph和GraphOps区分开来。例如我们可以计算各个定点的入度（in-degree）：

```
al inDegrees: VertexRDD[Int] = graph.inDegrees

```

在这个例子中，`graph.inDegree`操作返回的是一个`VertexRDD[Int]`。那如果要合并入度和出度，我们会用到一系列通用的图操作。首先我们创建一个叫User的类来更好地组织顶点属性，构建新的user属性图

```
// Define a class to more clearly model the user property
case class User(name: String, age: Int, inDeg: Int, outDeg: Int)
// Create a user Graph
val initialUserGraph: Graph[User, Int] = graph.mapVertices{ case (id, (name, age)) => User(name, age, 0, 0) }

```

注意到我们把每个顶点的的入度和出度都定义了0，然后我们来加入新的入度和出度信息。

```
// Fill in the degree information
val userGraph = initialUserGraph.outerJoinVertices(initialUserGraph.inDegrees) {
  case (id, u, inDegOpt) => User(u.name, u.age, inDegOpt.getOrElse(0), u.outDeg)
}.outerJoinVertices(initialUserGraph.outDegrees) {
  case (id, u, outDegOpt) => User(u.name, u.age, u.inDeg, outDegOpt.getOrElse(0))
}

```

在此我们使用的是`outerJoinVertices`方法

```
def outerJoinVertices[U, VD2](other: RDD[(VertexID, U)])
      (mapFunc: (VertexID, VD, Option[U]) => VD2)
    : Graph[VD2, ED]

```

我们发现`outerJoinVertices`有两个参数list，第一个包含了一个顶点值的RDD，第二个获得了一个由id、属性和Optionak matching value到一个新的顶点值。

使用`degreeGraph`打印出相互like的人的数目

```
for ((id, property) <- userGraph.vertices.collect) {
  println(s"User $id is called ${property.name} and is liked by ${property.inDeg} people.")
}

```

打印出有相同like数目的人

```
userGraph.vertices.filter {
  case (id, u) => u.inDeg == u.outDeg
}.collect.foreach {
  case (id, property) => println(property.name)
}

```

##The Map Reduce Triplets Operator

假如我们想要找到各个用户嘴年长的like者，`mapReduceTriplets `操作可以帮助我们，这允许邻居节点的聚合：

```
class Graph[VD, ED] {
  def mapReduceTriplets[MsgType](
      // Function from an edge triplet to a collection of messages (i.e., Map)
      map: EdgeTriplet[VD, ED] => Iterator[(VertexId, MsgType)],
      // Function that combines messages to the same vertex (i.e., Reduce)
      reduce: (MsgType, MsgType) => MsgType)
    : VertexRDD[MsgType]
}

```

这map function应用到每个三重边上,生成信息到相应的边上。这Reduce function聚集信息到同一个顶点，这个操作最后产生到一个VertexRDD，包含了每个顶点上所有聚集消息，然后我们就可以找到每个人最年长的like者

```
// Find the oldest follower for each user
val oldestFollower: VertexRDD[(String, Int)] = userGraph.mapReduceTriplets[(String, Int)](
  // For each edge send a message to the destination vertex with the attribute of the source vertex
  edge => Iterator((edge.dstId, (edge.srcAttr.name, edge.srcAttr.age))),
  // To combine messages take the message for the older follower
  (a, b) => if (a._2 > b._2) a else b
  )

```

产生结果：

```
David is the oldest follower of Alice.
Charlie is the oldest follower of Bob.
Ed is the oldest follower of Charlie.
Bob is the oldest follower of David.
Ed does not have any followers.
Charlie is the oldest follower of Fran.

```

```
userGraph.vertices.leftJoin(oldestFollower) { (id, user, optOldestFollower) =>
  optOldestFollower match {
    case None => s"${user.name} does not have any followers."
    case Some((name, age)) => s"${name} is the oldest follower of ${user.name}."
  }
}.collect.foreach { case (id, str) => println(str) }

```

可以找到每个用户like者的平均年纪

```
val averageAge: VertexRDD[Double] = userGraph.mapReduceTriplets[(Int, Double)](
  // map function returns a tuple of (1, Age)
  edge => Iterator((edge.dstId, (1, edge.srcAttr.age.toDouble))),
  // reduce function combines (sumOfFollowers, sumOfAge)
  (a, b) => ((a._1 + b._1), (a._2 + b._2))
  ).mapValues((id, p) => p._2 / p._1)

// Display the results
userGraph.vertices.leftJoin(averageAge) { (id, user, optAverageAge) =>
  optAverageAge match {
    case None => s"${user.name} does not have any followers."
    case Some(avgAge) => s"The average age of ${user.name}\'s followers is $avgAge."
  }
}.collect.foreach { case (id, str) => println(str) }

```

##SubGraph

假设我们想要去研究每个30岁及以上的用户的组织结构，为了支持这种类型的分析，GraphX提供了subgraph操作来返回满足特定条件的顶点和边的集合。如下就是限定了30岁及以上的用户

```
val olderGraph = userGraph.subgraph(vpred = (id, user) => user.age >= 30)

```
我们可以在自己的图上测试一下

```
// compute the connected components
val cc = olderGraph.connectedComponents

// display the component id of each user:
olderGraph.vertices.leftJoin(cc.vertices) {
  case (id, user, comp) => s"${user.name} is in component ${comp.get}"
}.collect.foreach{ case (id, str) => println(str) }

```

##Constructing an End-to-End Graph Analytics Pipeline on Real Data

在Wikipedia上的link data做一个测试

Graphx需要一个Kryo serializer来激活最大性能
`http://<MASTER_URL>:4040/environment/ and checking the spark.serializer property:`

![Alt text](http://ampcamp.berkeley.edu/4/exercises/img/spark_shell_ui_kryo.png)

退出现有的Spark-shell，打开一个编辑器, 添加到`/root/spark/conf/spark-env.sh`

```
SPARK_JAVA_OPTS+='
 -Dspark.serializer=org.apache.spark.serializer.KryoSerializer
 -Dspark.kryo.registrator=org.apache.spark.graphx.GraphKryoRegistrator '
export SPARK_JAVA_OPTS
```

或者可以偷懒直接复制到终端Term里：

`echo -e "SPARK_JAVA_OPTS+=' -Dspark.serializer=org.apache.spark.serializer.KryoSerializer -Dspark.kryo.registrator=org.apache.spark.graphx.GraphKryoRegistrator ' \nexport SPARK_JAVA_OPTS" >> /root/spark/conf/spark-env.sh`


然后更新命令

`/root/spark-ec2/copy-dir.sh /root/spark/conf`

最后重启cluster

```
/root/spark/sbin/stop-all.sh
sleep 3
/root/spark/sbin/start-all.sh
```

在重新启动Spark-Shell之后，如果访问`http://<MASTER_URL>:4040/environment/ `，其中serializer property `spark.serializer`属性应该被设置成` org.apache.spark.serializer.KryoSerializer`。

#####Getting Started (Again)

开启Spark shell

`/root/spark/bin/spark-shell`

导入包文件

```
import org.apache.spark.graphx._
import org.apache.spark.rdd.RDD
```
#####Load the Wikipedia Article

第一步是提取原生的数据到Spark，载入数据岛一个RDD

```
// We tell Spark to cache the result in memory so we won't have to repeat the
// expensive disk IO. We coalesce down to 20 partitions to avoid excessive
// communication.
val wiki: RDD[String] = sc.textFile("/wiki_links/part*").coalesce(20)

```

#####Look at the First Article
查看第一个文章内容

```
wiki.first
// res0: String = AccessibleComputing      [[Computer accessibility]]

```

#####Clean the Data

接下去是清理数据并提取图结构。每一行第一个词是文章题目，剩下的内容包含了文章中的链接。

现在我们开始使用我们已经得到的图结构去做第一轮的数据清洗，我们定义一个`Article`类来储存不同部分的文章。

```
// Define the article class
case class Article(val title: String, val body: String)

// Parse the articles
val articles = wiki.map(_.split('\t')).
  // two filters on article format
  filter(line => (line.length > 1 && !(line(1) contains "REDIRECT"))).
  // store the results in an object for easier access
  map(line => new Article(line(0).trim, line(1).trim)).cache

```

我们可以查看还有多少文章剩余，计算和缓存已经清洗完的RDD：`articles.count`

#####Making a Vertex RDD

现在我们可以创建新的RDD了，首先，natural vertex属性是文章的标题，我们同样打算利用哈希定义一种从文章题目到vertexID的映射算法

```
// Hash function to assign an Id to each article
def pageHash(title: String): VertexId = {
  title.toLowerCase.replace(" ", "").hashCode.toLong
}
// The vertices with id and article title:
val vertices = articles.map(a => (pageHash(a.title), a.title)).cache
```

同样我们可以计算个数`vertices.count`

#####Making the Edge RDD

接下去扩展边结构，我们知道wikipedia语法结构指明一个链接的格式像`[[Article We Are Linking To]]`， 我们就可以写一个一般表达式通过“[[”和“]]”来封闭，然后应用进去。

```
val pattern = "\\[\\[.+?\\]\\]".r
val edges: RDD[Edge[Double]] = articles.flatMap { a =>
  val srcVid = pageHash(a.title)
  pattern.findAllIn(a.body).map { link =>
    val dstVid = pageHash(link.replace("[[", "").replace("]]", ""))
    Edge(srcVid, dstVid, 1.0)
  }
}

```

##### Making the Graph

现在我们其实已经建立好需要的Graph了，接下去在不同views里的操作，可以自行切换调用Graph Constructor如vertex RDD，edge RDD以及一个默认的vertex attribute。

`val graph = Graph(vertices, edges, "").subgraph(vpred = {(v, d) => d.nonEmpty}).cache`

我们可以让Graph计算属性的数量：`graph.vertices.count`

`graph.triplets.count`

##### Running PageRank on Wikipedia

如果我们要做PageRank计算

`val prGraph = graph.staticPageRank(5).cache`

`Graph.staticPageRank`返回的是一个顶点属性就是每个页面对应的PageRank值的图，就是说结果图没有原先图种各个顶点的属性，包括题目。但我们还保留着原图，所以可以做一个join操作得到*prGraph*有相应的PageRank值和原先的信息。之后还可以做别的计算，比如找出十个最重要的顶点，打印出相应的题目，这一系列操作如下：

```
val titleAndPrGraph = graph.outerJoinVertices(prGraph.vertices) {
  (v, title, rank) => (rank.getOrElse(0.0), title)
}

titleAndPrGraph.vertices.top(10) {
  Ordering.by((entry: (VertexId, (Double, String))) => entry._2._1)
}.foreach(t => println(t._2._2 + ": " + t._2._1))

```


最后,让我们来找出在subgraph中在标题中提及了Berkeley的最重要的一个page

```
val berkeleyGraph = graph.subgraph(vpred = (v, t) => t.toLowerCase contains "berkeley")

val prBerkeley = berkeleyGraph.staticPageRank(5).cache

berkeleyGraph.outerJoinVertices(prBerkeley.vertices) {
  (v, title, r) => (r.getOrElse(0.0), title)
}.vertices.top(10) {
  Ordering.by((entry: (VertexId, (Double, String))) => entry._2._1)
}.foreach(t => println(t._2._2 + ": " + t._2._1))

```

 - - -

- [ampCamp Graphx](http://ampcamp.berkeley.edu/4/exercises/graph-analytics-with-graphx.html)
- [Graphx Official Guide](http://spark.incubator.apache.org/docs/latest/graphx-programming-guide.html)



