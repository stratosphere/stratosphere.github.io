--- 
layout: inner_docs_v05
title: Java API
sublinks:
  - {anchor: "introduction", title: "Introduction"}
  - {anchor: "example", title: "Example Program"}
  - {anchor: "linking", title: "Linking"}
  - {anchor: "skeleton", title: "Skeleton Program"}
  - {anchor: "types", title: "Data Types"}
  - {anchor: "transformations", title: "Data Transformations"}
  - {anchor: "data_sources", title: "Data Sources"}
  - {anchor: "data_sinks", title: "Data Sinks"}
  - {anchor: "debugging", title: "Debugging"}
  - {anchor: "iterations", title: "Iteration Operators"}
  - {anchor: "annotations", title: "Annotations"}
  - {anchor: "broadcast_variables", title: "Broadcast Variables"}
  - {anchor: "packaging", title: "Program Packaging"}
  - {anchor: "accumulators_counters", title: "Accumulators &amp; Counters"}
  - {anchor: "execution_plan", title: "Execution Plans"}
---

<div class="panel panel-default"><div class="panel-body">Please note that parts of the documentation are out of sync. We are in the process of updating everything to reflect the changes of the upcoming release. If you have any questions, we are happy to help on our <a href="{{site.baseurl}}/project/contact/">Mailinglist</a> or on <a href="https://github.com/stratosphere/stratosphere/issues">GitHub</a>.</div></div>

Java API
========

<section id="introduction">
Introduction
------------

