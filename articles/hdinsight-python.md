<properties 
	pageTitle="Use Python with Hive and Pig in Azure HDInsight" 
	description="Learn how to use Python User Defined Functions (UDF) from Hive and Pig in Azure HDInsight." 
	services="hdinsight" 
	documentationCenter="" 
	authors="Blackmist" 
	manager="paulettm" 
	editor="cgronlun"/>

<tags 
	ms.service="hdinsight" 
	ms.workload="big-data" 
	ms.tgt_pltfrm="na" 
	ms.devlang="python" 
	ms.topic="article" 
	ms.date="02/20/2015" 
	ms.author="larryfr"/>

#Use Python with Hive and Pig in HDInsight

Hive and Pig are great for working with data in HDInsight, but sometimes you need a more general purpose language. Hive and Pig allow you to create user-defined functions (UDF) by using a variety of programming languages. In this article, you will learn how to use a Python UDF from Hive and Pig.

> [AZURE.NOTE] The steps in this article apply to HDInsight cluster versions 3.2, 3.1, 3.0, and 2.1.


##<a name="python"></a>Python on HDInsight

Introduced in HDInsight 3.0 clusters, Python 2.7 is installed by default. Hive can be used with this version of Python for stream processing (data is passed between Hive and Python by using STDOUT/STDIN).

HDInsight also includes Jython, which is a Python implementation written in Java. Pig understands how to talk to Jython without having to resort to streaming, so it's preferable when using Pig.

###<a name="hivepython"></a>Hive and Python

Python can be used as a UDF from Hive through the HiveQL **TRANSFORM** statement. For example, the following HiveQL invokes a Python script that is stored in the **streaming.py** file:

**Linux-based HDInsight**

	add file wasb:///streaming.py;
	
	SELECT TRANSFORM (clientid, devicemake, devicemodel)
	  USING 'streaming.py' AS
	  (clientid string, phoneLable string, phoneHash string)
	FROM hivesampletable
	ORDER BY clientid LIMIT 50;

**Windows-based HDInsight**

	add file wasb:///streaming.py;
	
	SELECT TRANSFORM (clientid, devicemake, devicemodel)
	  USING 'D:\Python27\python.exe streaming.py' AS
	  (clientid string, phoneLable string, phoneHash string)
	FROM hivesampletable
	ORDER BY clientid LIMIT 50;

> [AZURE.NOTE] In Windows-based HDInsight clusters, the **USING** clause must specify the full path to python.exe. This is always `D:\Python27\python.exe`.

Here's what this example does:

1. The **add file** statement at the beginning of the file adds the **streaming.py** file to the distributed cache, so it's accessible by all nodes in the cluster.

2. The  **SELECT TRANSFORM ... USING** statement selects data from the **hivesampletable**, and passes **clientid**, **devicemake**, and **devicemodel** to the **streaming.py** script.

3. The **AS** clause describes the fields that are returned from **streaming.py**.

<a name="streamingpy"></a>
Here's the **streaming.py** file that is used by the HiveQL example:

	#!/usr/bin/env python

	import sys
	import string
	import hashlib
	
	while True:
	  line = sys.stdin.readline()
	  if not line:
	    break
	
	  line = string.strip(line, "\n ")
	  clientid, devicemake, devicemodel = string.split(line, "\t")
	  phone_label = devicemake + ' ' + devicemodel
	  print "\t".join([clientid, phone_label, hashlib.md5(phone_label).hexdigest()])

Because we are using streaming, this script has to do the following:

1. Read data from STDIN. This is accomplished by using `sys.stdin.readline()` in this example.

2. The trailing new line character is removed by using `string.strip(line, "\n ")`, because we want only the text data and not the end-of-line indicator.

2. When stream processing, a single line contains all the values with a Tab character between each value. So `string.split(line, "\t")` can be used to split the input at each Tab, and return only the fields.

3. When the processing is complete, the output must be written to STDOUT as a single line, with a Tab between each field. This is accomplished by using `print "\t".join([clientid, phone_label, hashlib.md5(phone_label).hexdigest()])`.

4. This all occurs within a `while` loop, that will repeat until no `line` is read, at which point `break` exits the loop, and the script terminates.

Beyond that, the script concatenates the input values for `devicemake` and `devicemodel`, and it calculates a hash of the concatenated value. Pretty simple, but it describes the basics of how any Python script that is invoked from Hive should function: loop, read input until there is no more, break each line of input apart at the Tabs, process, and write a single line of Tab-delimited output.

