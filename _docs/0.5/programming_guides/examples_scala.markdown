---
layout: inner_docs_v05
title:  "Scala Example Programs"
sublinks:
  - {anchor: "wordcount", title: "Word Count"}
  - {anchor: "page_rank", title: "Page Rank"}
  - {anchor: "connected_components", title: "Connected Components"}
  - {anchor: "relational", title: "Relational Query"}
---

## Scala Example Programs

<p class="lead">
The following example programs showcase different applications of Stratosphere from simple word counting to graph algorithms.
The code samples illustrate the use of **[Stratosphere's Scala API]({{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/scala.html)**. 

<p class="lead">
The full source code of the following and more examples can be found in the **[stratosphere-scala-examples](https://github.com/stratosphere/stratosphere/tree/release-{{site.current_stable}}/stratosphere-examples/stratosphere-scala-examples)** module.

<section id="wordcount">
<div class="page-header"><h2>Word Count</h2></div>

WordCount is the "Hello World" of Big Data processing systems. It computes the frequency of words in a text collection. The algorithm works in two steps: First, the texts are splits the text to individual words. Second, the words are grouped and counted.

<div id="wordcount-scala-code">
{% highlight scala %}
// read input data
val input = TextFile(textInput)

// tokenize words
val words = input.flatMap { _.split(" ") map { (_, 1) } }

// count by word
val counts = words.groupBy { case (word, _) => word }
  .reduce { (w1, w2) => (w1._1, w1._2 + w2._2) }

val output = counts.write(wordsOutput, CsvOutputFormat()))
{% endhighlight %}
</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-scala-examples/src/main/scala/eu/stratosphere/examples/scala/wordcount/WordCount.scala">WordCount example</a> implements the above described algorithm with input parameters:<code>&lt;degree of parallelism&gt;, &lt;text input path&gt;, &lt;output path&gt;</code>.<br>
As test data, any text file will do.

</section>


<section id="page_rank">
<div class="page-header"><h2>Page Rank</h2></div>

The PageRank algorithm computes the "importance" of pages in a graph defined by links, which point from one pages to another page. It is an iterative graph algorithm, which means that it repeatedly applies the same computation. In each iteration, each page distributes its current rank over all its neighbors, and compute its new rank as a taxed sum of the ranks it received from its neighbors. The PageRank algorithm was popularized by the Google search engine which uses the importance of webpages to rank the results of search queries.

<p>
In this simple example, PageRank is implemented with a <a href="{{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/java.html#iterations">bulk iteration</a> and a fixed number of iterations.

<div id="pagerank-scala-code">
{% highlight scala %}
// cases classes so we have named fields
case class PageWithRank(pageId: Long, rank: Double)
case class Edge(from: Long, to: Long, transitionProbability: Double)

// constants for the page rank formula
val dampening = 0.85
val randomJump = (1.0 - dampening) / NUM_VERTICES
val initialRank = 1.0 / NUM_VERTICES
  
// read inputs
val pages = DataSource(verticesPath, CsvInputFormat[Long]())
val edges = DataSource(edgesPath, CsvInputFormat[Edge]())

// assign initial rank
val pagesWithRank = pages map { p => PageWithRank(p, initialRank) }

// the iterative computation
def computeRank(ranks: DataSet[PageWithRank]) = {

    // send rank to neighbors
    val ranksForNeighbors = ranks join edges
        where { _.pageId } isEqualTo { _.from }
        map { (p, e) => (e.to, p.rank * e.transitionProbability) }
    
    // gather ranks per vertex and apply page rank formula
    ranksForNeighbors .groupBy { case (node, rank) => node }
                      .reduce { (a, b) => (a._1, a._2 + b._2) }
                      .map {case (node, rank) => PageWithRank(node, rank * dampening + randomJump) }
}

// invoke iteratively
val finalRanks = pagesWithRank.iterate(numIterations, computeRank)
val output = finalRanks.write(outputPath, CsvOutputFormat())
{% endhighlight %}
</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-scala-examples/src/main/scala/eu/stratosphere/examples/scala/graph/PageRank.scala">PageRank program</a> implements the above example.
It requires the following parameters to run: <code>&lt;pages input path&gt;, &lt;link input path&gt;, &lt;output path&gt; &lt;num pages&gt;, &lt;num iterations&gt;</code>.

<p>
Input files are plain text files and must be formatted as follows:
<ul>
<li>Pages represented as an (long) ID separated by new-line characters.<br> 
    For example <code>"1\n2\n12\n42\n63\n"</code> gives five pages with IDs 1, 2, 12, 42, and 63.
</li>
<li>Links are represented as pairs of page IDs which are separated by space characters. Links are separated by new-line characters.<br>
    For example <code>"1 2\n2 12\n1 12\n42 63\n"</code> gives four (directed) links (1)->(2), (2)->(12), (1)->(12), and (42)->(63).<br>
    For this simple implementation it is required that each page has at least one incoming and one outgoing link (a page can point to itself).
</li>
</ul>

</section>

<section id="connected_components">
<div class="page-header"><h2>Connected Components</h2></div>

The Connected Components algorithm identifies parts of a larger graph which are connected by assigning all vertices in the same connected part the same component ID. Similar to PageRank, Connected Components is an iterative algorithm. In each step, each vertex propagates its current component ID to all its neighbors. A vertex accepts the component ID from a neighbor, if it is smaller than its own component ID.<br>

This implementation uses a <a href="{{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/java.html#iterations">delta iteration</a>: Vertices that have not changed their component id do not participate in the next step. This yields much better performance, because the later iterations typically deal only with a few outlier vertices.

<div id="cc-scala-code">

{% highlight scala %}
// define case classes
case class VertexWithComponent(vertex: Long, componentId: Long)
case class Edge(from: Long, to: Long)

// get input data
val vertices = DataSource(verticesPath, CsvInputFormat[Long]())
val directedEdges = DataSource(edgesPath, CsvInputFormat[Edge]())

// assign each vertex its own ID as component ID
val initialComponents = vertices map { v => VertexWithComponent(v, v) }
val undirectedEdges = directedEdges flatMap { e => Seq(e, Edge(e.to, e.from)) }

def propagateComponent(s: DataSet[VertexWithComponent], ws: DataSet[VertexWithComponent]) = {
  val allNeighbors = ws join undirectedEdges
        where { _.vertex } isEqualTo { _.from }
        map { (v, e) => VertexWithComponent(e.to, v.componentId ) }
    
    val minNeighbors = allNeighbors groupBy { _.vertex } reduceGroup { cs => cs minBy { _.componentId } }

    // updated solution elements == new workset
    val s1 = s join minNeighbors
        where { _.vertex } isEqualTo { _.vertex }
        flatMap { (curr, candidate) =>
            if (candidate.componentId < curr.componentId) Some(candidate) else None
        }

  (s1, s1)
}

val components = initialComponents.iterateWithDelta(initialComponents, { _.vertex }, propagateComponent,
                    maxIterations)
val output = components.write(componentsOutput, CsvOutputFormat())
{% endhighlight %}

</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-scala-examples/src/main/scala/eu/stratosphere/examples/scala/graph/ConnectedComponents.scala">ConnectedComponents program</a> implements the above example.
It requires the following parameters to run: <code>&lt;vertex input path&gt;, &lt;edge input path&gt;, &lt;output path&gt; &lt;max num iterations&gt; &lt;degree of parallelism&gt;</code>.

<p>
Input files are plain text files and must be formatted as follows:
<ul>
<li>Vertices represented as IDs and separated by new-line characters.<br> 
    For example <code>"1\n2\n12\n42\n63\n"</code> gives five vertices (1), (2), (12), (42), and (63).
</li>
<li>Edges are represented as pairs for vertex IDs which are separated by space characters. Edges are separated by new-line characters.<br>
    For example <code>"1 2\n2 12\n1 12\n42 63\n"</code> gives four (undirected) edges (1)-(2), (2)-(12), (1)-(12), and (42)-(63).
</li>
</ul>

</section>


<section id="relational">
<div class="page-header"><h2>Relational Query</h2></div>

The Relational Query example assumes two tables, one with `orders` and the other with `lineitems` as specified by the [TPC-H decision support benchmark](http://www.tpc.org/tpch/). TPC-H is a standard benchmark in the database industry. See below for instructions how to generate the input data.

<p>
The example implements the following SQL query.


{% highlight sql %}
SELECT l_orderkey, o_shippriority, sum(l_extendedprice) as revenue
    FROM orders, lineitem
WHERE l_orderkey = o_orderkey
    AND o_orderstatus = "F" 
    AND YEAR(o_orderdate) > 1993
    AND o_orderpriority LIKE "5%"
GROUP BY l_orderkey, o_shippriority;
{% endhighlight %}

The Stratosphere Scala program, which implements the above query looks as follows.

<div class="relational-scala-code">

{% highlight scala %}
// --- define some custom classes to address fields by name ---
case class Order(orderId: Int, status: Char, date: String, orderPriority: String, shipPriority: Int)
case class LineItem(orderId: Int, extendedPrice: Double)
case class PrioritizedOrder(orderId: Int, shipPriority: Int, revenue: Double)

val orders = DataSource(ordersInputPath, DelimitedInputFormat(parseOrder))
val lineItem2600s = DataSource(lineItemsInput, DelimitedInputFormat(parseLineItem))

val filteredOrders = orders filter { o => o.status == "F" && o.date.substring(0, 4).toInt > 1993 && o.orderPriority.startsWith("5") }

val prioritizedItems = filteredOrders join lineItems
    where { _.orderId } isEqualTo { _.orderId } // join on the orderIds
    map { (o, li) => PrioritizedOrder(o.orderId, o.shipPriority, li.extendedPrice) }

val prioritizedOrders = prioritizedItems
    groupBy { pi => (pi.orderId, pi.shipPriority) } 
    reduce { (po1, po2) => po1.copy(revenue = po1.revenue + po2.revenue) }

val output = prioritizedOrders.write(ordersOutput, CsvOutputFormat(formatOutput))
{% endhighlight %}

</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-scala-examples/src/main/scala/eu/stratosphere/examples/scala/relational/RelationalQuery.scala">Relational Query program</a> implements the above query. <br>
It requires the following parameters to run: <code>&lt;orders input path&gt;, &lt;lineitem input path&gt;, &lt;output path&gt; &lt;degree of parallelism&gt;</code>.<br>

The orders and lineitem files can be generated using the [TPC-H benchmark](http://www.tpc.org/tpch/) suite's data generator tool (DBGEN). 
Take the following steps to generate arbitrary large input files for the provided Stratosphere programs:

1.  Download and unpack DBGEN
2.  Make a copy of *makefile.suite* called *Makefile* and perform the following changes:

{% highlight bash %}
# The Stratosphere program was tested with DB2 data format
DATABASE = DB2
MACHINE  = LINUX
WORKLOAD = TPCH
# according to your compiler, mostly gcc
CC       = gcc
{% endhighlight %}

1.  Build DBGEN using *make*
2.  Generate lineitem and orders relations using dbgen. A scale factor
    (-s) of 1 results in a generated data set with about 1 GB size.

{% highlight bash %}
./dbgen -T o -s 1
{% endhighlight %}
</section>