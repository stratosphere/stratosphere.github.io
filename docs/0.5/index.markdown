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
[<img src='img/AmazonHome.PNG' style='width: 100%;' title='Amazon Home' />](img/AmazonHome.PNG)
#### Create an EC2 key pair. <br/>

  1. Click on [EC2](https://console.aws.amazon.com/ec2/v2/home#KeyPairs:)
  2. Create new EC2 key pair.<br/>
  [<img src='img/KeyPairs.PNG' style='width: 100%;' title='Create new EC2 key pair' />](img/KeyPairs.PNG) <br/><br/><br/>
  3. Save it locally.<br/>
  
   * Key pairs are used to SSH into your instances. Read more about [how to access your instances](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html)<br/>
   * The access key gives one access to your full account. Keep it save! The access key file is not needed but EMR does not work without having created an access key. 

#### Create an Access key
 1. Click on Security Credentials in your account management tab in the right upper corner.
[<img src='img/IrelandSecurityCredentialsMenue.PNG' style='width: 100%;' title='Click on Security Credentials in the tab' />](img/IrelandSecurityCredentialsMenue.PNG) <br/><br/><br/>
 2. Continue to your security credentials.
[<img src='img/SecurityCredentialsFirst.PNG' style='width: 100%;' title='Continue to Security Credentials 'Click on Security Credentials in the tab' />](img/SecurityCredentialsFirst.PNG)<br/><br/><br/>
 3. Click on Access Keys and create a new one.
[<img src='img/SecurityCredentials.PNG' style='width: 100%;' title='Create new Acces Key' />](img/SecurityCredentials.PNG)<br/><br/><br/>
 4. Save access key locally if you need to use it later. Note: The access key is used internally by the EMR service. So it does not need to be downloaded. <br/>

<br/><br/><br/><br/><br/>
#### Creating an Elastic MapReduce Cluster
1. Click on [ElasticMapreduce](https://console.aws.amazon.com/elasticmapreduce/vnext/home)
 [<img src='img/AmazonHome.PNG' style='width: 100%;' title='Amazon Home' />](img/AmazonHome.PNG)<br/><br/><br/>
2. Click on create cluster [<img src='img/EMRNew.PNG'  style='width: 100%;' title='New EMR Cluster'/>](img/EMRNew.PNG)

<br/><br/><br/><br/><br/>
#### Step 'Set up cluster':
1. Chose a name </br>
2. Chose AMI version with at least Hadoop 2.2.0. AMI 3.0.3 (Hadoop 2.2.0) for example. <br/>
3. Remove all applications which additionally will be installed. 
  * Stratosphere does not need any additional applications installed. It runs on top of Hadoop YARN and Hadoop Distributed File System.

[<img src='img/SoftwareConfiguration.PNG' style='width: 100%;' title='Software Configuration' />](img/SoftwareConfiguration.PNG) <br/><br/><br/>
4. Choose number and type of instances.
 * The Stratosphere JobManger (Stratosphere master) runs on the master instance 
 * Stratosphere TaskManagers (Stratosphere worker/slave) run on core instances.

[<img src='img/HardwareConfigurationMedium.PNG' style='width: 100%;' title='Setting up hardware configuration' />](img/HardwareConfigurationMedium.PNG)<br/><br/><br/>
5. Choose Amazon EC2 key pair for SSH access <br/>
[<img src='img/SecurityAndAccess.PNG' style='width: 100%;' title='Security and access configuration. Select your EC@ key pair.'/>](img/SecurityAndAccess.PNG)     

<br/><br/><br/><br/><br/>
#### Step 'Create step to run stratosphere'
1. Select step 'Custom jar'.
2. Click 'Configure and add'.
3. Copy 's3://elasticmapreduce/libs/script-runner/script-runner.jar' into Jar S3 Location
Copy 's3n://stratosphere-bootstrap/installStart-stratosphere-yarn.sh -n 1 -j 1024 -t 1024' into arguments.
 * -n is the number of TaskManagers. Should be the same number as the core instance count or less.
 * -j memory (heapspace) for the JobManager.
 * -t memory for the TaskManagers.

[<img src='img/StepStartStratosphere.png' style='width: 100%;' title='Configure and add step' />](img/StepStartStratosphere.png.PNG)<br/><br/><br/>
4. Save step.

<br/><br/><br/><br/><br/>
#### Create cluster and reuse it
* Click create cluster to start the Amazon instances and install Stratosphere on them. 
* It will take some time until Stratosphere is started, the completed installation step will indicate that Stratosphere is running.
* Use {Master public DNS}:9026 to access the YARN interface. To access on port 9026, the EMR master Security Group (under EC2 -> NETWORK & SECURITY-> Security Groups. It is normally called: 'ElasticMapReduce-master') needs to allow access on port 9026. For more information on how to allow access to your instances, [read here](http://docs.aws.amazon.com/gettingstarted/latest/wah/getting-started-security-group.html).
* The settings can be copied by cloning this cluster. This cluster can be reused as a template for a configured and running Stratosphere cluster. 

[<img src='img/StepCompleted.PNG' style='width: 100%;' title='Running stratosphere'/>](img/StepCompleted.PNG) 

<br/><br/><br/><br/><br/>
#### Accessing Your Stratosphere Interface
* You need to allow access to your master on TCP port 9026 and 9046 from outside
* First access your YARN interface by typing into your browser '{Master public DNS}:9026'. Replace {Master public DNS} with the actual master public DNS or public IP. You can find it at the top of your cluster summary.
* Copy the ID of the running Stratosphere application

[<img src='img/YarnApplication.png' style='width: 100%;' title='Configure and add step' />](img/YarnApplication.png)

* Type into your browser '{Master public DNS}:9046/proxy/{Stratospher Apllication ID}/index.html'. Replace {Stratospher Apllication ID} with the application ID which can be find on the YARN web interface.
* You now see the Stratosphere Dashboard.

[<img src='img/StratosphereInterfaceProxy.png' style='width: 100%;' title='Configure and add step' />](img/StratosphereInterfaceProxy.png)


* Stratosphere is running!

<br/><br/><br/><br/><br/>
#### Troubleshoot - What to do when something went wrong?:
<br/><br/><br/><br/><br/>
##### Termination with errors No active keys found for user account - Create AWS Access Key
[<img src='img/TerminatedWithErrors.PNG' style='width: 100%;' title='Cluster terminated with errors' />](img/TerminatedWithErrors.PNG)<br/><br/><br/>
1. Create access key. Described in "new to Amazon?".
</section>
