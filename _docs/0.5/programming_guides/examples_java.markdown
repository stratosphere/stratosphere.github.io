---
layout: inner_docs_v05
title:  "Java Example Programs"
sublinks:
  - {anchor: "wordcount", title: "Word Count"}
  - {anchor: "page_rank", title: "Page Rank"}
  - {anchor: "connected_components", title: "Connected Components"}
  - {anchor: "relational", title: "Relational Query"}
---

## Java Example Programs

<p class="lead">
The following example programs showcase different applications of Stratosphere from simple word counting to graph algorithms.
The code samples illustrate the use of **[Stratosphere's Java API]({{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/java.html)**. 

<p class="lead">
The full source code of the following and more examples can be found in the **[stratosphere-java-examples](https://github.com/stratosphere/stratosphere/tree/release-{{site.current_stable}}/stratosphere-examples/stratosphere-java-examples)** module.

<section id="wordcount">
<div class="page-header"><h2>Word Count</h2></div>

WordCount is the "Hello World" of Big Data processing systems. It computes the frequency of words in a text collection. The algorithm works in two steps: First, the texts are splits the text to individual words. Second, the words are grouped and counted.

<div id="wordcount-java-code">
{% highlight java %}

// get input data
DataSet<String> text = getTextDataSet(env);

DataSet<Tuple2<String, Integer>> counts = 
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new Tokenizer())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .aggregate(Aggregations.SUM, 1);

counts.writeAsCsv(outputPath, "\n", " ");

// User-defined functions
public static final class Tokenizer extends FlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
        // normalize and split the line
        String[] tokens = value.toLowerCase().split("\\W+");
        
        // emit the pairs
        for (String token : tokens) {
            if (token.length() > 0) {
                out.collect(new Tuple2<String, Integer>(token, 1));
            }   
        }
    }
}
{% endhighlight %}

</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/wordcount/WordCount.java">WordCount example</a> implements the above described algorithm with input parameters: <code>&lt;text input path&gt;, &lt;output path&gt;</code>.<br>
As test data, any text file will do.

</section>


<section id="page_rank">
<div class="page-header"><h2>Page Rank</h2></div>

The PageRank algorithm computes the "importance" of pages in a graph defined by links, which point from one pages to another page. It is an iterative graph algorithm, which means that it repeatedly applies the same computation. In each iteration, each page distributes its current rank over all its neighbors, and compute its new rank as a taxed sum of the ranks it received from its neighbors. The PageRank algorithm was popularized by the Google search engine which uses the importance of webpages to rank the results of search queries.

<p>
In this simple example, PageRank is implemented with a <a href="{{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/java.html#iterations">bulk iteration</a> and a fixed number of iterations.

<div id="pagerank-java-code">
{% highlight java %}
// get input data
DataSet<Tuple2<Long, Double>> pagesWithRanks = getPagesWithRanksDataSet(env);
DataSet<Tuple2<Long, Long[]>> pageLinkLists = getLinksDataSet(env);

// set iterative data set
IterativeDataSet<Tuple2<Long, Double>> iteration = pagesWithRanks.iterate(maxIterations);

DataSet<Tuple2<Long, Double>> newRanks = iteration
        // join pages with outgoing edges and distribute rank
        .join(pageLinkLists).where(0).equalTo(0).flatMap(new JoinVertexWithEdgesMatch())
        // collect and sum ranks
        .groupBy(0).aggregate(SUM, 1)
        // apply dampening factor
        .map(new Dampener(DAMPENING_FACTOR, numPages));

DataSet<Tuple2<Long, Double>> finalPageRanks = iteration.closeWith(
        newRanks, 
        newRanks.join(iteration).where(0).equalTo(0)
        // termination condition
        .filter(new EpsilonFilter()));

finalPageRanks.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class JoinVertexWithEdgesMatch 
                    extends FlatMapFunction<Tuple2<Tuple2<Long, Double>, Tuple2<Long, Long[]>>, 
                                            Tuple2<Long, Double>> {

    @Override
    public void flatMap(Tuple2<Tuple2<Long, Double>, Tuple2<Long, Long[]>> value, 
                        Collector<Tuple2<Long, Double>> out) {
        Long[] neigbors = value.f1.f1;
        double rank = value.f0.f1;
        double rankToDistribute = rank / ((double) neigbors.length);
            
        for (int i = 0; i < neigbors.length; i++) {
            out.collect(new Tuple2<Long, Double>(neigbors[i], rankToDistribute));
        }
    }
}

public static final class Dampener extends MapFunction<Tuple2<Long,Double>, Tuple2<Long,Double>> {
    private final double dampening, randomJump;

    public Dampener(double dampening, double numVertices) {
        this.dampening = dampening;
        this.randomJump = (1 - dampening) / numVertices;
    }

    @Override
    public Tuple2<Long, Double> map(Tuple2<Long, Double> value) {
        value.f1 = (value.f1 * dampening) + randomJump;
        return value;
    }
}

public static final class EpsilonFilter 
                    extends FilterFunction<Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>>> {

    @Override
    public boolean filter(Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>> value) {
        return Math.abs(value.f0.f1 - value.f1.f1) > EPSILON;
    }
}

{% endhighlight %}

</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/graph/PageRankBasic.java">PageRank program</a> implements the above example.
It requires the following parameters to run: <code>&lt;pages input path&gt;, &lt;links input path&gt;, &lt;output path&gt; &lt;num pages&gt;, &lt;num iterations&gt;</code>.

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

