---
layout: inner_docs_v05
title:  "Command Line Interface"
---

## Command Line Interface

Stratosphere provides a command line interface to run programs that are packages as JAR files, and control their execution. 
The command line interface is part of any Stratosphere setup, available in local single node setups, and in distributed setups. It is located under `<stratosphere-home>/bin/stratosphere`, and targets the running Stratosphere master (JobManager) that was started from the same installation directory.

A prerequisite to using the command line interface is that the Stratosphere master (JobManager) has been started (via `start-local.sh` or `start-cluster.sh`).

The command line can be used to

- submit jobs for execution,
- cancel a running job,
- provide information about a job, and
- list running and waiting jobs.

### Usage

The command line syntax is as follows:

```
./stratosphere <ACTION> [OPTIONS] [ARGUMENTS]

General options:
     -h,--help      Show the help for the CLI Frontend, or a specific action.
     -v,--verbose   Print more detailed error messages.


Action "run" - compiles and submits a Stratosphere program that is given in the form of a JAR file.

  "run" options:

     -p,--parallelism <parallelism> The degree of parallelism for the execution. This value is used unless the program overrides the degree of parallelism on the execution environment or program plan (see link to ../programming_guides/java#skeleton) for details. If this option is not set, then the execution will use the default parallelism specified in the stratosphere-conf.yaml file.

     -c,--class <classname>         The class with the entry point (main method, or getPlan() method). Needs only be specified if the JAR file has no manifest pointing to that class. See (link to ../programming_guides/java#packaging) for details.

     -m,--jobmanager <host:port>    Option to submit the program to a different Stratosphere master (JobManager).

  "run" arguments:

     - The first argument is the path to the JAR file of the program.
     - All successive arguments are passed to the program's main method (or getPlan() method).


Action "info" - displays information about a Stratosphere program.

  "info" action arguments:
     -d,--description               Show description of the program, if the main class implements the 'ProgramDescription' interface.

     -e,--plan                      Show the execution data flow plan of the program, in JSON representation.

     -p,--parallelism <parallelism> The degree of parallelism for the execution, see above. The parallelism is relevant for the execution plan. The option is only evaluated if used together with the -e option.

     -c,--class <classname>         The class with the entry point (main method, or getPlan() method). Needs only be specified if the JAR file has no manifest pointing to that class. See (link to ../programming_guides/java#packaging) for details.

     -m,--jobmanager <host:port>    Option to connect to a different Stratosphere master (JobManager). Connecting to a master is relevant to compile the execution plan. The option is only evaluated if used together with the -e option.

  "info" arguments:

     - The first argument is the path to the JAR file of the program.
     - All successive arguments are passed to the program's main method (or getPlan() method).


Action "list" lists submitted Stratosphere programs.

  "list" action arguments:

     -r,--running                   Show running programs and their JobIDs

     -s,--scheduled                 Show scheduled prorgrams and their JobIDs

     -m,--jobmanager <host:port>    Option to connect to a different Stratosphere master (JobManager).


Action "cancel" cancels a submitted Stratosphere program.

  "cancel" action arguments:

     -i,--jobid <jobID>             JobID of program to cancel
```

### Example Usage:

-   Run example program with no arguments.

        ./bin/stratosphere run ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar \
                               -a 4 file:///home/user/hamlet.txt file:///home/user/wordcount_out

-   Run example program with arguments for intput and result files

        ./bin/stratosphere run ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar \
                               file:///home/user/hamlet.txt file:///home/user/wordcount_out

-   Run example program with parallelism 16 and arguments for intput and result files

        ./bin/stratosphere run -p 16 ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar \
                                file:///home/user/hamlet.txt file:///home/user/wordcount_out

-   Run example program on a specific JobManager:

        ./bin/stratosphere run -m myJMHost:6123 \
                               ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar \
                               -file:///home/user/hamlet.txt file:///home/user/wordcount_out


-   Display the expected arguments for the WordCount example program:

        ./bin/stratosphere info -d ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar

-   Display the optimized execution plan for the WordCount example program as JSON:

        ./bin/stratosphere info -p 
                                ./examples/stratosphere-java-examples-{{site.current_stable}}-WordCount.jar \
                                file:///home/user/hamlet.txt file:///home/user/wordcount_out

-   List scheduled and running jobs (including their JobIDs):

        ./bin/stratosphere list -s -r

-   Cancel a job:

        ./bin/stratosphere cancel -i <jobID>
