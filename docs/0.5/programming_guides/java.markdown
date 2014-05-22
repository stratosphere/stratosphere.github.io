---
layout: inner_docs_v05
title: Java API
toc:
#  - {anchor: "introduction", title: "Introduction"}
  - {anchor: "example", title: "Example Program"}
  - {anchor: "linking", title: "Linking"}
  - {anchor: "skeleton", title: "Program Skeleton"}
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

Analysis programs in Stratosphere are regular Java Programs that implement transformations on data sets (e.g., filtering, , mapping, joining, grouping). The data sets are initially created from certain sources (e.g., by reading files, or from collections). Results are written to sinks, which may for example write the data to (distributed) files, or to standard output (for example your terminal).

Stratosphere programs can run in a variety of contexts, for example locally as standalone programs, locally embedded in other programs, or on clusters of many machines (see [program skeleton](#skeleton) for how to define different environments). All programs are executed lazily: When the program is run and transformation methods are invoked on a data set, it creates a transformation operation. That transformation operation is only executed once program execution is triggered on the environment. Whether the program is executed locally or on a cluster depends on the environment of the program.

The Java API is strongly typed: All data sets and transformations accept typed elements. This allows to catch typing errors very early and supports safe refactoring of programs.

The sections on the [program skeleton](#skeleton) and [transformations](#transformations) show the general template of a program and describe the available transformations.

<div class="panel panel-default">
  <div class="panel-body">
    <strong>
    While most parts of the new Java API are already working, we are still in the process of stabilizing it. If you encounter any problems, feel free to <a href="https://github.com/stratosphere/stratosphere/issues">post an issue on GitHub</a> or <a href="https://groups.google.com/forum/#!forum/stratosphere-dev">write to our mailing list</a>. You can also check out our stable <a href="{{ site.baseurl }}/docs/0.4/programming_guides/java.html">Java Record API</a>, which is the default API of all previous versions.
    </strong>
  </div>
</div>
</section>

<section id="toc">

<div id="docs_05_toc">
  <div class="list-group">
{% for sublink in page.toc %}
   <a href="#{{ sublink.anchor }}" class="list-group-item">{{forloop.index}}. <strong>{{ sublink.title }}</strong></a>
{% endfor %}
  </div>
</div>

</section>

<section id="example">
Example Program
---------------

The following program is a complete, working example of WordCount. You can copy &amp; paste the code to run it locally. You only have to include Stratosphere's Java API library into your project (see Section [Linking with Stratosphere](#linking)) and specify the imports. Then you are ready to go!

```java
public class WordCountExample {
    public static void main(String[] args) throws Exception {
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        DataSet<String> text = env.fromElements(
            "Who's there?",
            "I think I hear them. Stand, ho! Who's there?");

        DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .aggregate(Aggregations.SUM, 1);

        wordCounts.print();

        env.execute("Word Count Example");
    }

    public static final class LineSplitter extends FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String line, Collector<Tuple2<String, Integer>> out) {
            for (String word : line.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}
```

<div class="back-to-top"><a href="#toc">Back to top</a></div>
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

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="skeleton">
Program Skeleton
----------------

As we already saw in the example, Stratosphere programs look like regular Java
programs with a `main()` method. Each program consists of the same basic parts:

1. Obtain an `ExecutionEnvironment`,
2. Load your data,
3. Specify transformations on this data,
4. Specify where to store the results of your computations, and
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
you created a JAR file from you program, the Stratosphere cluster manager will
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

DataSet<Integer> tokenized = text.map(new MapFunction<String, Integer>() {
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
<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="types">
Data Types
----------

Stratosphere's Java API allows the use of different data types for the input and output of operators.
Both `DataSet` and functions like `MapFunction`, `ReduceFunction`, etc. are parameterized with data types using Java generics in order to ensure type-safety.

There are four different categories of data types:

1. **Basic Java Types**
2. **Tuples**
3. **Custom Types**
4. **Values**

#### Basic Java Types

The API supports all common basic Java types:

- `Short`, `Integer`, `Long`
- `Float`, `Double`
- `Byte`, `Boolean`
- `Character`, `String`

You can use all of them to parameterize both `DataSet` and function implementations, e.g. `DataSet<String>` for a `String` data set or `MapFunction<String, Integer>` for a mapper from `String` to `Integer`.

{% highlight java %}
DataSet<String> numbers = env.fromElements("1", "2");

numbers.map(new MapFunction<String, Integer>() {
    @Override
    public String map(String value) throws Exception {
        return Integer.parseInt(value);
    }
});
{% endhighlight %}

#### Tuples

You can use the `Tuple` classes for ordered list of elements. The Java API provides classes from `Tuple1` up to `Tuple22`. Every field of a tuple can be an arbitrary Stratosphere type; including further tuples (nested tuples).

```java
DataSet<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public String map(Tuple2<String, Integer> value) throws Exception {
        return value.getField(1);
    }
});
```

Fields of a Tuple can be accessed directly by using `tuple.f4` or `tuple.getField(4)`. The field numbering starts with 0. In order to access fields
more intuitively and generate more readable code, it is also possible to extend a subclass of `Tuple` and add getters and setters with custom names.

#### Custom Types

You can use your custom Java classes as Stratosphere types, if they are `Serializable`.
Consider this simple class:

```java
public static class WordCount implements Serializable {
    public String word;
    public int count;

    public WordCount() {}

    public WordCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}
```

You can now create a `DataSet<WordCount>` and run transformations, which operate on your custom type.

```java
DataSet<WordCount> wordCounts = env.fromElements(
    new WordCount("hello", 1),
    new WordCount("world", 2));

wordCounts.map(new MapFunction<WordCount, Integer>() {
    @Override
    public String map(WordCount value) throws Exception {
        return value.count;
    }
});
```

When working with operators that require a Key for grouping or matching records
you need to implement a `KeySelector` for your custom type. This is different
from tuples where you can simply specify grouping fields by indices. (see
[Section Data Transformations](#transformations))

```java
wordCounts.groupBy(new KeySelector<WordCount, String>() {
    public String getKey(WordCount v) {
        return v.word;
    }
}).reduce(new MyReduceFunction());
```

#### Values

Stratosphere also provides the `Value` interface with two methods: `read` and `write`. Using this you can implement a
data type with custom serialization and deserialization code.

Stratosphere also provides serializable wrapper types around Java basic types and Collections implementing the Java `List` or `Map` interfaces:

- `ShortValue`, `IntValue`, `LongValue`
- `FloatValue`, `DoubleValue`
- `ByteValue`, `BooleanValue`
- `CharValue`, `StringValue`
- `ListValue`, `MapValue`


<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="transformations">
Data Transformations
--------------------

A data transformation transforms one or more `DataSet`s into a new `DataSet`. Advanced data analysis programs can be assembled by chaining multiple transformations.

### Map

The Map transformation applies a user-defined `MapFunction` on each element of a DataSet.
It implements a one-to-one mapping, that is, exactly one element must be returned by
the function.

The following code transforms a `DataSet` of Integer pairs into a `DataSet` of Integers:

```java
// MapFunction that adds two integer values
public class IntAdder extends MapFunction<Tuple2<Integer, Integer>, Integer> {
  @Override
  public Integer map(Tuple2<Integer, Integer> in) {
    return in.f0 + in.f1;
  }
}

// [...]
DataSet<Tuple2<Integer, Integer>> intPairs = // [...]
DataSet<Integer> intSums = intPairs.map(new IntAdder());
```

### FlatMap

The FlatMap transformation applies a user-defined `FlatMapFunction` on each element of a `DataSet`.
This variant of a map function can return arbitrary many result elements (including none) for each input element.

The following code transforms a `DataSet` of text lines into a `DataSet` of words:

```java
// FlatMapFunction that tokenizes a String by whitespace characters and emits all String tokens.
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

The Filter transformation applies a user-defined `FilterFunction` on each element of a `DataSet` and retains only those elements for which the function returns `true`.

The following code removes all Integers smaller than zero from a `DataSet`:

```java
// FilterFunction that filters out all Integers smaller than zero.
public class NaturalNumberFilter extends FilterFunction<Integer> {
  @Override
  public boolean filter(Integer number) {
    return number >= 0;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> naturalNumbers = intNumbers.filter(new NaturalNumberFilter());
```

### Project (Tuple DataSets only)

The Project transformation removes or moves `Tuple` fields of a `Tuple` `DataSet`.
The `project(int...)` method selects `Tuple` fields that should be retained by their index and defines their order in the output `Tuple`.
The `types(Class<?> ...)`method must give the types of the output `Tuple` fields.

Projections do not require the definition of a user function.

The following code shows different ways to apply a Project transformation on a `DataSet`:

```java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// converts Tuple3<Integer, Double, String> into Tuple2<String, Integer>
DataSet<Tuple2<String, Integer>> out = in.project(2,0).types(String.class, Integer.class);
```

### Transformations on grouped DataSet

The reduce operations can operate on grouped data sets. Specifying the key to
be used for grouping can be done in two ways:

- a `KeySelector` function or
- one or more field position keys (`Tuple` `DataSet` only).

Please look at the reduce examples to see how the grouping keys are specified.

### Reduce on grouped DataSet

A Reduce transformation that is applied on a grouped `DataSet` reduces each group to a single element using a user-defined `ReduceFunction`.
For each group of input elements, a `ReduceFunction` successively combines pairs of elements into one element until only a single element for each group remains.

#### Reduce on DataSet grouped by KeySelector Function

A `KeySelector` function extracts a key value from each element of a `DataSet`. The extracted key value is used to group the `DataSet`.
The following code shows how to group a POJO `DataSet` using a `KeySelector` function and to reduce it with a `ReduceFunction`.

```java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter extends ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping with inline-defined KeySelector function
                         .groupBy(
                           new KeySelector<WC, String>() {
                             public String getKey(WC wc) { return wc.word; }
                           })
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());
```

#### Reduce on DataSet grouped by Field Position Keys (Tuple DataSets only)

Field position keys specify one or more fields of a `Tuple` `DataSet` that are used as grouping keys.
The following code shows how to use field position keys and apply a `ReduceFunction`.

```java
DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples =
                                         tuples
                                         // group DataSet on first and second field of Tuple
                                         .groupBy(0,1)
                                         // apply ReduceFunction on grouped DataSet
                                         .reduce(new MyTupleReducer());
```

### GroupReduce on grouped DataSet

A GroupReduce transformation that is applied on a grouped `DataSet` calls a user-defined `GroupReduceFunction` for each group. The difference
between this and `Reduce` is that the user defined function gets the whole group at once.
The function is invoked with an iterator over all elements of a group and can return an arbitrary number of result elements using the collector.

#### GroupReduce on DataSet grouped by Field Position Keys (Tuple DataSets only)

The following code shows how duplicate strings can be removed from a `DataSet` grouped by Integer.

```java
public class DistinctReduce
         extends GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String> {
  // Set to hold all unique strings of a group
  Set<String> uniqStrings = new HashSet<String>();

  @Override
  public void reduce(Iterator<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {
    // clear set
    uniqStrings.clear();
    // there is at least one element in the iterator
    Tuple2<Integer, String> first = in.next();
    Integer key = first.f0;
    uniqStrings.add(first.f1);
    // add all strings of the group to the set
    while(in.hasNext()) {
      uniqStrings.add(in.next().f1);
    }
    // emit all unique strings
    Tuple2<Integer, String> t = new Tuple2<Integer, String>(key, "");
    for(String s : uniqStrings) {
      t.f1 = s;
      out.collect(t);
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Tuple2<Integer, String>> output =
                                 input
                                 // group DataSet by the first tuple field
                                 .groupBy(0)
                                 // apply GroupReduceFunction on each group and
                                 //   remove elements with duplicate strings.
                                 .reduceGroup(new DistinctReduce());
```

**Note:** Stratosphere internally works a lot with mutable objects. Collecting objects like in the above example only works because Strings are immutable in Java!

#### GroupReduce on DataSet grouped by KeySelector Function

Works analogous to `KeySelector` functions in Reduce transformations.

#### GroupReduce on sorted groups (Tuple DataSets only)

A `GroupReduceFunction` accesses the elements of a group using an iterator. Optionally, the iterator can hand out the elements of a group in a specified order. In many cases this can help to reduce the complexity of a user-defined `GroupReduceFunction` and improve its efficiency.
Right now, this feature is only available for `Tuple` `DataSet`.

The following code shows another example how to remove duplicate Strings in a `DataSet` grouped by an Integer and sorted by String.

```java
// GroupReduceFunction that removes consecutive identical elements
public class DistinctReduce
         extends GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {
  @Override
  public void reduce(Iterator<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {
    // there is at least one element in the iterator
    Tuple2<Integer, String> first = in.next();
    Integer key = first.f0;
    String comp = first.f1;
    // for each element in group
    while(in.hasNext()) {
      String next = in.next().f1;
      // check if strings are different
      if(!next.equals(comp)) {
        // emit a new element
        out.collect(new Tuple2<Integer, String>(key, comp));
        // update compare string
        comp = next;
      }
    }
    // emit last element
    out.collect(new Tuple2<Integer, String>(key, comp));
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Double> output = input
                         // group DataSet by the first tuple field
                         .groupBy(0)
                         // sort groups on second tuple field
                         .sortGroup(1, Order.ASCENDING)
                         // // apply GroupReduceFunction on DataSet with sorted groups
                         .reduceGroup(new DistinctReduce());
```

**Note:** A GroupSort often comes for free if the grouping is established using a sort-based execution strategy of an operator before the reduce operation.

#### Combinable GroupReduceFunctions

In contrast to a `ReduceFunction`, a `GroupReduceFunction` is not implicitly combinable. In order to make a `GroupReduceFunction` combinable, you need to implement (override) the ```combine()``` method and annotate the `GroupReduceFunction` with the ```@Combinable``` annotation as shown here:

The following code shows how to compute multiple sums using a combinable `GroupReduceFunction`:

```java
// Combinable GroupReduceFunction that computes two sums.
@Combinable
public class MyCombinableGroupReducer
         extends GroupReduceFunction<Tuple3<String, Integer, Double>,
                                     Tuple3<String, Integer, Double>> {
  @Override
  public void reduce(Iterator<Tuple3<String, Integer, Double>> in,
                     Collector<Tuple3<String, Integer, Double>> out) {
    // one element is always present in iterator
    Tuple3<String, Integer, Double> curr = in.next();
    String key = curr.f0;
    int intSum = curr.f1;
    double doubleSum = curr.f2;
    // sum up all ints and doubles
    while(in.hasNext()) {
      curr = in.next();
      intSum += curr.f1;
      doubleSum += curr.f2;
    }
    // emit a tuple with both sums
    out.collect(new Tuple3<String, Integer, Double>(key, intSum, doubleSum));
  }

  @Override
  public void combine(Iterator<Tuple3<String, Integer, Double>> in,
                      Collector<Tuple3<String, Integer, Double>> out)) {
    // in some cases combine() calls can simply be forwarded to reduce().
    this.reduce(in, out);
  }
}
```

### Aggregate on grouped Tuple DataSet

There are some common aggregation operations that are frequently used. The Aggregate transformation provides the following build-in aggregation functions:

- Sum,
- Min,
- Max, and
- Average.

The Aggregate transformation can only be applied on a `Tuple` `DataSet` and supports only field positions keys for grouping.

The following code shows how to apply an Aggregation transformation on a `DataSet` grouped by field position keys:

```java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                          // group DataSet on second field
                                          .groupBy(1)
                                          // compute sum of the first field
                                          .aggregate(SUM, 0)
                                          // compute average of the third field
                                          .and(AVG, 2);
```

To apply multiple aggregations on a DataSet it is necessary to use the `.and()` function after the first aggregate, that means `.aggregate(SUM, 0).and(AVG, 2)` produces the sum of field 0 and the average of field 2 of the original DataSet. 
In contrast to that `.aggregate(SUM, 0).aggregate(AVG, 2)` will apply an aggregation on an aggregation. In the given example it would produce the average of field 2 after calculating the sum of field 0 grouped by field 1.

**Note:** Right now, aggregation functions are type preserving. This means that for example computing the average of Integer values will yield an Integer value, i.e., the result is rounded.
The set of aggregation functions will be extended in the future.

### Reduce on full DataSet

The Reduce transformation applies a user-defined `ReduceFunction` to all elements of a `DataSet`.
The `ReduceFunction` subsequently combines pairs of elements into one element until only a single element remains.

The following code shows how to sum all elements of an Integer `DataSet`:

```java
// ReduceFunction that sums Integers
public class IntSummer extends ReduceFunction<Integer> {
  @Override
  public Integer reduce(Integer num1, Integer num2) {
    return num1 + num2;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> sum = intNumbers.reduce(new IntSummer());
```

Reducing a full `DataSet` using the Reduce transformation implies that the final Reduce operation cannot be done in parallel. However, a `ReduceFunction` is automatically combinable such that a Reduce transformation does not limit scalability for most use cases.

### GroupReduce on full DataSet

The GroupReduce transformation applies a user-defined `GroupReduceFunction` on all elements of a `DataSet`.
A `GroupReduceFunction` can iterate over all elements of `DataSet` and return an arbitrary number of result elements.

The following example shows how to apply a GroupReduce transformation on a full `DataSet`:

```java
DataSet<Integer> input = // [...]
// apply a (preferably combinable) GroupReduceFunction to a DataSet
DataSet<Double> output = input.reduceGroup(new MyGroupReducer());
```

**Note:** A GroupReduce transformation on a full `DataSet` cannot be done in parallel if the `GroupReduceFunction` is not combinable. Therefore, this can be a very compute intensive operation. See the paragraph on "Combineable `GroupReduceFunction`s" above to learn how to implement a combinable `GroupReduceFunction`.

### Aggregate on full Tuple DataSet

There are some common aggregation operations that are frequently used. The Aggregate transformation provides the following build-in aggregation functions:

- Sum,
- Min,
- Max, and
- Average.

The Aggregate transformation can only be applied on a `Tuple` `DataSet`.

The following code shows how to apply an Aggregation transformation on a full `DataSet`:

```java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input
                                          // compute sum of the first field
                                          .aggregate(SUM, 0)
                                          // compute average of the second field
                                          .and(AVG, 1);
```

**Note:** Right now, aggregation functions are type preserving. This means that for example computing the average of Integer values will yield an Integer value, i.e., the result is rounded. In the current implementation, aggregation functions are not combinable. Therefore, aggregating a non-grouped Dataset can have severe performance implications. Improving the performance of built-in aggregation functions as well as extending the set of supported aggregation functions is on our roadmap.

### Join

The Join transformation joins two `DataSet`s into one `DataSet`. The elements of both `DataSet`s are joined on one or more keys which can be specified using

- a `KeySelector` function or
- one or more field position keys (`Tuple` `DataSet` only).

There are a few different ways to perform a Join transformation which are shown in the following.

#### Default Join (Join into Tuple2)

The default Join transformation produces a new `Tuple``DataSet` with two fields. Each tuple holds a joined element of the first input `DataSet` in the first tuple field and a matching element of the second input `DataSet` in the second field.

The following code shows a default Join transformation using field position keys:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Double, Integer>> input2 = // [...]
// result dataset is typed as Tuple2
DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Double, Integer>>>
            result =
            input1.join(input2)
                  // key definition on first DataSet using a field position key
                  .where(0)
                  // key definition of second DataSet using a field position key
                  .equalTo(1);
```

#### Join with JoinFunction

A Join transformation can also call a user-defined `JoinFunction` to process joining tuples.
A `JoinFunction` receives one element of the first input `DataSet` and one element of the second input `DataSet` and returns exactly one element.

The following code performs a join of `DataSet` with custom java objects and a `Tuple` `DataSet` using `KeySelector` functions and shows how to call a user-defined `JoinFunction`:

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
                   // key definition of first DataSet using a KeySelector function
                   .where(new KeySelection<Rating, String>() {
                            public String getKey(Rating r) { return r.category; }
                          })
                   // key definition of second DataSet using a KeySelector function
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
                  // key definition on first DataSet using a field position key
                  .where(0)
                  // key definition of second DataSet using a field position key
                  .equalTo(0)
                  // select and reorder fields of matching tuples
                  .projectFirst(0,2).projectSecond(1).projectFirst(1)
                  .types(Integer.class, String.class, Double.class, Byte.class);
```

`projectFirst(int...)` and `projectSecond(int...)` select the fields of the first and second joined input that should be assembled into an output `Tuple`. The order of indexes defines the order of fields in the output tuple.
The join projection works also for non-`Tuple` `DataSet`s. In this case, `projectFirst()` or `projectSecond()` must be called without arguments to add a joined element ot the output `Tuple`.

#### Join with DataSet Size Hint

In order to guide the optimizer to pick the right execution strategy, you can hint the size of a `DataSet` to join as shown here:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result1 =
            // hint that the second DataSet is very small
            input1.joinWithTiny(input2)
                  .where(0)
                  .equalTo(0);

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result2 =
            // hint that the second DataSet is very large
            input1.joinWithHuge(input2)
                  .where(0)
                  .equalTo(0);
```

### Cross

The Cross transformation combines two `DataSet`s into one `DataSet`. It builds all pairwise combinations of the elements of both input `DataSet`s, i.e., it builds a Cartesian product.
The Cross transformation either calls a user-defined `CrossFunction` on each pair of elements or applies a projection. Both modes are shown in the following.

**Note:** Cross is potentially a *very* compute-intensive operation which can challenge even large compute clusters!

#### Cross with User-Defined Function

A Cross transformation can call a user-defined `CrossFunction`. A `CrossFunction` receives one element of the first input and one element of the second input and returns exactly one result element.

The following code shows how to apply a Cross transformation on two `DataSet`s using a `CrossFunction`:

```java
public class Coord {
  public int id;
  public int x;
  public int y;
}

// CrossFunction computes the Euclidean distance between two Coord objects.
public class EuclideanDistComputer
         extends CrossFunction<Coord, Coord, Tuple3<Integer, Integer, Double>> {

  @Override
  public Tuple3<Integer, Integer, Double> cross(Coord c1, Coord c2) {
    // compute Euclidean distance of coordinates
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
                   .with(new EuclideanDistComputer());
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
                  .projectSecond(0).projectFirst(1,0).projectSecond(1)
                  .types(Integer.class, Byte.class, Integer.class, Double.class);
```

The field selection in a Cross projection works the same way as in the projection of Join results.

#### Cross with DataSet Size Hint

In order to guide the optimizer to pick the right execution strategy, you can hint the size of a `DataSet` to cross as shown here:

```java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple4<Integer, String, Integer, String>>
            udfResult =
                  // hint that the second DataSet is very small
            input1.crossWithTiny(input2)
                  // apply any Cross function (or projection)
                  .with(new MyCrosser());

DataSet<Tuple3<Integer, Integer, String>>
            projectResult =
                  // hint that the second DataSet is very large
            input1.crossWithHuge(input2)
                  // apply a projection (or any Cross function)
                  .projectFirst(0,1).projectSecond(1).types(Integer.class, String.class, String.class)
```

### CoGroup

The CoGroup transformation jointly processes groups of two `DataSet`s. Both `DataSet`s are grouped on a defined key and groups of both `DataSet`s that share the same key are handed together to a user-defined `CoGroupFunction`. If for a specific key only one `DataSet` has a group, the `CoGroupFunction` is called with this group and an empty group.
A `CoGroupFunction` can separately iterate over the elements of both groups and return an arbitrary number of result elements.

Similar to Reduce, GroupReduce, and Join, keys can be defined using

- a `KeySelector` function or
- one or more field position keys (`Tuple` `DataSet` only).

#### CoGroup on DataSets grouped by Field Position Keys (Tuple DataSets only)

```java
// Some CoGroupFunction definition
public class MyCoGrouper
         extends CoGroupFunction<Tuple2<String, Integer>, Tuple2<String, Double>, Double> {
  // set to hold unique Integer values
  Set<Integer> ints = new HashSet<Integer>();

  @Override
  public void coGroup(Iterator<Tuple2<String, Integer>> iVals,
                      Iterator<Tuple2<String, Double>> dVals,
                      Collector<Double> out) {
    // clear Integer set
    ints.clear();
    // add all Integer values in group to set
    while(iVals.hasNext()) {
      ints.add(iVals.next().f1);
    }
    // multiply each Double value with each unique Integer values of group
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
                         // group first DataSet on first tuple field
                         .where(0)
                         // group second DataSet on first tuple field
                         .equalTo(0)
                         // apply CoGroup function on each pair of groups
                         .reduceGroup(new MyCoGrouper());
```

#### CoGroup on DataSets grouped by Key Selector Function

Works analogous to key selector functions in Join transformations.

### Union

Produces the union of two `DataSet`s, which have to be of the same type. A union of more than two `DataSet`s can be implemented with multiple union calls, as shown here:

```java
DataSet<Tuple2<String, Integer>> vals1 = // [...]
DataSet<Tuple2<String, Integer>> vals2 = // [...]
DataSet<Tuple2<String, Integer>> vals3 = // [...]
DataSet<Tuple2<String, Integer>> unioned = vals1.union(vals2)
                    .union(vals3);
```


<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="data_sources">
Data Sources
------------

DataSets are created by Data Sources. Basically, a data source generates a sequence of data items which can be processed by subsequent transformations. The data which is turned into data items by a data source can originate from any external sources. It can be read from a file, queried from a database or key-value store, or even be generated on the fly.

In the following we show how to create DataSets using some common data sources as well as generic InputFormats.

### Read Data from text files

The following examples show some common ways to read text files:

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read text file from local files system
DataSet<String> localLines = env.readTextFile("file:///path/to/my/textfile");

// read text file from a HDFS running at nnHost:nnPort
DataSet<String> hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile");

// read 1st (int), 3rd (double), and 6th (String) fields of a CSV file
DataSet<Tuple3<Integer, Double, String>> csvData = 
    env.readCsvFile("file:///path/to/my/csvfile")
    // set field and line delimiters
    .fieldDelimiter('|').lineDelimiter("\n")
    // select fields to include
    .includeFields("10100100")
    // give types of fields
    .types(Integer.class, Double.class, String.class);
```

### Read Data using generic InputFormats

Stratosphere uses the abstraction of InputFormats to read data from any data store (very similar to Hadoop MR). An InputFormat defines how a data source can be divided into individual splits which can be read in parallel, how to read the data of a source, and finally how to generate data items from it. This is a very generic way of handling input data. You can implement your own InputFormat to read any data and generate data items from it. Stratosphere provides a couple of abstract base classes to ease the implementation of custom file-based InputFormats.

The following examples show how to create DataSets using some InputFormats which are provided by Stratosphere:

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Read text lines using a TextInputFormat
DataSet<String> textLines = 
    env.createInput(
      // create and configure input format
      new TextInputFormat(new Path("file:///path/to/my/textfile")), 
      // specify type information for DataSet
      BasicTypeInfo.STRING_TYPE_INFO
    );

// Read data from a relational database using the JDBC input format
DataSet<Tuple2<String, Integer> dbData = 
    env.createInput(
      // create and configure input format
      JDBCInputFormat.buildJDBCInputFormat()
                     .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                     .setDBUrl("jdbc:derby:memory:persons")
                     .setQuery("select name, age from persons")
                     .finish(),
      // specify type information for DataSet
      new TupleTypeInfo(Tuple2.class, STRING_TYPE_INFO, INT_TYPE_INFO)
    );

```
**Note:** Stratosphere's program compiler needs to infer the data types of the data items which are returned by an InputFormat. If this information cannot be automatically inferred, it is necessary to manually provide the type information as shown in the examples above.

### Read Data from Java Collections

Stratosphere also provides a data source to read from Java collections. This feature is especially beneficial when developing and debugging data analysis programs. Please find details about collection data sources in the [Debugging](#debugging) section.

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="data_sinks">
Data Sinks
----------

Data Sinks are used to emit DataSets which are produced by a Stratosphere program. Stratosphere provides different build-in DataSinks but also supports the generic abstraction of OutputFormats (very similar to Hadoop MR). Implement your own OutputFormat to write the result of a Stratosphere program in any format to any location you like.

In the following we show some examples how to emit data from a Stratosphere program.

### Print DataSet to Standard-Out

```java
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write DataSet to std-out using the toString() method of the data type
myResult.print();

// write DataSet to std-err using the toString() method of the data type
myResult.printToErr();
```

### Write DataSet to a text file

```java
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write DataSet to a file on the local file system
myResult.writeAsText("file:///my/result/on/localFS");

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
myResult.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS");

// write DataSet to a file and overwrite the file if it exists
myResult.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE);

// write DataSet to a CSV file with new-line ('\n') as record and blank (' ') as field separators
myResult.writeAsCsv("file:///my/result/on/localFS", "\n", " ");
```
**NOTE:** Only Tuple DataSets can be written to CSV files.


### Write DataSet using a generic OutputFormat

```
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write Tuple DataSet to a relational database
myResult.output(
    // build and configure OutputFormat
    JDBCOutputFormat.buildJDBCOutputFormat()
                    .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                    .setDBUrl("jdbc:derby:memory:persons")
                    .setQuery("insert into persons (name, age, height) values (?,?,?)")
                    .finish()
    );

```

### Insert DataSet into Java Collection

Stratosphere also provides a data sink to insert data to a Java collection. This feature is especially beneficial for developing, debugging, and testing of data analysis programs. Please find details about collection data sinks in the following [Debugging](#debugging) section.

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="debugging">
Debugging
---------

Before running a data analysis program on a large data set in a distributed cluster, it is a good idea to make sure that the implemented algorithm works as desired. Hence, implementing data analysis programs is usually an incremental process of checking results, debugging, and improving. 

<p>
Stratosphere provides a few nice features to significantly ease the development process of data analysis programs by supporting local debugging from within an IDE, injection of test data, and collection of result data. This section give some hints how to ease the development of Stratosphere programs.
</p>

### Local Execution Environment

A `LocalEnvironment` starts a Stratosphere system within the same JVM process it was created in. If you start the LocalEnvironement from an IDE, you can set breakpoint in your code and easily debug your program. 

<p>
A LocalEnvironment is created and used as follows:
</p>

```java
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

DataSet<String> lines = env.readTextFile(pathToTextFile);
// build your program

env.execute();

```

### Collection Data Sources and Sinks

Providing input for an analysis program and checking its output is cumbersome done by creating input files and reading output files. Stratosphere features special data sources and sinks which are backed by Java collections to ease testing. Once a program has been tested, the sources and sinks can be easily replaced by sources and sinks that read from / write to external data stores such as HDFS.

<p>
Collection data sources can be used as follows:
</p>

```java
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

// Create a DataSet from a list of elements
DataSet<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataSet from any Java collection
List<Tuple2<String, Integer>> data = ...
DataSet<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataSet from an Iterator
Iterator<Long> longIt = ...
DataSet<Long> myLongs = env.fromCollection(longIt, Long.class);
```

**Note:** Currently, the collection data source requires that data types and iterators implement `Serializable`. Furthermore, collection data sources can not be executed in parallel (degree of parallelism = 1).

<p>
A collection data sink is specified as follows:
</p>

```java
DataSet<Tuple2<String, Integer>> myResult = ...

List<Tuple2<String, Integer>> outData = new ArrayList<Tuple2<String, Integer>>();
myResult.output(new LocalCollectionOutputFormat(outData));
```

**Note:** Collection data sources will only work correctly, if the whole program is executed in the same JVM!

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="iterations">
Iteration Operators
-------------------

Iterations implement loops in Stratosphere programs. The iteration operators encapsulate a part of the program and execute it repeatedly, feeding back the result of one iteration (the partial solution) into the next iteration. There are two types of iterations in Stratosphere: **BulkIteration** and **DeltaIteration**.

This section provides quick examples on how to use both operators. Check out the [Introduction to Iterations]({{site.baseurl}}/docs/0.5/programming_guides/iterations.html) page for a more detailed introduction.

#### Bulk Iterations

To create a BulkIteration call the `iterate(int)` method of the `DataSet` the iteration should start at. This will return an `IterativeDataSet`, which can be transformed with the regular operators. The single argument to the iterate call specifies the maximum number of iterations.

To specify the end of an iteration call the `closeWith(DataSet)` method on the `IterativeDataSet` to specify which transformation should be fed back to the next iteration. You can optionally specify a termination criterion with `closeWith(DataSet, DataSet)`, which evaluates the second DataSet and terminates the iteration, if this DataSet is empty. If no termination criterion is specified, the iteration terminates after the given maximum number iterations.

The following example iteratively estimates the number Pi. The goal is to count the number of random points, which fall into the unit circle. In each iteration, a random point is picked. If this point lies inside the unit circle, we increment the count. Pi is then estimated as the resulting count divided by the number of iterations multiplied by 4.

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Create initial IterativeDataSet
IterativeDataSet<Integer> initial = env.fromElements(0).iterate(10000);

DataSet<Integer> iteration = initial.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer i) throws Exception {
        double x = Math.random();
        double y = Math.random();

        return i + ((x * x + y * y < 1) ? 1 : 0);
    }
});

// Iteratively transform the IterativeDataSet
DataSet<Integer> count = initial.closeWith(iteration);

count.map(new MapFunction<Integer, Double>() {
    @Override
    public Double map(Integer count) throws Exception {
        return count / (double) 10000 * 4;
    }
}).print();

env.execute("Iterative Pi Example");
{% endhighlight %}

You can also check out the [K-Means example](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/clustering/KMeans.java), which uses a BulkIteration to cluster a set of unlabeled points.

#### Delta Iterations

Delta iterations exploit the fact that certain algorithms do not change every data point of the solution in each iteration.

In addition to the partial solution that is fed back (called workset) in every iteration, delta iterations maintain state across iterations (called solution set), which can be updated through deltas. The result of the iterative computation is the state after the last iteration. Please refer to the [Introduction to Iterations]({{site.baseurl}}/docs/0.5/programming_guides/iterations.html) for an introduction to the basic principle of delta iterations.

Defining a DeltaIteration is similar to defining a BulkIteration. For delta iterations, two data sets form the input to each iteration (workset and solution set), and two data sets are produced as the result (new workset, solution set delta) in each iteration.

To create a DeltaIteration call the `iterateDelta(DataSet, int, int)` (or `iterateDelta(DataSet, int, int[])` respectively). This method is called on the initial solution set. The arguments are the initial delta set, the maximum number of iterations and the key positions. The returned `DeltaIterativeDataSet` can be used for operators inside the iteration and represents the work set. You can access the solution set by joining with the returned DataSet from `iteration.getSolutionSet()`.

{% highlight java %}
DataSet<Tuple2<Long, Double>> initialSolutionSet = env
    .readCsvFile(solutionSetInputPath)
    .fieldDelimiter(' ')
    .types(Long.class, Double.class);

DataSet<Tuple2<Long, Double>> initialDeltaSet = env
    .readCsvFile(deltasInputPath)
    .fieldDelimiter(' ')
    .types(Long.class, Double.class);

DataSet<Tuple3<Long, Long, Long>> dependencySetInput = env
    .readCsvFile(dependencySetInputPath)
    .fieldDelimiter(' ')
    .types(Long.class, Long.class, Long.class);

int maxIterations = 100;
int keyPosition = 0;

DeltaIterativeDataSet<Tuple2<Long, Double>, Tuple2<Long, Double>> iteration = initialSolutionSet
    .iterateDelta(initialDeltaSet, maxIterations, keyPosition);

DataSet<Tuple2<Long, Double>> updateRanks = iteration
    .join(dependencySetInput)
    .where(0)
    .equalTo(0)
    .with(new PRDependenciesComputationMatchDelta())
    .groupBy(1)
    .reduceGroup(new UpdateRankReduceDelta());

DataSet<Tuple2<Long, Double>> oldRankComparison = updateRanks
    .join(iteration.getSolutionSet())
    .where(0)
    .equalTo(0)
    .with(new RankComparisonMatch());

iteration.closeWith(oldRankComparison, updateRanks).writeAsCsv(outputPath);
{% endhighlight %}

<div class="back-to-top"><a href="#toc">Back to top</a></div>
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

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="broadcast_variables">
Broadcast Variables
-------------------

You can easily broadcast a data set `DataSet<T>` to all nodes executing a specific operator. The data set will then be accessible at the operator as an `Collection<T>`.

- **Broadcast**: broadcast sets are registered by name via `withBroadcastSet(DataSet, String)`, and
- **Access**: accessible via `getRuntimeContext().getBroadcastVariable(String)` at the target operator.

```java
// 1. The DataSet to be broadcasted
DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3);

DataSet<String> data = env.fromElements("a", "b");

data.map(new MapFunction<String, String>() {
    @Override
    public void open(Configuration parameters) throws Exception {
      // 3. Access the broadcasted DataSet as a Collection
      Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
    }


    @Override
    public String map(String value) throws Exception {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName"); // 2. Broadcast the DataSet
```

Make sure that the names (`broadcastSetName` in the previous example) match when registering and accessing broadcasted data sets. For a complete example program, have a look at [BroadcastVariableExample](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/broadcastvar/BroadcastVariableExample).

**Note**: As the content of broadcast variables is kept in-memory on each node, it should not become too large. For simpler things like scalar values you should use `withParameters(...)`.

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>

<section id="packaging">
Program Packaging & Distributed Execution
-----------------------------------------

As described in the [program skeleton](#skeleton) section, Stratosphere programs can be executed on clusters by using the `RemoteEnvironment`. Alternatively, programs can be packaged into JAR Files (Java Archives) for execution. Packaging the program is a prerequisite to executing them through the [command line interface]({{ site.baseurl }}/docs/0.5/program_execution/cli_client.html) or the [web interface]({{ site.baseurl }}/docs/0.5/program_execution/web_interface.html).

#### Packaging Programs

To support execution from a packaged JAR file via the command line or web interface, a program must use the environment obtained by `ExecutionEnvironment.getExecutionEnvironment()`. This environment will act as the cluster's environment when the JAR is submitted to the command line or web interface. If the Stratosphere program is invoked differently than through these interfaces, the environment will act like a local environment.

To package the program, simply export all involved classes as a JAR file. The JAR file's manifest must point to the class that contains the program's *entry point* (the class with the `public void main(String[])` method). The simplest way to do this is by putting the *main-class* entry into the manifest (such as `main-class: eu.stratosphere.example.MyProgram`). The *main-class* attribute is the same one that is used by the Java Virtual Machine to find the main method when executing a JAR files through the command `java -jar pathToTheJarFile`. Most IDEs offer to include that attribute automatically when exporting JAR files.


#### Packaging Programs through Plans

Additionally, the Java API supports packaging programs as *Plans*. This method resembles the way that the *Scala API* packages programs. Instead of defining a progam in the main method and calling `execute()` on the environment, plan packaging returns the *Program Plan*, which is a description of the program's data flow. To do that, the program must implement the `eu.stratosphere.api.common.Program` interface, defining the `getPlan(String...)` method. The strings passed to that method are the command line arguments. The program's plan can be created from the environment via the `ExecutionEnvironment#createProgramPlan()` method. When packaging the program's plan, the JAR manifest must point to the class implementing the `eu.stratosphere.api.common.Program` interface, instead of the class with the main method.


#### Summary

The overall procedure to invoke a packaged program is as follows:

  1. The JAR's manifest is searched for a *main-class* or *program-class* attribute. If both attributes are found, the *program-class* attribute takes precedence over the *main-class* attribute. Both the command line and the web interface support a parameter to pass the entry point class name manually for cases where the JAR manifest contains neither attribute.
  2. If the entry point class implements the `eu.stratosphere.api.common.Program`, then the system calls the `getPlan(String...)` method to obtain the program plan to execute. The `getPlan(String...)` method was the only possible way of defining a program in the *Record API* (see [0.4 docs]({{ site.baseurl }}/docs/0.4/)) and is also supported in the new Java API.
  3. If the entry point class does not implement the `eu.stratosphere.api.common.Program` interface, the system will invoke the main method of the class.

<div class="back-to-top"><a href="#toc">Back to top</a></div>
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

First you have to create an accumulator object (here a counter) in the operator function where you want to use it. Operator function here refers to the (anonymous inner)
class implementing the user defined code for an operator.

    private IntCounter numLines = new IntCounter();

Second you have to register the accumulator object, typically in the ```open()``` method of the operator function. Here you also define the name.

    getRuntimeContext().addAccumulator("num-lines", this.numLines);

You can now use the accumulator anywhere in the operator function, including in the ```open()``` and ```close()``` methods.

    this.numLines.add(1);

The overall result will be stored in the ```JobExecutionResult``` object which is returned when running a job using the Java API (currently this only works if the execution waits for the completion of the job).

    myJobExecutionResult.getAccumulatorResult("num-lines")

All accumulators share a single namespace per job. Thus you can use the same accumulator in different operator functions of your job. Stratosphere will internally merge all accumulators with the same name.

Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for a complete example.

A note on accumulators and iterations: Currently the result of accumulators is only available after the overall job ended. We plan to also make the result of the previous iteration available in the next iteration.

__Custom accumulators:__

To implement your own accumulator you simply have to write your implementation of the Accumulator interface. Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for an example. Feel free to create a pull request if you think your custom accumulator should be shipped with Stratosphere.

You have the choice to implement either [Accumulator](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Accumulator.java) or [SimpleAccumulator](https://github.com/stratosphere/stratosphere/blob/{{ site.docs_05_stable_gh_tag }}/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/SimpleAccumulator.java). ```Accumulator<V,R>``` is most flexible: It defines a type ```V``` for the value to add, and a result type ```R``` for the final result. E.g. for a histogram, ```V``` is a number and ```R``` is a histogram. ```SimpleAccumulator``` is for the cases where both types are the same, e.g. for counters.

<div class="back-to-top"><a href="#toc">Back to top</a></div>
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

<div class="back-to-top"><a href="#toc">Back to top</a></div>
</section>
