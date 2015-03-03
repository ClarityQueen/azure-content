<properties 
	pageTitle="How to use Apache Giraph with Azure HDInsight" 
	description="Learn how to use Apache Giraph to perform graph processing with Azure HDInsight" 
	services="hdinsight" 
	documentationCenter="" 
	authors="blackmist" 
	manager="paulettm" 
	editor=""/>

<tags 
	ms.service="hdinsight" 
	ms.workload="big-data" 
	ms.tgt_pltfrm="na" 
	ms.devlang="na" 
	ms.topic="article" 
	ms.date="08/14/2014" 
	ms.author="larryfr"/>

#Learn how to use Apache Giraph with Azure HDInsight

[Apache Giraph][giraph] enables you to perform graph processing by using Hadoop, and it can be used with Azure HDInsight. 

Graphs model relationships between objects, such as the connections between routers on a large network like the Internet, or relationships between people on social networks (sometimes referred to as a social graph). Graph processing allows you to reason about the relationships between objects in a graph, such as:

* Identifying potential friends based on your current relationships.

* Identifying the shortest route between two computers in a network.

* Calculating the page rank of web pages.

##You will learn how to

* [Build and deploy Apache Giraph to an HDInsight cluster](#build)

* [Run the SimpleShortestPathsComputation example](#run)

	For a list of other examples provided with Giraph, see [Package org.apache.giraph.examples](https://giraph.apache.org/apidocs/org/apache/giraph/examples/package-summary.html).

* [Troubleshoot problems you may encounter](#tshoot)

##Requirements

* An Azure HDInsight cluster, version 3.1 or 3.0

* [Git](http://git-scm.com/)

* Java 1.6

* [Apache Maven](http://maven.apache.org/) 3.0 or higher

##<a id="build"></a>Build and deploy Giraph

Giraph is not provided as part of the HDInsight cluster, so it must be built from the source. You can find more information about building Giraph in the [Giraph repository](https://github.com/apache/giraph).

1.  As of July 2014, Giraph requires a patch to work with the blobs (formerly known as Windows Azure Storage Blob or WASB) that are used by HDInsight for storage. The patch has been submitted to the Apache Giraph project, but it has not been accepted yet. Download the patch from the __attachments__ section of [GIRAPH-930](https://issues.apache.org/jira/browse/GIRAPH-930) and save it to the local drive as __giraph-930.diff__.

1. From a command line, use the following Git command to create a clone of the Giraph repository.

		git clone https://github.com/apache/giraph.git

2. Change to the __giraph__ directory that you created with the clone operation in Step 2.

		cd giraph

3. Merge the patch into the local repository by using the following command:

		git apply giraph-930.diff

	Replace __giraph-930.diff__ with the path to the file that you created in Step 1.

3. Build Giraph for your HDInsight cluster version by using one of the following commands:

	* For __HDInsight 3.0__ (Hadoop 2.2)

			mvn package -Phadoop_0.20.203 - DskipTests

	* For __HDInsight 3.1__ (Hadoop 2.4)

			mvn package -Phadoop_0.23 -DskipTests

	After the build is complete, you will find the examples Java Archive (JAR) file at __\\giraph\\giraph-examples\\target__.

4. Upload the example JAR file to the primary storage site for your HDInsight cluster by using [Azure PowerShell][aps] and the [HDInsight-Tools][tools]:

		Add-HDInsightFile giraph-examples-1.1.0-SNAPSHOT-for-hadoop-0.23.1-jar-with-dependencies.jar example/jars/giraph.jar clustername

	Replace __giraph-examples-1.1.0-SNAPSHOT-for-hadoop-0.23.1-jar-with-dependencies.jar__ with the path and name of the JAR file that was produced in the previous step, and replace __clustername__ with the name of your HDInsight cluster. For example, if you build the package with the `-Phadoop_0.20.203` parameter, the file name of the JAR will include __hadoop-0.20.203__.

	After the command completes, you will find the JAR file at wasb:///example/jars/giraph.jar.

	> [AZURE.NOTE] for a list of tools that can be used to upload files to HDInsight, see [Upload data for Hadoop jobs in HDInsight](http://azure.microsoft.com/documentation/articles/hdinsight-upload-data/).

##<a id="run"></a>Run the example

The SimpleShortestPathsComputation sample demonstrates the basic [Pregel](http://people.apache.org/~edwardyoon/documents/pregel.pdf) implementation for finding the shortest path between objects in a graph. Use the following steps to upload the sample data, run a job by using the SimpleShortestPathsComputation example, and then view the results.

> [AZURE.NOTE] The source for this and other examples is available in the [release-1.1 branch](https://github.com/apache/giraph/tree/release-1.1) of the [GitHub repository](https://github.com/apache/giraph).

1. Create a new file named __tiny\_graph.txt__. It should contain the following lines:

		[0,0,[[1,1],[3,3]]]
		[1,0,[[0,1],[2,2],[3,1]]]
		[2,0,[[1,2],[4,4]]]
		[3,0,[[0,3],[1,1],[4,4]]]
		[4,0,[[3,4],[2,4]]]

	This data describes a relationship between objects in a [directed graph](http://en.wikipedia.org/wiki/Directed_graph) by using the format `[source_id,source_value,[[dest_id], [edge_value],...]]`. Each line represents a relationship between a __source\_id__ and one or more __dest\_id__ objects. The __edge\_value__ (or weight) can be thought of as the strength or distance of the connection between the __source\_id__ and the __dest\_id__. 

	Drawn out and using the value (or weight) as the distance between objects, the previous data might look like this:

	![tiny_graph.txt drawn as circles with lines of varying distance between](.\media\hdinsight-giraph\giraph-graph.png)


2. Upload the __tiny\_graph.txt__ file to the primary storage for your HDInsight cluster by using [Azure PowerShell][aps] and the [HDInsight-Tools][tools]:

		Add-HDInsightFile tiny_graph.txt example/data/tiny_graph.txt clustername

	Replace __clustername__ with the name of your HDInsight cluster.

3. Use the following Windows PowerShell command to run the __SimpleShortstPathsComputation__ example, and use the __tiny\_graph.txt__ file as input. This requires that you have installed and configured [Azure PowerShell][aps].

		$clusterName = "clustername"
		# Giraph examples JAR
		$jarFile = "wasb:///example/jars/giraph.jar"
		# Arguments for this job
		$jobArguments = "org.apache.giraph.examples.SimpleShortestPathsComputation", `
		                "-ca", "mapred.job.tracker=headnodehost:9010", `
		                "-vif", "org.apache.giraph.io.formats.JsonLongDoubleFloatDoubleVertexInputFormat", `
		                "-vip", "wasb:///example/data/tinygraph.txt", `
		                "-vof", "org.apache.giraph.io.formats.IdWithValueTextOutputFormat", `
		                "-op",  "wasb:///example/output/shortestpaths", `
		                "-w", "2"
		# Create the definition
		$jobDefinition = New-AzureHDInsightMapReduceJobDefinition `
		  -JarFile $jarFile `
		  -ClassName "org.apache.giraph.GiraphRunner" `
		  -Arguments $jobArguments
		
		# Run the job, write output to the PowerShell window
		$job = Start-AzureHDInsightJob -Cluster $clusterName -JobDefinition $jobDefinition
		Write-Host "Wait for the job to complete ..." -ForegroundColor Green
		Wait-AzureHDInsightJob -Job $job
		Write-Host "STDERR"
		Get-AzureHDInsightJobOutput -Cluster $clusterName -JobId $job.JobId -StandardError
		Write-Host "Display the standard output ..." -ForegroundColor Green
		Get-AzureHDInsightJobOutput -Cluster $clusterName -JobId $job.JobId -StandardOutput

	In the previous example, replace __clustername__ with the name of your HDInsight cluster.

###View the results

When the job has completed, the results will be stored in the __wasb:///example/out/shotestpaths__ folder as __part-m-#####__ files. Use [Azure PowerShell][aps] and the [HDInsight-Tools][tools] to download the output files:

	Find-HDInsightFile example/output/shortestpaths/part* clustername | foreach-object { Get-HDInsightFile $_.name .  itsfullofstorage }
	Cat example/output/shortestpaths/part*

This creates the __example/output/shortestpaths__ directory structure in the current directory, and downloads the files beginning with __part__. The __Cat__ cmdlet displays the content of the files, which should appear similar to the following:

	0	1.0
	4	5.0
	2	2.0
	1	0.0
	3	1.0

The SimpleShortestPathComputation example is hard coded to start with the object ID 1 and find the shortest path to other objects. So the output should be read as `destination_id distance`, where distance is the value (or weight) of the edges traveled between object ID 1 and the target ID.

You can verify the results by traveling the shortest paths between ID 1 and all other objects. Note that the shortest path between ID 1 and ID 4 is 5. This is the total distance between <span style="color:orange">ID 1 and ID  3</span>, and then between <span style="color:red">ID 3 and ID 4</span>.

![Drawing of objects as circles with shortest paths drawn between](.\media\hdinsight-giraph\giraph-graph-out.png)

##<a id="tshoot"></a>Troubleshooting

####Output directory already exists

Giraph jobs create the specified output directory at run time. If the directory already exists, an error message appears that states the output directory already exists.

If you want to run a job multiple times, you must either remove the output directory between jobs or specify a different output directory for each job.

####<a id="cmd"></a>Using the Hadoop command line

This article demonstrates how to run a Giraph job by using Windows PowerShell, but you can also run the job by using the Hadoop command line.

> [AZURE.NOTE] The Hadoop command line is only available when you are connected to the HDInsight cluster by using Remote Desktop.
> 
> Remote Desktop sessions to Azure compute resources, such as the HDInsight cluster, may only work from Windows-based Remote Desktop clients.

To connect to the HDInsight cluster, perform the following steps:

1. In the [Azure portal](https://manage.windowsazure.com), select your HDInsight cluster, and then select __Configuration__.

2. At the bottom of the page, select __Enable Remote__ and provide the user name, password, and expiration date for the Remote Desktop connection.

3. After the request to enable Remote Desktop is processed, a new entry to __Connect__ appears at the bottom of the page. Select this to download the .RDP file for the Remote Desktop session.

4. The .RDP file can be saved or opened immediately to launch the Remote Desktop client. During the connection process, you will be asked to provide the user name and password that you used when you enabled the Remote Desktop connection.

5. When connected, use the __Hadoop command line__ icon on the desktop to start the Hadoop command line.

6. The following example demonstrates how to copy the __giraph.jar__ file to the cluster head node, and then use the Hadoop command line to run the job.

		hadoop fs -copyToLocal wasb:///example/jar/giraph.jar
		hadoop jar giraph.jar org.apache.giraph.GiraphRunner org.apache.giraph.examples.SimpleShortestPathsComputation -ca "mapred.job.tracker=headnodehost:9010" -vif org.apache.giraph.io.formats.JsonLongDoubleFloatDoubleVertexInputFormat -vip wasb:///example/data/tinygraph.txt -vof org.apache.giraph.io.formats.IdWithValueTextOutputFormat -op wasb:///example/output/shortestpaths -w 2


####Older versions of HDInsight

If you want to use Giraph with older versions of HDInsight, you must compile it for the specific Hadoop version that is supported by that version. See [What's new in HDInsight cluster versions](http://azure.microsoft.com/documentation/articles/hdinsight-component-versioning/) to determine the version of Hadoop that corresponds with your HDInsight version.

Additionally, older versions of HDInsight may require you to run the Giraph job from the Hadoop command line. If you receive error messages when you run the job from Windows PowerShell, try running the job from the [Hadoop command line](#cmd).

##Next steps

Now that you have learned how to use Giraph with HDInsight, try using [Pig][] and [Hive][] with HDInsight.

[giraph]: http://giraph.apache.org
[tools]: https://github.com/Blackmist/hdinsight-tools
[aps]: http://azure.microsoft.com/documentation/articles/install-configure-powershell/
[pig]: http://azure.microsoft.com/documentation/articles/hdinsight-use-pig/
[hive]: http://azure.microsoft.com/documentation/articles/hdinsight-use-hive/