Analysis programs in Stratosphere are regular Java Programs that implement transformations on data sets (e.g., filtering, , mapping, joining, grouping). The data sets are initially created from certain sources (e.g., by reading files, or from collections). The results are returned by sinks, which may for example write the data to (distributed) files, or print it to the command line. The sections on the [program skeleton](#skeleton) and [transformations](#transformations) show the general template of a program and describe the available transformations.

Stratosphere programs can run in a variety of contexts, for example locally as standalone programs, locally embedded in other programs, or on clusters of many machines (see [program skeleton](#skeleton) for how to define different environments). All programs are executed lazily: When the program is run and the transformation method on the data set is invoked, it creates a specific transformation operation. That transformation operation is only executed once program execution is triggered on the environment. Whether the program is executed locally or on a cluster depends on the environment of the program.

The Java API is strongly typed: All data sets and transformations accept typed elements. This allows to catch typing errors very early and supports safe refactoring of programs.
</section>

<section id="example">
Example Program
---------------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>


<section id="linking">
Linking with Stratosphere
-------------------------

To write programs with Stratosphere, you need to include Stratosphere’s Java API library in your project. 

The simplest way to do this is to use the [quickstart scripts]({{site.baseurl}}/quickstart/java.html). They create a blank project from a template (called Maven Archetype), which sets up everything for you. To manually create the project, you can use the archetype and create a project by calling:

{% highlight bash %}
mvn archetype:generate /
    -DarchetypeGroupId=eu.stratosphere /
    -DarchetypeArtifactId=quickstart-java /
    -DarchetypeVersion={{site.docs_05_stable}}
{% endhighlight %}

If you want to add Stratosphere to an existing Maven project, add the following entry to your *dependencies* in the *pom.xml* file of your project:

{% highlight xml %}
<dependency>
  <groupId>eu.stratosphere</groupId>
  <artifactId>stratosphere-java</artifactId>
  <version>{{site.docs_05_stable}}</version>
</dependency>
<dependency>
  <groupId>eu.stratosphere</groupId>
  <artifactId>stratosphere-clients</artifactId>
  <version>{{site.docs_05_stable}}</version>
</dependency>
{% endhighlight %}

The *stratosphere-clients* dependency is only necessary for a local execution environment. You only need to include it, if you want to execute Stratosphere programs on your local machine (for example for testing or debugging).
</section>


<section id="skeleton">
Skeleton Program
----------------

As we already saw in the example, Stratosphere programs look like regular Java
programs with a `main()` method. Each program consists of the same basic parts:

1. Obtain an `ExecutionEnvironment`,
2. Load your data,
3. Specify transformations on this data,
4. Store the results of your computations, and
5. Execute your program on a cluster or on your local computer.

We will now give an overview of each of those steps but please refer
to the respective sections for more details. Note that all [core classes
of the Java API](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-java/src/main/java/eu/stratosphere/api/java) are found in the package `eu.stratosphere.api.java`.

The `ExecutionEnvironment` is the basis for all Stratosphere programs. You can
obtain one using these static methods on class `ExecutionEnvironment`:

```java
getExecutionEnvironment()

createLocalEnvironment()
createLocalEnvironment(int degreeOfParallelism)
    
createRemoteEnvironment(String host, int port, String... jarFiles)
createRemoteEnvironment(String host, int port, int degreeOfParallelism, String... jarFiles)
```

Typically, you only need to use `getExecutionEnvironment()`, since this
will do the right thing depending on the context: if you are executing
your program inside an IDE or as a regular Java program it will create
a local environment that will execute your program on your local machine. If
you created a jar file from you program, the Stratosphere cluster manager will
execute your main method and `getExecutionEnvironment()` will return
an execution environment for executing your program on a cluster.

For specifying data sources the execution environment has several methods
to read from files using various methods: you can just read them line by line,
as CSV files, or using completely custom data input formats. To just read
a text file you could use:

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<String> text = env.readTextFile("file:///path/to/file");
```

This will give you a `DataSet` on which you can then apply transformations. For
more information on data sources and input formats, please refer to
[Data Sources](#data_sources).

Once you have a `DataSet` you can apply transformations to create a new
`DataSet` which you can then write to a file, transform again, or
combine with other `DataSet`s. You apply transformations by calling
methods on `DataSet` with your own custom transformation function. For example,
map looks like this:

```java
DataSet<String> input = ...;

DataSet<Integer> tokenized = text.flatMap(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
```

This will create a new `DataSet` by converting every String in the original
set to an Integer. For more information and a list of all the transformations,
please refer to [Transformations](#transformations).

Once you have a `DataSet` that needs to be written to disk you call one
of these methods on `DataSet`:

```java
writeAsText(String path)
writeAsCsv(String path)
write(FileOutputFormat<T> outputFormat, String filePath)

print()
```

The last method is only useful for developing/debugging on a local machine,
it will output the contents of the `DataSet` to standard output.
The first two do as the name suggests, the third one can be used to specify a
custom data output format. Keep in mind, that these calls do not actually
write to a file yet. Only when your program is completely specified and you
call the `execute` method on your `ExecutionEnvironment` are all the
transformations executed and is data written to disk. Please refer
to [Data Sinks](#data_sinks) for more information on writing to files and also
about custom data output formats.

Once you specified the complete program you need to call `execute` on
the `ExecutionEnvironment`. This will either execute on your local
machine or submit your program for execution on a cluster, depending on
how you created the execution environment.
</section>

<section id="types">
Data Types
----------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="transformations">
Data Transformations
--------------------

Data transformations transform one or more `DataSet`s into another `DataSet`. Advanced data analysis programs can be constructed by chaining multiple transformations.

### Map

The Map transformation applies a user-defined `MapFunction` on each element of a DataSet.</br>
A `MapFunction` returns exactly one result element for each input element.

The following code transforms a `DataSet` of Integer pairs into a `DataSet` of Integers:

```java
// Map function that adds two integer values
public class IntAdder extends MapFunction<Tuple2<Integer, Integer>, Integer> {
  @Override
  public Integer map(Tuple2<Integer, Integer> in) throws Exception {
    return in.f0 + in.f1;
  }
}

// [...]
DataSet<Tuple2<Integer, Integer>> intPairs = // [...]
DataSet<Integer> intSums = intPairs.map(new IntAdder());
```

### FlatMap

The FlatMap transformation applies a user-defined `FlatMapFunction` on each element of a `DataSet`.</br>
A `FlatMapFunction` can return arbitrary many result elements (including none) for each input element.

The following code transforms a `DataSet` of text lines into a `DataSet` of words:

```java
// FlatMap function that tokenizes a String by whitespace characters and emits all String tokens.
public class Tokenizer extends FlatMapFunction<String, String> {
  @Override
  public void flatMap(String value, Collector<String> out) {
    for (String token : value.split("\\W")) {
      out.collect(token);
    }
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<String> words = textLines.flatMap(new Tokenizer());

```

### Filter

The Filter transformation applies a user-defined `FilterFunction` on each element of a `DataSet` and retains only those elements for which the `FilterFunction` returns `true`.</br>

The following code removes all Integers smaller than zero from a `DataSet`:

```java
// Filter function that filters out all Integers smaller than zero.
public class NaturalNumberFilter extends FilterFunction<Integer> {
  @Override
  public boolean filter(Integer number) throws Exception {
    return number >= 0;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> naturalNumbers = intNumbers.filter(new NaturalNumberFilter());
```

### Project

The Project transformation removes or moves tuple fields of a `Tuple` `DataSet`.</br>
Projections do not require the definition of a user function.

The following code shows different ways to use the Project transformations:

```java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// use field indexes to remove and move fields
DataSet<Tuple2<String, Integer>> out1 = in.project(2,0).types(String.class, Integer.class);
// use field masks to remove fields
DataSet<Tuple2<Integer, String>> out2 = in.project("101").types(Integer.class, String.class);
// use field flags to remove fields
DataSet<Tuple1<Double>> out3 = in.project(false, true, false).types(Double.class)
```

### Reduce on grouped DataSet

A `DataSet` can be grouped on one or more keys. Keys can be defined in two ways using

- a `KeySelector` function or 
- one or more field position keys. 

A Reduce transformation that is applied on a grouped `DataSet` reduces each group to a single element using a user-defined `ReduceFunction`.
For each group of input elements, a `ReduceFunction` successively combines pairs of elements into one element until only a single element for each group remains.

#### Reduce on DataSet grouped by KeySelector Function

A `KeySelector` function extracts a key value from each element of a `DataSet`. The extracted key value is used to group the `DataSet`.
The following code shows how to group and reduce a POJO `DataSet` using a `KeySelector` and a `ReduceFunction`.

```java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// Reduce function that sums Integer attributes of a POJO
public class WordCounter extends ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) throws Exception {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words;
DataSet<WC> wordCounts = words = // [...]
                          // DataSet grouping with inline-defined KeySelector function 
                          .groupBy(
                            new KeySelector<WC, String>() { 
                              public String getKey(WC wc) { return wc.word; } 
                            })
                          // reduce on grouped DataSet
                          .reduce(new WordCounter());
```

#### Reduce on DataSet grouped by Field Position Keys (Tuple DataSets only)

Field position keys specify one or more fields of a `Tuple` `DataSet` that are used as grouping keys.</br>
The following code shows how to use field position keys and apply a `ReduceFunction`.

```java
// Some Reduce function definition
public class MyTupleReducer extends ReduceFunction<Tuple3<String, Integer, Double>> {
  // [...]
}

// [...]
DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples = 
                                         tuples
                                         // group DataSet on first and second field of Tuple
                                         .groupBy(0,1)
                                         // reduce on grouped DataSet
                                         .reduce(new MyTupleReducer());
```

#### Reduce on sorted groups (Tuple DataSets only)

The elements within a group can be sorted using the GroupSort feature. Right now, GroupSort is only supported for `Tuple` `DataSet`s.

```java
// Some Reduce function definition
public class MyTupleReducer extends ReduceFunction<Tuple3<String, Integer, Double>> {
  // [...]
}

DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples = 
                                         tuples
                                         // group dataset on first tuple field
                                         .groupBy(0)
                                         // sort groups on second tuple field in descending order
                                         .sortGroup(1, Order.DESCENDING)
                                         // reduce on grouped dataset
                                         .reduce(new MyTupleReducer());
```

### GroupReduce on grouped DataSet

A `DataSet` can be grouped on one or more keys. Keys can be defined in two ways using

- a `KeySelector` function or 
- one or more field position keys. 

A GroupReduce transformation that is applied on a grouped `DataSet` calls user-defined `GroupReduceFunction` for each group.</br>
A `GroupReduceFunction` is called once for each group with an iterator over all elements of the group and can return an arbitrary number of result elements.

For the sake of brevity, we only show the GroupReduce transformation with grouping by field position keys. The usage of a key selection function and group sorting is analogous to the Reduce transformation.

#### GroupReduce on DataSet grouped by Field Position Keys (Tuple DataSets only)

***GOOD EXAMPLE REQUIRED HERE***

```java
// Some Reduce function definition **Good example required**
public class MyTupleReducer extends GroupReduceFunction<Tuple3<String, Integer, Double>, Double> {
  // [...]
}

// [...]
DataSet<Tuple3<String, Integer, Double>> input = // [...]
DataSet<Double> output = input
                         // group dataset by the second tuple field
                         .groupBy(1)
                         // apply GroupReduce function on each group
                         .reduceGroup(new MyTupleReducer());
```

#### GroupReduce on DataSet grouped by KeySelector Function

Works analogous to `KeySelector` functions in Reduce transformations.

#### GroupReduce on sorted groups (Tuple DataSets only)

Works analogous to group sortings in Reduce transformations.

#### Combinable GroupReduce Functions

In contrast to a `ReduceFunction`, a `GroupReduceFunction` is not implicitly combinable. In order to make a `GroupReduceFunction` combinable, you need to implement (override) the ```combine()``` method and annotate the `GroupReduceFunction` with the ```@Combinable``` annotation as shown here:

***GOOD EXAMPLE REQUIRED HERE***

```java
// Combinable GroupReduce function that
@Combinable
public class MyCombinableGroupReducer extends GroupReduceFunction<Integer, Integer> {
  @Override
  public void reduce(Iterator<Integer> in, Collector<Integer> out) throws Exception {
    // [...]
  }

  @Override
  public void combine(Iterator<Integer> in, Collector<Integer> out) throws Exception {
    // Good example required here
    // this.reduce(in, out);
  }
}
```

### Reduce on full DataSet

The Reduce transformation applies a user-defined `ReduceFunction` to all elements of a `DataSet`.</br>
The `ReduceFunction` subsequently combines pairs of elements into one element until only a single element remains.

The following example shows how to sum all elements in an Integer `DataSet`:

```java
// Reduce function that sums Integers
public class IntSummer extends ReduceFunction<Integer> {
  @Override
  public Integer reduce(Integer num1, Integer num2) throws Exception {
    return num1 + num2;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> sum = intNumbers.reduce(new IntSummer());
```

Reducing a full `DataSet` using the Reduce transformation implies that the final Reduce operation cannot be done in parallel. However, a `ReduceFunction` is automatically combinable such that a Reduce transformation does not limit scalability for most use cases.

### GroupReduce on full DataSet

The GroupReduce transformation applies a user-defined `GroupReduceFunction` on all elements of a `DataSet`.</br>
A `GroupReduceFunction` can iterate over all elements of `DataSet` and return an arbitrary number of result elements.

***GOOD EXAMPLE REQUIRED HERE***

```java
// GroupReduce function definition that ... (Good example required here)
public class ABC extends GroupReduceFunction<Integer, Double> {

  @Override
  public void reduce(Iterator<Integer> in, Collector<Double> out) throws Exception {
    // [...]
  }
}

// [...]
DataSet<Integer> input = // [...]
DataSet<Double> output = input.reduceGroup(new ABC());
```

### Aggregate

***TO BE DONE!***

Sum, Min, Max, Avg (note computations preserve type for AVG)

### Join

The Join transformation joins two `DataSet`s into one `DataSet`. Elements of `DataSet`s are joined on one or more keys which can be specified using 

- a `KeySelector` function or 
- one or more field position keys. 

There are a few different ways to perform a Join transformation which are shown in the following.

#### Join into Tuple2 (Default Join)

The default join transformation produces a new `DataSet` of `Tuple`s with two fields. Each tuple holds a joined element of the first input `DataSet` in the first tuple field and a matching element of the second input `DataSet` in the second field.
The following example shows a default join using field position keys:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Double, Integer>> input2 = // [...]
// result dataset is typed as Tuple2
DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Double, Integer>>> 
            result =
            input1.join(input2)
                  // key definition on first input using a field position key
                  .where(0)
                  // key definition of second input using a field position key
                  .equalTo(1);
```

#### Join with JoinFunction

A Join transformation can also call a user-defined `JoinFunction` to process joining tuples. </br>
A `JoinFunction` receives one element of the first input `DataSet` and one element of the second input `DataSet` and returns exactly one element.

The following example performs a join of `DataSet` with custom java objects and a `Tuple` `DataSet` using `KeySelector` functions and shows how to call a user-defined `JoinFunction`:

```java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointWeighter 
         extends JoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {
  
  @Override
  public Tuple2<String, Double> join(Rating rating, Tuple2<String, Double> weight) {
    // multiply the points and rating and construct a new output tuple
    return new Tuple2<String, Double>(rating.name, rating.points * weight.f1);
  }
}

DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Double>> weights = // [...]
DataSet<Tuple2<String, Double>> 
            weightedRatings =
            ratings.join(weights)
                   // key definition of first input using a KeySelector function
                   .where(new KeySelection<Rating, String>() { 
                            public String getKey(Rating r) { return r.category; } 
                          })
                   // key definition of second input using a KeySelector function
                   .equalTo(new KeySelection<Tuple2<String, Double>, String>() {
                              public String getKey(Tuple2<String, Double> t) { return t.f0; }
                            })
                   // applying the JoinFunction on joining pairs
                   .with(new PointWeighter());
```

#### Join with Projection (**NOT SUPPORTED YET**)

A Join transformation can construct result tuples using a projection as shown here:

```java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, String, Double, Byte>
            result =
            input1.join(input2)
                  // key definition on first input using a field position key
                  .where(0)
                  // key definition of second input using a field position key
                  .equalTo(0)
                  // select and reorder fields of matching tuples
                  .project(0,2,4,3).types(Integer.class, String.class, Double.class, Byte.class);
```

For the projection of join results, the field indicies address the fields of the first and the second tuple consecutively. In the above example, indicies 0 to 2 address the first, second, and third field of the first input and 3 and 4 address the first and second field of the second input, respectively. </br>
The join projection works also on non-`Tuple` `DataSet`s. In this case, the field indicies consider the non-`Tuple` elements as a `Tuple` with a single field.

#### Join with DataSet Size Hint

In order to guide the optimizer to pick the right execution strategy, you can hint the size of `DataSet`s to join as shown here:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>> 
            result1 =
            // hint that the second input is very small
            input1.joinWithTiny(input2)
                  .where(0)
                  .equalTo(0);

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>> 
            result2 =
            // hint that the second input is very large
            input1.joinWithHuge(input2)
                  .where(0)
                  .equalTo(0);
```

### Cross

The Cross transformation builds all element pair combinations of two `DataSet`s, i.e., it builds the Cartesian product.</br>
The transformation either calls a user-defined `CrossFunction` on each pair of elements or applies a projection. </br>
**Note:** Cross is potentially a *very* compute-intensive operation which can challenge even large compute clusters!

#### Cross with User-Defined Function

A Cross transformation can call a `CrossFunction`. A `CrossFunction` receives one element of the first input and one element of the second input and returns exactly one result element. </br>
Applying a Cross transformation on two `DataSet`s using a `CrossFunction` is done as follows:

```java
public class Coord {
  public int id;
  public int x;
  public int y;
}

// CrossFunction computes the distance between two Coord objects.
public class EuclidianDistComputer 
         extends CrossFunction<Coord, Coord, Tuple3<Integer, Integer, Double>> {
  
  @Override
  public Tuple3<Integer, Integer, Double> cross(Coord c1, Coord c2) {
    // compute Euclidian distance of coordinates
    double dist = Math.sqrt(Math.pow(c1.x - c2.x, 2) + Math.pow(c1.y - c2.y, 2));
    return new Tuple3<Integer, Integer, Double>(c1.id, c2.id, dist);
  }
}

DataSet<Coord> coords1 = // [...]
DataSet<Coord> coords2 = // [...]
DataSet<Tuple3<Integer, Integer, Double>> 
            distances =
            coords1.cross(coords2)
                   // apply CrossFunction
                   .with(new EuclidianDistComputer());
```

#### Cross with Projection (**NOT SUPPORTED YET**)

A Cross transformation can also construct result tuples using a projection as shown here:

```java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, Byte, Integer, Double>
            result =
            input1.cross(input2)
                  // select and reorder fields of matching tuples
                  .project(3,1,0,4).types(Integer.class, Byte.class, Integer.class, Double.class);
```

The field selection in a Cross projection works the same way as in the projection of Join results.

#### Cross with DataSet Size Hint

In order to guide the optimizer to pick the right execution strategy, you can hint the size of `DataSet`s to cross as shown here:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple4<Integer, String, Integer, String>>
            udfResult =
            // hint that the second input is very small
            input1.crossWithTiny(input2)
            // apply any Cross function (or projection)
                  .with(new MyCrosser());

DataSet<Tuple3<Integer, Integer, String>> 
            projectResult =
            // hint that the second input is very large
            input1.crossWithHuge(input2)
            // apply a projection (or any Cross function)
                  .project(0,2,1).types(Integer.class, Integer.class, String.class)
```

### CoGroup

The CoGroup transformation jointly processes groups of two `DataSet`s. Both `DataSet`s are grouped on a defined key and groups of both `DataSet`s with the same key are handed together to a user-defined `CoGroupFunction`. If only one `DataSet` has a group for a certain key, the `CoGroupFunction` is called with the existing group and an empty group.</br>
A `CoGroupFunction` can separately iterate over the elements of both groups and return an arbitrary number of result elements.

Similar to Reduce, GroupReduce, and Join, keys can be defined using

- a `KeySelector` function or 
- one or more field position keys.

#### CoGroup on DataSets grouped by Field Position Keys (Tuple DataSets only)

```java
// Some CoGroup function definition
public class MyCoGrouper
         extends CoGroupFunction<Tuple2<String, Integer>, Tuple2<String, Double>, Double> {
  @Override
  public void coGroup(Iterator<Tuple2<String, Integer>> iVals, 
                      Iterator<Tuple2<String, Double>> dVals, 
                      Collector<Double> out) {
    HashSet<Integer> ints = new HashSet<Integer>();
    // remove duplicate int values in group
    while(iVals.hasNext()) {
      ints.add(iVals.next().f1);
    }
    // multiply each double value with each unique int values of group
    while(dVals.hasNext()) {
      for(Integer i : ints) {
        out.collect(dVals.next().f1 * i));
      }
    }
  }
}

// [...]
DataSet<Tuple2<String, Integer>> iVals = // [...]
DataSet<Tuple2<String, Double>> dVals = // [...]
DataSet<Double> output = iVals.coGroup(dVals)
                         // group first input on first tuple field
                         .where(0)
                         // group second input on first tuple field
                         .equalTo(0)
                         // apply CoGroup function on each pair of groups
                         .reduceGroup(new MyCoGrouper());
```

#### CoGroup on DataSets grouped by Key Selector Function

Works analogous to key selector functions in Join transformations.

### Union (**NOT SUPPORTED YET**)

</section>

<section id="data_sources">
Data Sources
------------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="data_sinks">
Data Sinks
----------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="debugging">
Debugging
---------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="iterations">
Iteration Operators
-------------------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="annotations">
Annotations
-----------

Annotations allow the user to specify constant fields between input and output data of a user defined operator. The user can give these semantic hints by annotating the operator classes. A field is considered constant if its value is not modified and its position remains the same.

Currently, the usage of annotations is only possible with operators working on the `Tuple` classes as input and output types. Custom object support will be added in the future.

The following annotations are available:

* `@ConstantFields`: constant fields of a single input operator (like map).

* `@ConstantFieldsFirst`: constant fields for the first input of a two input operator (like join).

* `@ConstantFieldsSecond`: constant fields for the second input of a two input operator (like join).

* `@AllFieldsConstant`: all fields remain constant, no field is modified.

* `@ConstantFieldsExcept`: all fields of a single input operator except the given ones are constant.

* `@ConstantFieldsFirstExcept`: all fields of the first input of a two input operator except the given ones are constant.

* `@ConstantFieldsSecondExcept`: all fields of the second input of a two input operator except the given ones are constant.

**Note**: It is important to be strict when providing annotations. Only annotate fields, when you are certain that they are constant. If the behaviour of the operator is not clearly predictable, no annotation should be provided.

**Example**

The following example shows a simple map operator, which works on `Tuple2` types. It calculates the square of the first field. Since only field one is ever modified, all other fields are considered constant.

{% highlight java %}
@ConstantFields(fields={0})
public static final class Square extends
    FlatMapFunction<Tuple2<String, Integer>, Tuple2<String, Integer>> {

    @Override
    public void flatMap(Tuple2<String, Integer> value, Collector<Tuple2<String, Integer>> out) {
        Integer i = value.getField(1);
        value.setField(i*i, 1);

        out.collect(value);
    }
}
{% endhighlight %}

</section>

<section id="broadcast_variables">
Broadcast Variables
-------------------

<section id="broadcast_variables">

Broadcast Variables allow to broadcast computation results to all nodes executing an operator. The following example shows how to set a broadcast variable and how to access it within an operator.

{% highlight java %}
// in getPlan() method

FileDataSource someMainInput = new FileDataSource(...);

FileDataSource someBcInput = new FileDataSource(...);

MapOperator myMapper = MapOperator.builder(MyMapper.class)
    .setBroadcastVariable("my_bc_var", someBcInput) // set the variable
    .input(someMainInput)
    .build();

// [...]

public class MyMapper extends MapFunction {

    private Collection<Record> myBcRecords;

    @Override
    public void open(Configuration parameters) throws Exception {
        // receive the variables' content
        this.myBcRecords = this.getRuntimeContext().getBroadcastVariable("my_bc_var"); 
    }

    @Override
    public void map(Record record, Collector<Record> out) {       
       for (Record r : myBcRecords) {
           // do something with the records
       }

    }
}
{% endhighlight %}
*Note*: As the content of broadcast variables is kept in-memory on each node, it should not become too large. For simpler things like scalar values you should use `setParameter(...)`.

An example of how to use Broadcast Variables in practice can be found in the <a href="https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/kmeans/KMeans.java">K-Means example</a>.
</section>
</section>

<section id="packaging">
Program Packaging
-----------------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="accumulators_counters">
Accumulators &amp; Counters
---------------------------

Accumulators are simple constructs with an **add operation** and a **final accumulated result**, which is available after the job ended.

The most straightforward accumulator is a **counter**: You can increment it using the ```Accumulator.add(V value)``` method. At the end of the job Stratosphere will sum up (merge) all partial results and send the result to the client. Since accumulators are very easy to use, they can be useful during debugging or if you quickly want to find out more about your data.

Stratosphere currently has the following **built-in accumulators**. Each of them implements the [Accumulator](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Accumulator.java) interface.

- [__IntCounter__](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/IntCounter.java), [__LongCounter__](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/LongCounter.java) and [__DoubleCounter__](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/DoubleCounter.java): See below for an example using a counter.
- [__Histogram__](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Histogram.java): A histogram implementation for a discrete number of bins. Internally it is just a map from Integer to Integer. You can use this to compute distributions of values, e.g. the distribution of words-per-line for a word count program.

__How to use accumulators:__

First you have to create an accumulator object (here a counter) in the stub where you want to use it.

    private IntCounter numLines = new IntCounter();

Second you have to register the accumulator object, typically in the ```open()``` method of the stub. Here you also define the name.

    getRuntimeContext().addAccumulator("num-lines", this.numLines);

You can now use the accumulator anywhere in the stub, including in the ```open()``` and ```close()``` methods.

    this.numLines.add(1);

The overall result will be stored in the ```JobExecutionResult``` object which is returned when running a job using the Java API (currently this only works if the execution waits for the completion of the job).

    myJobExecutionResult.getAccumulatorResult("num-lines")

All accumulators share a single namespace per job. Thus you can use the same accumulator in different stubs of your job. Stratosphere will internally merge all accumulators with the same name.

Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for a complete example.

A note on accumulators and iterations: Currently the result of accumulators is only available after the overall job ended. We plan to also make the result of the previous iteration available in the next iteration.

__Custom accumulators:__

To implement your own accumulator you simply have to write your implementation of the Accumulator interface. Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for an example. Feel free to create a pull request if you think your custom accumulator should be shipped with Stratosphere.

You have the choice to implement either [Accumulator](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Accumulator.java) or [SimpleAccumulator](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/SimpleAccumulator.java). ```Accumulator<V,R>``` is most flexible: It defines a type ```V``` for the value to add, and a result type ```R``` for the final result. E.g. for a histogram, ```V``` is a number and ```R``` is a histogram. ```SimpleAccumulator``` is for the cases where both types are the same, e.g. for counters.
</section>

<section id="execution_plan">
Execution Plans
---------------

Depending on various parameters such as data size or number of machines in the cluster, Stratosphere's optimizer automatically chooses an execution strategy for your program. In many cases, it can be useful to know how exactly Stratosphere will execute your program.

__Plan Visualization Tool__

Stratosphere 0.5 comes packaged with a visualization tool for execution plans. The HTML document containing the visualizer is located under ```tools/planVisualizer.html```. It takes a JSON representation of the job execution plan and visualizes it as a graph with complete annotations of execution strategies.

The following code shows how to print the execution plan JSON from your program:

    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
    
    ...
    
    System.out.println(env.getExecutionPlan());


To visualize the execution plan, do the following:

1. **Open** ```planVisualizer.html``` with your web browser,
2. **Paste** the JSON string into the text field, and
3. **Press** the draw button.

After these steps, a detailed execution plan will be visualized.

![alt text](http://stratosphere.eu/img/blog/plan_visualizer2.png "A stratosphere job execution graph.")

__Web Interface__

Stratosphere offers a web interface for submitting and executing jobs. If you choose to use this interface to submit your packaged program, you have the option to also see the plan visualization.

The script to start the webinterface is located under ```bin/start-webclient.sh```. After starting the webclient (per default on **port 8080**), your program can be uploaded and will be added to the list of available programs on the left side of the interface.

You are able to specify program arguments in the textbox at the bottom of the page. Checking the plan visualization checkbox shows the execution plan before executing the actual program.

</section>