See [Running the examples](#running) for how to run this example in your HDInsight cluster.

###<a name="pigpython"></a>Pig and Python

A Python script can be used as a UDF from Pig through the **GENERATE** statement. The following example uses a Python script that is stored in the **jython.py** file:

	Register 'wasb:///jython.py' using jython as myfuncs;
    LOGS = LOAD 'wasb:///example/data/sample.log' as (LINE:chararray);
    LOG = FILTER LOGS by LINE is not null;
    DETAILS = FOREACH LOG GENERATE myfuncs.create_structure(LINE);
    DUMP DETAILS;

Here's how this example works:

1. It registers the file that contains the Python script (**jython.py**) by using **Jython**, and it exposes the file to Pig as **myfuncs**. Jython is a Python implementation in Java, and it runs in the same Java virtual machine as Pig. This allows us to treat the Python script like a traditional function call vs. the streaming approach that is used with Hive.

2. The next line loads the sample data file, **sample.log** into **LOGS**. Because this log file doesn't have a consistent schema, it also defines each record (**LINE** in this case) as a **chararray**. A chararray is, essentially, a string.

3. The third line filters out any null values and stores the result of the operation into **LOG**.

4. Next, it iterates over the records in **LOG** and uses **GENERATE** to invoke the **create_structure** method. This method is contained in the **jython.py** script, which is loaded as **myfuncs**.  **LINE** is used to pass the current record to the function.

5. Finally, the output is dumped to STDOUT by using the **DUMP** command. This is to immediately show the results after the operation completes. In a real script, you would normally use **STORE** to store the data in a new file.

<a name="jythonpy"></a>
Here's the **jython.py** script that is used by the Pig example:

	@outputSchema("log: {(date:chararray, time:chararray, classname:chararray, level:chararray, detail:chararray)}")
	def create_structure(input):
	  if (input.startswith('java.lang.Exception')):
	    input = input[21:len(input)] + ' - java.lang.Exception'
	  date, time, classname, level, detail = input.split(' ', 4)
	  return date, time, classname, level, detail

Remember that we previously defined the **LINE** input as a chararray because there was no consistent schema for the input? The **jython.py** script transforms the data into a consistent schema for output. It works like this:

1. The **@outputSchema** statement defines the format of the data that will be returned to Pig. In this case, it's a **data bag**, which is a Pig data type. The bag contains the following fields, all of which are chararray (strings):

	* date: The date the log entry was created.
	* time: The time the log entry was created.
	* classname: The class name the entry was created for.
	* level: The log level.
	* detail: The verbose details for the log entry.

2. Next, **def create_structure(input)** defines the function that Pig will pass line items to.

3. The example data, **sample.log**, mostly conforms to the date, time, classname, level, and detail schema that we want to return. But it also contains a few lines that begin with the string *java.lang.Exception*, which need to be modified to match the schema. The **if** statement checks for those, then massages the input data to move the *java.lang.Exception* string to the end, bringing the data inline with our expected output schema.

4. Next, the **split** command is used to split the data at the first four space characters. This results in five values, which are assigned into **date**, **time**, **classname**, **level**, and **detail**.

5. Finally, the values are returned to Pig.

When the data is returned to Pig, it will have a consistent schema as defined in the **@outputSchema** statement.

##<a name="running"></a>Running the examples

If you are using a Linux-based HDInsight cluster, use the **SSH** steps that follow. If you are using a Windows-based HDInsight cluster and a Windows client, use the **Windows PowerShell** steps.

###SSH

For more information about using SSH, see <a href="../hdinsight-hadoop-linux-use-ssh-unix/" target="_blank">Use SSH with Linux-based Hadoop on HDInsight from Linux, Unix, or OS X</a> or <a href="../hdinsight-hadoop-linux-use-ssh-windows/" target="_blank">Use SSH with Linux-based Hadoop on HDInsight from Windows</a>.

1. Use the Python examples [streaming.py](#streamingpy) and [jython.py](#jythonpy) to create local copies of the files on your development machine.

2. Use `scp` to copy the files to your HDInsight cluster. For example, the following code would copy the files to a cluster named **mycluster**:

		scp streaming.py jython.py myuser@mycluster-ssh.azurehdinsight.net:

3. Use SSH to connect to the cluster. For example, the following code would connect to a cluster named **mycluster** as the user **myuser**:

		ssh myuser@mycluster-ssh.azurehdinsight.net 

4. From the SSH session, add the Python files that you uploaded previously to the blob storage for the cluster.

		hadoop fs -copyFromLocal streaming.py /streaming.py
		hadoop fs -copyFromLocal jython.py /jython.py

After uploading the files, use the following steps to run the Hive and Pig jobs.

####Hive

1. Use the `hive` command to start the Hive shell. You should see a `hive>` prompt when the shell has loaded.

2. Enter the following code at the `hive>` prompt:

		add file wasb:///streaming.py;
		SELECT TRANSFORM (clientid, devicemake, devicemodel)
		  USING 'streaming.py' AS
		  (clientid string, phoneLabel string, phoneHash string)
		FROM hivesampletable
		ORDER BY clientid LIMIT 50;

3. After you enter the last line, the job should start. Eventually, it will return output similar to the following:

		100041	RIM 9650	d476f3687700442549a83fac4560c51c
		100041	RIM 9650	d476f3687700442549a83fac4560c51c
		100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9
		100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9
		100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9

####Pig

1. Use the `pig` command to start the Pig shell. You should see a `grunt>` prompt when the shell has loaded.

2. Enter the following statements at the `grunt>` prompt.

		Register wasb:///jython.py using jython as myfuncs;
	    LOGS = LOAD 'wasb:///example/data/sample.log' as (LINE:chararray);
	    LOG = FILTER LOGS by LINE is not null;
	    DETAILS = foreach LOG generate myfuncs.create_structure(LINE);
	    DUMP DETAILS;

3. After you enter the following line, the job should start. Eventually, it will return output similar to the following:

		((2012-02-03,20:11:56,SampleClass5,[TRACE],verbose detail for id 990982084))
		((2012-02-03,20:11:56,SampleClass7,[TRACE],verbose detail for id 1560323914))
		((2012-02-03,20:11:56,SampleClass8,[DEBUG],detail for id 2083681507))
		((2012-02-03,20:11:56,SampleClass3,[TRACE],verbose detail for id 1718828806))
		((2012-02-03,20:11:56,SampleClass3,[INFO],everything normal for id 530537821))

###Windows PowerShell

These steps use Windows Azure PowerShell. If this is not already installed and configured on your development machine, see [How to install and configure Azure PowerShell](http://azure.microsoft.com/documentation/articles/install-configure-powershell/) before using the following steps.

1. Use the Python examples [streaming.py](#streamingpy) and [jython.py](#jythonpy) to create local copies of the files on your development machine.

2. Use  the following Windows PowerShell script to upload the **streaming.py** and **jython.py** files to the server. Substitute the name of your Azure HDInsight cluster and the path to the **streaming.py** and **jython.py** files in the first three lines of the script.

		$clusterName = YourHDIClusterName
		$pathToStreamingFile = "C:\path\to\streaming.py"
		$pathToJythonFile = "C:\path\to\jython.py"

		$hdiStore = get-azurehdinsightcluster -name $clusterName
		$storageAccountName = $hdiStore.DefaultStorageAccount.StorageAccountName.Split(".",2)[0]
		$storageAccountKey = $hdiStore.defaultstorageaccount.storageaccountkey
		$defaultContainer = $hdiStore.DefaultStorageAccount.StorageContainerName
		
		$destContext = new-azurestoragecontext -storageaccountname $storageAccountName -storageaccountkey $storageAccountKey
		set-azurestorageblobcontent -file $pathToStreamingFile -Container $defaultContainer -Blob "streaming.py" -context $destContext
		set-azurestorageblobcontent -file $pathToJythonFile -Container $defaultContainer -Blob "jython.py" -context $destContext

	This script retrieves information for your HDInsight cluster, then it extracts the account and the key for the default storage account and uploads the files to the root of the container.

	> [AZURE.NOTE] Other methods of uploading the scripts can be found in the [Upload data for Hadoop jobs in HDInsight](/en-us/documentation/articles/hdinsight-upload-data/) document.

After you upload the files, use the following Windows PowerShell scripts to start the job. When the job completes, the output should be written to the Windows PowerShell console.

####For Hive
    
    # Replace 'YourHDIClusterName' with the name of your cluster
	$clusterName = YourHDIClusterName

	$HiveQuery = "add file wasb:///streaming.py;" +
	             "SELECT TRANSFORM (clientid, devicemake, devicemodel) " +
	               "USING 'D:\Python27\python.exe streaming.py' AS " +
	               "(clientid string, phoneLabel string, phoneHash string) " +
	             "FROM hivesampletable " +
	             "ORDER BY clientid LIMIT 50;"
	
	$jobDefinition = New-AzureHDInsightHiveJobDefinition -Query $HiveQuery -StatusFolder '/hivepython'
	
	$job = Start-AzureHDInsightJob -Cluster $clusterName -JobDefinition $jobDefinition
	Write-Host "Wait for the Hive job to complete ..." -ForegroundColor Green
	Wait-AzureHDInsightJob -Job $job
    # Uncomment the following to see stderr output
    # Get-AzureHDInsightJobOutput -StandardError -JobId $job.JobId -Cluster $clusterName
	Write-Host "Display the standard output ..." -ForegroundColor Green
	Get-AzureHDInsightJobOutput -Cluster $clusterName -JobId $job.JobId -StandardOutput

The output for the **Hive** job should appear similar to the following:

	100041	RIM 9650	d476f3687700442549a83fac4560c51c
	100041	RIM 9650	d476f3687700442549a83fac4560c51c
	100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9
	100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9
	100042	Apple iPhone 4.2.x	375ad9a0ddc4351536804f1d5d0ea9b9

####For Pig

	# Replace 'YourHDIClusterName' with the name of your cluster
	$clusterName = YourHDIClusterName

	$PigQuery = "Register wasb:///jython.py using jython as myfuncs;" +
	            "LOGS = LOAD 'wasb:///example/data/sample.log' as (LINE:chararray);" +
	            "LOG = FILTER LOGS by LINE is not null;" +
	            "DETAILS = foreach LOG generate myfuncs.create_structure(LINE);" +
	            "DUMP DETAILS;"
	
	$jobDefinition = New-AzureHDInsightPigJobDefinition -Query $PigQuery -StatusFolder '/pigpython'
	
	$job = Start-AzureHDInsightJob -Cluster $clusterName -JobDefinition $jobDefinition
	Write-Host "Wait for the Pig job to complete ..." -ForegroundColor Green
	Wait-AzureHDInsightJob -Job $job
    # Uncomment the following to see stderr output
    # Get-AzureHDInsightJobOutput -StandardError -JobId $job.JobId -Cluster $clusterName
	Write-Host "Display the standard output ..." -ForegroundColor Green
	Get-AzureHDInsightJobOutput -Cluster $clusterName -JobId $job.JobId -StandardOutput

The output for the **Pig** job should appear similar to the following:

	((2012-02-03,20:11:56,SampleClass5,[TRACE],verbose detail for id 990982084))
	((2012-02-03,20:11:56,SampleClass7,[TRACE],verbose detail for id 1560323914))
	((2012-02-03,20:11:56,SampleClass8,[DEBUG],detail for id 2083681507))
	((2012-02-03,20:11:56,SampleClass3,[TRACE],verbose detail for id 1718828806))
	((2012-02-03,20:11:56,SampleClass3,[INFO],everything normal for id 530537821))

##<a name="troubleshooting"></a>Troubleshooting

Both of the Windows PowerShell script examples contain a commented line that will display error output for the job. If you are not seeing the expected output for the job, uncomment the following line and see if the error information indicates a problem:

	# Get-AzureHDInsightJobOutput -StandardError -JobId $job.JobId -Cluster $clusterName

The error information (STDERR) and the result of the job (STDOUT) are also logged to the default blob container for your clusters at the following locations:

<table>
<tr>
<td>For this job...</td><td>Look at these files in the blob container</td>
</tr>
<td>Hive</td><td>/HivePython/stderr</br>/HivePython/stdout</td>
</tr>
<td>Pig</td><td>/PigPython/stderr</br>/PigPython/stdout</td>
</tr>
</table>

##<a name="next"></a>Next steps

If you need to load Python modules that aren't provided by default, see [How to deploy a module to Azure HDInsight](http://blogs.msdn.com/b/benjguin/archive/2014/03/03/how-to-deploy-a-python-module-to-windows-azure-hdinsight.aspx) for an example.

For other ways to use Pig and Hive, and to learn about using MapReduce, see the following resources:

* [Use Hive with HDInsight](../hdinsight-use-hive)

* [Use Pig with HDInsight](../hdinsight-use-pig)

* [Use MapReduce with HDInsight](../hdinsight-use-mapreduce)
