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

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
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
Linking
-------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
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
to the respective sections for more details.

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

The Stratosphere Java API allows the use of many different data types for the input and output of operators. Data sets as well as functions 
like `MapFunction`, `ReduceFunction`, etc. are parameterized with data types (using Java generics) in order to ensure type-safe programming.

Basically, there are four different categories of data types:

- Java Basic Types
- Tuples
- Custom Types
- Values

All data types are described in the following sections.

### Java Basic Types

As already shown in the [Skeleton Program](#skeleton) section, data sets and functions can handle the basic Java data types e.g. `DataSet<String>`. The API supports all common basic Java data types such as `Integer`, `Long`, `Double`, `Float`, `Boolean`, `Byte`, `Character`, `Short` and `String`.

### Tuples

The API provides classes reaching from `Tuple1` up to `Tuple22` that allow for ordered lists of elements (=*a data structure with multiple fields*). An element of a Tuple can be every Stratosphere data type. Nesting of Tuples is also possible. However, in order to ensure type-safety, Tuples always need to be parameterized (using Java generics). The following code snippet shows an example of how to use Tuples:

```java
// Map function to convert a String to a Tuple containing the String and its length
MapFunction<?, ?> function = new MapFunction<String, Tuple2<String, Integer>>() {

    @Override
    public Tuple2<String, Integer> map(String in) throws Exception {
        return new Tuple2<String, Integer>(in, in.length());
    }
};

// DataSet from an CSV file
DataSet<Tuple2<Integer, String>> nations = env.readCsvFile(tpchNation)
                                            .fieldDelimiter('|')
                                            .includeFields("1100")
                                            .types(Integer.class, String.class);
```

Fields of a Tuple can be accessed directly by using `tuple.f4` or `tuple.getField(4)`. In order to access fields 
more intuitively and generate more readable code, it is also possible to extend a subclass of `Tuple` and add getters and setters with custom names.

### Custom Types

Custom Types allow for using custom Java classes as data types. A Custom Type can also be used within tuples. The following code snippet shows an example of a Custom Type and its usage. Since functions like the `ReduceFunction` or `JoinFunction` currently only support the definition of Tuple fields as keys, a special `KeySelector` needs to be implemented to specify the key for a Custom Type.

```java
public static class MyCustomType {
    public String word;
    public int count;
		
    public MyCustomType() {}

    public MyCustomType(String word, int count) {
        this.word = word;
        this.count = count;
    }

    @Override
    public String toString() {
        return "(" + word + ", " + count + ")";
    }
}

// [...]

DataSet<CustomType> dataSet = data
                               .groupBy(new KeySelector<WC, String>() { public String getKey(MyCustomType v) { return v.word; } })
			       .reduce(reduceFunction);
```


### Values

Stratosphere Values are serializable types which act as a wrapper around Java basic data types and collections implementing the Java `List` or `Map` interfaces. Currently the API supports `IntValue`, `LongValue`, `DoubleValue`, `FloatValue`, `BooleanValue`, `ByteValue`, `CharValue`, `ShortValue`, `StringValue`, `ListValue` and `MapValue`. In most cases Tuples and Basic Types should be preferred to Values.

</section>

<section id="transformations">
Data Transformations
--------------------

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
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

Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
</section>

<section id="broadcast_variables">
Broadcast Variables
-------------------

Broadcast Variables allow to broadcast computation results to all nodes executing an operator. The following example shows how to set a broadcast variable and how to access it within an operator.

{% highlight java %}

DataSet<String> text = env.readTextFile(inputPath1);

DataSet<Integer> someBcInput = env.readCsvFile(inputPath2)
                                .includeFields("100")
                                .types(Integer.class);

// custom map function which accesses a broadcast variable
DataSet<String> output = text.map(new MapFunction<String, String>>() {

    private Collection<Integer> myBcData;

    @Override
    public void open(Configuration parameters) throws Exception {
        // receive the variables' content
        this.myBcData = this.getRuntimeContext().<Integer>getBroadcastVariable("my_bc_var"); 
    }

    @Override
    public Tuple2<String, Integer> map(String in) throws Exception {
        for (Integer i : myBcData) {
       	   // do something with the data
       }
    }
});

output.withBroadcastSet(someBcInput, "my_bc_var"); // set the variable
{% endhighlight %}
*Note*: As the content of broadcast variables is kept in-memory on each node, it should not become too large. For simpler things like scalar values you should use `withParameters(...)`.

An example of how to use Broadcast Variables in practice can be found in the ```eu.stratosphere.example.java.broadcastvar.BroadcastVariableExample``` class.
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

Stratosphere currently has the following **built-in accumulators**. Each of them implements the [Accumulator](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Accumulator.java) interface.

- [__IntCounter__](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/IntCounter.java), [__LongCounter__](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/LongCounter.java) and [__DoubleCounter__](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/DoubleCounter.java): See below for an example using a counter.
- [__Histogram__](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Histogram.java): A histogram implementation for a discrete number of bins. Internally it is just a map from Integer to Integer. You can use this to compute distributions of values, e.g. the distribution of words-per-line for a word count program.

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

Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for a complete example.

A note on accumulators and iterations: Currently the result of accumulators is only available after the overall job ended. We plan to also make the result of the previous iteration available in the next iteration.

__Custom accumulators:__

To implement your own accumulator you simply have to write your implementation of the Accumulator interface. Please look at the [WordCountAccumulator example](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/wordcount/WordCountAccumulators.java) for an example. Feel free to create a pull request if you think your custom accumulator should be shipped with Stratosphere.

You have the choice to implement either [Accumulator](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/Accumulator.java) or [SimpleAccumulator](https://github.com/stratosphere/stratosphere/blob/master/stratosphere-core/src/main/java/eu/stratosphere/api/common/accumulators/SimpleAccumulator.java). ```Accumulator<V,R>``` is most flexible: It defines a type ```V``` for the value to add, and a result type ```R``` for the final result. E.g. for a histogram, ```V``` is a number and ```R``` is a histogram. ```SimpleAccumulator``` is for the cases where both types are the same, e.g. for counters.
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
