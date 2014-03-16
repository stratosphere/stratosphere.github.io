--- 
layout: inner_with_sidebar
title: "Documentation (0.5-SNAPSHOT)"
links: 
  - { anchor: "jdbc", title: "JDBC Input/Output Format" }
  - { anchor: "collection_data_source", title: "CollectionDataSource" }
  - { anchor: "broadcast_variables", title: "Broadcast Variables" }
  - { anchor: "emr_tutorial", title: "Stratosphere in EMR" }
---

<p class="lead">
Documentation on *new features* and *changes* contained in the current *0.5-SNAPSHOT* development branch. It should be read *in addition* to the documentation for the latest 0.4 release.

<section id="jdbc">
### JDBC Input/Output Format

The JDBC input and output formats allow to read from and write to any JDBC-accessible database.
</section>

<section id="collection_data_source">
### CollectionDataSource

The CollectionDataSource allows you to use local Java and Scala Collections as input to your Stratosphere programs.
</section>

<section id="broadcast_variables">
### Broadcast Variables

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

An example of how to use Broadcast Variables in practice can be found in the <a href="https://github.com/stratosphere/stratosphere/blob/master/stratosphere-examples/stratosphere-java-examples/src/main/java/eu/stratosphere/example/java/record/kmeans/KMeans.java">K-Means example</a>.
</section>

<section id='emr_tutorial'>
### Stratosphere in Amazon Elastic MapReduce (EMR)

#### Introduction
* This tutorial explains how to deploy a Stratosphere environment with Amazon Elastic MapReduce. 
* Setup in this tutorial can be used as a template for future configurations

<br/><br/><br/><br/><br/>
#### First steps: New to Amazon?
[<img src='img/AmazonHome.png' style='width: 100%;' title='Amazon Home' />](img/AmazonHome.png)
#### Create an EC2 key pair. <br/>

  1. Click on [EC2](https://console.aws.amazon.com/ec2/v2/home#KeyPairs:)
  2. Create new EC2 key pair.<br/>
  [<img src='img/KeyPairs.png' style='width: 100%;' title='Create new EC2 key pair' />](img/KeyPairs.png) <br/><br/><br/>
  3. Save it locally.<br/>
  
   * Key pairs are used to SSH into your instances. Read more about [how to access your instances](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html)<br/>
   * The access key gives one access to your full account. Keep it save! The access key file is not needed but EMR does not work without having created an access key. 

#### Create an Access key
 1. Click on Security Credentials in your account management tab in the right upper corner.
[<img src='img/IrelandSecurityCredentialsMenue.png' style='width: 100%;' title='Click on Security Credentials in the tab' />](img/IrelandSecurityCredentialsMenue.png) <br/><br/><br/>
 2. Continue to your security credentials.
[<img src='img/SecurityCredentialsFirst.png' style='width: 100%;' title='Continue to Security Credentials 'Click on Security Credentials in the tab' />](img/SecurityCredentialsFirst.png)<br/><br/><br/>
 3. Click on Access Keys and create a new one.
[<img src='img/SecurityCredentials.png' style='width: 100%;' title='Create new Acces Key' />](img/SecurityCredentials.png)<br/><br/><br/>
 4. Save access key locally. <br/>

<br/><br/><br/><br/><br/>
#### Creating an Elastic MapReduce Cluster
1. Click on [ElasticMapreduce](https://console.aws.amazon.com/elasticmapreduce/vnext/home)
 [<img src='img/AmazonHome.png' style='width: 100%;' title='Amazon Home' />](img/AmazonHome.png)<br/><br/><br/>
2. Click on create cluster [<img src='img/EMRNew.png'  style='width: 100%;' title='New EMR Cluster'/>](img/EMRNew.png)

<br/><br/><br/><br/><br/>
#### Step 'Set up cluster':
1. Chose a name </br>
2. Chose AMI version with at least Hadoop 2.2.0. AMI 3.0.3 (Hadoop 2.2.0) for example. <br/>
3. Remove all applications which additionally will be installed. 
  * Stratosphere does not need any additional applications installed. It runs on top of Hadoop Yarn and Hadoop Distributed File System.

[<img src='img/SoftwareConfiguration.png' style='width: 100%;' title='Software Configuration' />](img/SoftwareConfiguration.png) <br/><br/><br/>
4. Choose number and type of instances.
 * The Stratosphere JobManger (Stratosphere master) runs on the master instance 
 * Stratosphere TaskManagers (Stratosphere worker/slave) run on core instances.

[<img src='img/HardwareConfigurationMedium.png' style='width: 100%;' title='Setting up hardware configuration' />](img/HardwareConfigurationMedium.png)<br/><br/><br/>
5. Choose Amazon EC2 key pair for SSH access <br/>
[<img src='img/SecurityAndAccess.png' style='width: 100%;' title='Security and access configuration. Select your EC@ key pair.'/>](img/SecurityAndAccess.png)     

<br/><br/><br/><br/><br/>
#### Step 'Create bootstrap-action':
1. Select bootstrap-action 'run-if'. <br/>
2. Click 'Configure and add'. <br/>
3. Copy 'instance.isMaster=true s3n://stratosphere-bootstrap/install-stratosphere-yarn.sh' into arguments. <br/>
 * This will run the Stratosphere installation on the master node. <br/>

[<img src='img/BootstrapActionStratosphere.png' style='width: 100%;' title='Configure and add bootstrap action.' />](img/BootstrapActionStratosphere.png)<br/><br/><br/>
4. Add bootstrap action.

<br/><br/><br/><br/><br/>
#### Step 'Create step to run stratosphere'
1. Select step 'Custom jar'.
2. Click 'Configure and add'.
3. Copy 's3://elasticmapreduce/libs/script-runner/script-runner.jar' into Jar S3 Location
Copy '/home/hadoop/start-stratosphere.sh -n 2 -j 1024 -t 1024' into arguments.
 * -n is the number of TaskManagers.
 * -j memory (heapspace) for the JobManager.
 * -t memory for the TaskManagers.

[<img src='img/StepsRunStratosphere.png' style='width: 100%;' title='Configure and add step' />](img/StepsRunStratosphere.png)<br/><br/><br/>
4. Add step.

<br/><br/><br/><br/><br/>
#### Create cluster and reuse it
* Click create cluster to start the Amazon instances and install Stratosphere on them. 
* It will take some time until Stratosphere is started, the completed installation step will indicate that Stratosphere is running.

[<img src='img/StepCompleted.png' style='width: 100%;' title='Running stratosphere'/>](img/StepCompleted.png)
* The settings can be copied by cloning this cluster. Hence this cluster can be reused as a templated for a configured and running Stratosphere cluster. 


###### Accessing Stratosphere Interface
* The Yarn interface and Stratopshere interface will be up. [This tutorial (Step 4)](http://stratosphere.eu/blog/tutorial/2014/02/18/amazon-elastic-mapreduce-cloud-yarn.html) shows how to get access to both interfaces.

<br/><br/><br/><br/><br/>
#### Troubleshoot - What to do when something went wrong?:
<br/><br/><br/><br/><br/>
##### Termination with errors No active keys found for user account - Create AWS Access Key
[<img src='img/TerminatedWithErrors.png' style='width: 100%;' title='Cluster terminated with errors' />](img/TerminatedWithErrors.png)<br/><br/><br/>
1. Create access key. Described in "new to Amazon?".
</section>
