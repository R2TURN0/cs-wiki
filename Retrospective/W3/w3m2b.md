W3M2b - Understanding of Hadoop Configuration Files

학습 목표

*** This is a 2nd part of W3M2.
The objective of this homework project is to demonstrate your understanding of setting up and configuring an Apache Hadoop multi-node cluster using Docker. You will use at least two Docker containers and configure core-site.xml, hdfs-site.xml, mapred-site.xml, and yarn-site.xml with important parameters.
사전지식

Configuration Files:

core-site.xml

fs.defaultFS: Specifies the default file system URI.
hadoop.tmp.dir: Specifies the temporary directory.
io.file.buffer.size: Specifies the buffer size for reading/writing files.
hdfs-site.xml

dfs.replication: Defines the default replication factor for HDFS.
dfs.blocksize: Specifies the default block size.
dfs.namenode.name.dir: Specifies the path on the local filesystem where the NameNode stores the namespace and transaction logs.
mapred-site.xml

mapreduce.framework.name: Specifies the framework name for MapReduce.
mapreduce.jobhistory.address: Specifies the address of the JobHistoryServer used to access information about completed MapReduce jobs.
mapreduce.task.io.sort.mb: Specifies the amount of memory to use while sorting map output.
yarn-site.xml

yarn.resourcemanager.address: The address of the ResourceManager IPC.
yarn.nodemanager.resource.memory-mb: Determines the amount of memory available to YARN.
yarn.scheduler.minimum-allocation-mb: Specifies the minimum allocation for every container request at the ResourceManager.
기능요구사항

Modifying Configuration Settings

For each configuration file, change the identified settings to the specified values. Ensure that you follow the correct XML structure and syntax.
core-site.xml

Change fs.defaultFS to hdfs://namenode:9000.
Change hadoop.tmp.dir to /hadoop/tmp.
Change io.file.buffer.size to 131072.
hdfs-site.xml

Change dfs.replication to 2.
Change dfs.blocksize to 134217728 (128 MB).
Change dfs.namenode.name.dir to /hadoop/dfs/name.
mapred-site.xml

Change mapreduce.framework.name to yarn.
Change mapreduce.jobhistory.address to namenode:10020.
Change mapreduce.task.io.sort.mb to 256.
yarn-site.xml

Change yarn.resourcemanager.address to namenode:8032.
Change yarn.nodemanager.resource.memory-mb to 8192.
Change yarn.scheduler.minimum-allocation-mb to 1024.
Configuration Modification Script:

The script should accept the path to the Hadoop configuration directory as an argument.
It should back up the original configuration files before making any changes.
The script should modify the specified settings in the XML files to the given values.
It should handle any errors gracefully and report the status of each change.
Backup the original configuration files.
Modify the specified settings in the configuration files.
Restart the Hadoop services.
Verification Script:

The script should confirm the default file system name is set correctly by running a relevant Hadoop command and parsing the output.
It should create a test file in HDFS and check its replication factor to verify the change.
The script should run a simple MapReduce job and ensure it uses the YARN framework.
It should query YARN ResourceManager to verify the total available memory for YARN.
The script should verify the temporary directory, buffer size, block size, NameNode directory, JobTracker, sort memory buffer size, ResourceManager hostname, NodeManager memory allocation, and container minimum allocation.
Query the Hadoop configuration to check the modified settings.
Print the results, indicating whether each setting matches the expected value.
Create a test file in HDFS and verify the replication factor.
프로그래밍 요구사항

Write a shell script or a Python program to modify the specified settings in the configuration files.
Develop another script or program to verify the configuration changes.
Use appropriate commands and APIs to interact with the Hadoop cluster.
예상결과 및 동작예시

Example Expected Outcome:

Configuration Modification Script

Backing up core-site.xml...
Modifying core-site.xml...
Backing up hdfs-site.xml...
Modifying hdfs-site.xml...
Backing up mapred-site.xml...
Modifying mapred-site.xml...
Backing up yarn-site.xml...
Modifying yarn-site.xml...
Stopping Hadoop DFS...
Stopping YARN...
Starting Hadoop DFS...
Starting YARN...
Configuration changes applied and services restarted.
Verification Script

PASS: ['hdfs', 'getconf', '-confKey', 'fs.defaultFS'] -> hdfs://namenode:9000
PASS: ['hdfs', 'getconf', '-confKey', 'hadoop.tmp.dir'] -> /hadoop/tmp
PASS: ['hdfs', 'getconf', '-confKey', 'io.file.buffer.size'] -> 131072
PASS: ['hdfs', 'getconf', '-confKey', 'dfs.replication'] -> 2
PASS: ['hdfs', 'getconf', '-confKey', 'dfs.blocksize'] -> 134217728
PASS: ['hdfs', 'getconf', '-confKey', 'dfs.namenode.name.dir'] -> /hadoop/dfs/name
PASS: ['hadoop', 'getconf', '-confKey', 'mapreduce.framework.name'] -> yarn
PASS: ['hadoop', 'getconf', '-confKey', 'mapreduce.job.tracker'] -> namenode:9001
PASS: ['hadoop', 'getconf', '-confKey', 'mapreduce.task.io.sort.mb'] -> 256
PASS: ['yarn', 'getconf', '-confKey', 'yarn.resourcemanager.address'] -> namenode:8032
PASS: ['yarn', 'getconf', '-confKey', 'yarn.nodemanager.resource.memory-mb'] -> 8192
PASS: ['yarn', 'getconf', '-confKey', 'yarn.scheduler.minimum-allocation-mb'] -> 1024
PASS: Replication factor is 2
Each PASS line indicates that the setting matches the expected value.
If a setting does not match, it would print FAIL with the actual value, like this:
FAIL: ['hdfs', 'getconf', '-confKey', 'fs.defaultFS'] -> hdfs://namenode:8020 (expected hdfs://namenode:9000)
Submission:

Submit 2 scripts or Python programs
Provide the original and the changed configuration files (core-site.xml, hdfs-site.xml, mapred-site.xml, yarn-site.xml).
Include all the files to run testing scripts if necessary.
Provide a README file with detailed instructions for setting up and running the scripts or programs
팀 활동 요구사항

4개의 xml files들을 살펴 보고 각 화일의 셋팅 중에 중요하거나 유용하다고 생각되는 것들을 각자 골라서 용법을 파악해 보세요. 그런 다음 팀과 함께 토의한 다음, 위키에 정리합시다.