This implementation uses a <a href="{{site.baseurl}}/docs/{{site.current_stable}}/programming_guides/java.html#iterations">delta iteration</a>: Vertices that have not changed their component ID do not participate in the next step. This yields much better performance, because the later iterations typically deal only with a few outlier vertices.

<div id="cc-java-code">

{% highlight java %}

// read vertex and edge data
DataSet<Long> vertices = getVertexDataSet(env);
DataSet<Tuple2<Long, Long>> edges = getEdgeDataSet(env).flatMap(new UndirectEdge());

// assign the initial component IDs (equal to the vertex ID)
DataSet<Tuple2<Long, Long>> verticesWithInitialId = vertices.map(new DuplicateValue<Long>());
        
// open a delta iteration
DeltaIteration<Tuple2<Long, Long>, Tuple2<Long, Long>> iteration =
        verticesWithInitialId.iterateDelta(verticesWithInitialId, maxIterations, 0);

// apply the step logic: 
DataSet<Tuple2<Long, Long>> changes = iteration.getWorkset()
        // join with the edges
        .join(edges).where(0).equalTo(0).with(new NeighborWithComponentIDJoin())
        // select the minimum neighbor component ID
        .groupBy(0).aggregate(Aggregations.MIN, 1)
        // update if the component ID of the candidate is smaller
        .join(iteration.getSolutionSet()).where(0).equalTo(0)
        .flatMap(new ComponentIdFilter());

// close the delta iteration (delta and new workset are identical)
DataSet<Tuple2<Long, Long>> result = iteration.closeWith(changes, changes);

// emit result
result.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class DuplicateValue<T> extends MapFunction<T, Tuple2<T, T>> {
    
    @Override
    public Tuple2<T, T> map(T vertex) {
        return new Tuple2<T, T>(vertex, vertex);
    }
}

public static final class UndirectEdge 
                    extends FlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {
    Tuple2<Long, Long> invertedEdge = new Tuple2<Long, Long>();
    
    @Override
    public void flatMap(Tuple2<Long, Long> edge, Collector<Tuple2<Long, Long>> out) {
        invertedEdge.f0 = edge.f1;
        invertedEdge.f1 = edge.f0;
        out.collect(edge);
        out.collect(invertedEdge);
    }
}

public static final class NeighborWithComponentIDJoin 
                    extends JoinFunction<Tuple2<Long, Long>, Tuple2<Long, Long>, Tuple2<Long, Long>> {

    @Override
    public Tuple2<Long, Long> join(Tuple2<Long, Long> vertexWithComponent, Tuple2<Long, Long> edge) {
        return new Tuple2<Long, Long>(edge.f1, vertexWithComponent.f1);
    }
}

public static final class ComponentIdFilter 
                    extends FlatMapFunction<Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>>, 
                                            Tuple2<Long, Long>> {

    @Override
    public void flatMap(Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>> value, 
                        Collector<Tuple2<Long, Long>> out) {
        if (value.f0.f1 < value.f1.f1) {
            out.collect(value.f0);
        }
    }
}

{% endhighlight %}

</div>

The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/graph/ConnectedComponents.java">ConnectedComponents program</a> implements the above example.
It requires the following parameters to run: <code>&lt;vertex input path&gt;, &lt;edge input path&gt;, &lt;output path&gt; &lt;max num iterations&gt;</code>.

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

The Stratosphere Java program, which implements the above query looks as follows.

<div class="relational-java-code">

{% highlight java %}

// get orders data set: (orderkey, orderstatus, orderdate, orderpriority, shippriority)
DataSet<Tuple5<Integer, String, String, String, Integer>> orders = getOrdersDataSet(env);
// get lineitem data set: (orderkey, extendedprice)
DataSet<Tuple2<Integer, Double>> lineitems = getLineitemDataSet(env);

// orders filtered by year: (orderkey, custkey)
DataSet<Tuple2<Integer, Integer>> ordersFilteredByYear =
        // filter orders
        orders.filter(
            new FilterFunction<Tuple5<Integer, String, String, String, Integer>>() {
                @Override
                public boolean filter(Tuple5<Integer, String, String, String, Integer> t) {
                    // status filter
                    if(!t.f1.equals(STATUS_FILTER)) {
                        return false;
                    // year filter
                    } else if(Integer.parseInt(t.f2.substring(0, 4)) <= YEAR_FILTER) {
                        return false;
                    // order priority filter
                    } else if(!t.f3.startsWith(OPRIO_FILTER)) {
                        return false;
                    }
                    return true;
                }
            })
        // project fields out that are no longer required
        .project(0,4).types(Integer.class, Integer.class);

// join orders with lineitems: (orderkey, shippriority, extendedprice)
DataSet<Tuple3<Integer, Integer, Double>> lineitemsOfOrders = 
        ordersFilteredByYear.joinWithHuge(lineitems)
                            .where(0).equalTo(0)
                            .projectFirst(0,1).projectSecond(1)
                            .types(Integer.class, Integer.class, Double.class);

// extendedprice sums: (orderkey, shippriority, sum(extendedprice))
DataSet<Tuple3<Integer, Integer, Double>> priceSums = 
        // group by order and sum extendedprice
        lineitemsOfOrders.groupBy(0,1).aggregate(Aggregations.SUM, 2);

// emit result
priceSums.writeAsCsv(outputPath);

{% endhighlight %}

</div>


The <a href="https://github.com/stratosphere/stratosphere/blob/release-{{site.current_stable}}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/relational/RelationalQuery.java">Relational Query program</a> implements the above query. <br>
It requires the following parameters to run: <code>&lt;orders input path&gt;, &lt;lineitem input path&gt;, &lt;output path&gt;</code>.<br>

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