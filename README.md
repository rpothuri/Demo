# Docker Container running Spark Jobserver on Mesos
Environment
Ubuntu 16.04
Spark 1.6.2 
Vanilla Spark Job server 
Mesos 1.1.0
For Docker installation on Ubuntu, please refer to the following link.
All the required services for running the SJS on mesos are in this single container. 
Below is the Dockerfile based on Ubuntu Image
FROM ubuntu:latest
MAINTAINER rohit <rpothuri>
 
# Overall ENV vars
ENV SBT_VERSION 0.13.11
ENV SCALA_VERSION 2.10.5
ENV SPARK_VERSION 1.6.2
ENV SPARK_JOBSERVER_BRANCH joshua
ENV SPARK_JOBSERVER_BUILD /spark-jobserver-build
ENV SPARK_JOBSERVER_BUILD_HOME /spark-jobserver-build/spark-jobserver
ENV SPARK_MASTER_IP=127.0.0.1
ENV SPARK_LOCAL_IP=127.0.0.1
ENV MESOS_NATIVE_JAVA_LIBRARY /usr/local/lib/libmesos.so
ENV SPARK_EXECUTOR_URI /usr/local/spark/spark-1.6.2-bin-hadoop2.6.tgz
ENV MESOS_BUILD_DIR /mesos-1.1.0/build
ENV SPARK_HOME /usr/local/spark
ENV SPARK_VERSION_STRING spark-$SPARK_VERSION-bin-hadoop2.6
ENV SPARK_DOWNLOAD_URL http://d3kbcqa49mib13.cloudfront.net/$SPARK_VERSION_STRING.tgz
 
#Dependencies for SJS and Mesos
RUN apt-get update && apt-get install -yq --no-install-recommends --force-yes \
wget \
git \
curl \
openjdk-8-jdk \
libsvn-dev \
zlib1g-dev \
libapr1-dev \
maven \
vim \
libsasl2-modules \
libsasl2-dev \
libcurl4-nss-dev \
python-boto \
python-dev \
build-essential && \
rm -rf /var/lib/apt/lists/*
 
# Download Mesos package
RUN wget http://www.apache.org/dist/mesos/1.1.0/mesos-1.1.0.tar.gz && \
   tar -zxf mesos-1.1.0.tar.gz
 
# This step will take forever to complete (close to 1.5 hours)  
# Build
RUN cd mesos-1.1.0 && mkdir build && cd build && ../configure && make -j 4 V=0
 
WORKDIR $MESOS_BUILD_DIR
RUN make install && \
mkdir /var/lib/mesos
EXPOSE 5050
 
 
WORKDIR /
 
# SBT install
RUN wget https://dl.bintray.com/sbt/debian/sbt-$SBT_VERSION.deb --no-check-certificate && \
dpkg -i sbt-$SBT_VERSION.deb && \
rm sbt-$SBT_VERSION.deb
 
# Download and unzip Spark
RUN wget $SPARK_DOWNLOAD_URL && \
    mkdir -p /usr/local/spark && \
    tar xvf $SPARK_VERSION_STRING.tgz -C /tmp && \
    cp -rf /tmp/$SPARK_VERSION_STRING/* /usr/local/spark/ && \
    cp spark-$SPARK_VERSION-bin-hadoop2.6.tgz /usr/local/spark/ && \
    rm -rf -- /tmp/$SPARK_VERSION_STRING && \
    rm spark-$SPARK_VERSION-bin-hadoop2.6.tgz
 
#Add the spark-env.sh file with setting the mesos ENV variables
RUN mv /usr/local/spark/conf/spark-env.sh.template /usr/local/spark/conf/spark-env.sh
ADD spark-env.sh /usr/local/spark/conf/spark-env.sh
 
# Clone Spark-Jobserver repository
RUN mkdir -p $SPARK_JOBSERVER_BUILD
WORKDIR $SPARK_JOBSERVER_BUILD
RUN git clone https://github.com/spark-jobserver/spark-jobserver.git
 
# Build Spark-Jobserver
WORKDIR $SPARK_JOBSERVER_BUILD_HOME
ADD settings.sh config/settings.sh
ADD settings.conf config/settings.conf   
RUN bin/server_package.sh settings
 
# Cleanup files, folders and variables
RUN unset SPARK_VERSION_STRING && \
    unset SPARK_DOWNLOAD_URL && \
    unset SPARK_JOBSERVER_BRANCH && \
    unset SPARK_JOBSERVER_BUILD_HOME && \
    unset SBT_VERSION && \
    unset SCALA_VERSION && \
    unset SPARK_VERSION && \
    unset SPARK_JOBSERVER_BUILD
 
     
EXPOSE 8090 9999
  
#Commands to run after starting up the container
#CMD ./bin/mesos-master.sh --ip=127.0.0.1 --work_dir=/var/lib/mesos
#CMD ./bin/mesos-agent.sh --master=127.0.0.1:5050 --launcher=posix --no-switch_user --no-systemd_enable_support --work_dir=/var/lib/mesos
#RUN /tmp/job-server/server_start.sh
 
Running SJS on Mesos
On the Machine. Open terminal and login as root user. create a directory (say sjsmesos)
mkdir sjsmesos
Navigate into that directory and create Dockerfile.
cd sjsmesos
touch Dockerfile
Copy the dockerfile code above into the Dockerfile created. 
Copy the following files into the sjsmesos directory.(settings.conf, settings.sh, spark-env.sh)
Once the files are copied, use the below command to create a docker image.
docker build -t bdf/sjsmesos:v1 .
Building Mesos takes a lot of time (~ 1.5 hrs)
Once the image is built successfully, run the following command to spin up the docker container. This gives the bash shell for the container.
#For running on local Mode
docker run -it -p 8090:8090 bdf/sjsmesos:v1 bash
 
#For running on Mesos
docker run -it -e SPARK_MASTER=mesos://localhost:5050 -p 8090:8090 bdf/sjsmesos:v1 bash
Once we login to the container bash shell, run the following command to start the Mesos Master service
cd mesos-1.1.0/build
./bin/mesos-master.sh --ip=127.0.0.1 --work_dir=/var/lib/mesos
 
Open an other terminal on the Ubuntu Machine. Get the name of the container that is running. Once we get the name of our sjsmesos container, open an other instance of the same container to run the Mesos Slave service.
# In the terminal
docker ps -l
# note down the container name from result of the above command. Run the following command to login to the running container.
docker exec -i -t <container name> /bin/bash
#Once in the container bash shell, run the following commands to start the Mesos slave service
cd mesos-1.1.0/build
./bin/mesos-agent.sh --master=127.0.0.1:5050 --launcher=posix --no-switch_user --no-systemd_enable_support --work_dir=/var/lib/mesos
Open an other terminal and repeat the same procedure as above to login into the running container. Once we are in the bash shell of the running container, use the following commands to start the spark job server
/tmp/job-server/server_start.sh
# This will start the SJS and we can access the web interface using the localhost:8090 address
To test the SJS we can use the examples provided in the other documentations. Note: Use the timeout settings mentioned in the troubleshooting section below while running the jobs.

########################################################################################################################################

Troubleshooting
1) Error Message => "message": "Ask timed out on [Actor[akka://JobServer/user/context-supervisor/sql-context#786713198]] after [20000 ms]",
Make sure that the following environment variables are set properly in the spark-env.sh file in the spark home directory (<SPARK_HOME>/conf) and the respective files exist
1.     MESOS_NATIVE_JAVA_LIBRARY=<path to libmesos.so>
2.     SPARK_EXECUTOR_URI=<URL of spark-1.6.2.tar.gz>. This has to be set (in spark-defaults.conf) too 
 
         Also Adjust the Spray's default request timeout and idle timeout to values higher in the conf file passed while starting up the Spark job server (In our case it is settings.conf file present in the /tmp/job-server/ directory)
      In case of network timeout errors, change the maximum-frame-size configuration setting in the settings.conf file to higher value (~100MiB)
Changes in Settings.conf file
#Timeout setting changes
spray.can.server {
  idle-timeout = 210 s
  request-timeout = 200 s
}
  
#Network timeout changes (only maximum-frame-size setting to be changed as shown below)
akka {
  # Use SLF4J/logback for deployed environment logging
  loggers = ["akka.event.slf4j.Slf4jLogger"]
 
  actor {
    provider = "akka.cluster.ClusterActorRefProvider"
    warn-about-java-serializer-usage = off
  }
 
  remote {
    log-remote-lifecycle-events = off
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
      send-buffer-size = 512000b
      receive-buffer-size = 512000b
      # This controls the maximum message size, including job results, that can be sent
      maximum-frame-size = 100 MiB
    }
  }
 
         Also add the timeout parameter along with the request. Eg shown below
curl -d "" '127.0.0.1:8090/jobs?appName=sql&classPath=spark.jobserver.SqlLoaderJob&context=sql-context&sync=true&timeout=20'
 
2) Mesos Slave Service startup issue - systemd does not boot in a container
Add the following conf settings while starting up the Mesos Slave service

./bin/mesos-agent.sh --master=127.0.0.1:5050 --launcher=posix --no-switch_user --no-systemd_enable_support --work_dir=/var/lib/mesos


*************************************************************************************************************************************

Building/Developing Joshua Spark Job Server on Cloudera VM

Prerequisites
Oracle VirtualBox
Cloudera Quickstart VM zip file - http://www.cloudera.com/downloads/quickstart_vms/5-8.html Select VirtualBox and then Download and unzip.
Install and Configure Cloudera Quickstart VM
These instructions describe how to install the VM and create and mount a shared folder containing the project source. The project source resides on the host machine. An alternative is to checkout the source on the guest VM, but using an IDE on the host is a much better experience.
Launch the VirtualBox client and create a new VM. 
Choose expert mode, select a type of Linux and a Version of Red Hat (64-bit).
Provide as much memory as possible, at least 16GB if possible.
Select "Use an existing virtual hard disk file" and select the *.vmdk file downloaded above. Click OK.
Right-click on the VM and select Settings.. Click "Shared Folders" and select a host folder that contains or will contain the project source, e.g. "/home/foo/prj/com.rxcorp.bdf.spark-jobserver". Enter a folder/share name (e.g. "joshua" and click auto-mount.
Add the Guest Additions to the VM.
From a terminal window in the VM
sudo yum groupinstall "Development Tools"
From the terminal window in the VM
sudo yum install kernel-devel
From the VirtuBox Devices menu choose "Insert Guest Additions CD image...". You will be prompted to download the disk image file. 
Click Insert when prompted with "Do you wish to register this disk image file and insert it into the virtual optical drive?"
Click OK and then Run when prompted to autorun the CD image. When prompted for the root password enter "cloudera". The script to install the Guest Additions will run.
Restart the guest.
Mount the shared folder.
Create the mount directory somewhere, e.g. /home/cloudera/prj/joshua 
"sudo mount -t vboxsf -o rw,uid=501,gid=501 joshua /home/cloudera/prj/joshua/" The first joshua is the share name from step 1d, the uid and gid correspond to the cloudera user and group.
Install Cloudera Manager - Click "Launch Cloudera Express" on the desktop. 
Parcelize the cluster. 
Login to Cloudera Magaer at http://quickstart.cloudera:7180 cloudera/cloudera. 
Go to Hosts/Parcels and click Download for the CDH 5 parcel.
Click Distribute for the CDH 5 parcel.
Click Activate for the CDH 5 parcel.
Start services. From the main Cloudera Manager page.
Start HDFS.
Start Hive.
Start Zookeeper
Start YARN
Remove the packages that are no longer needed now because we have the parcels. This will also get rid of a health warning (shouldn't have packages and parcels at the same time). Run "sudo yum remove foo" where foo is one of: crunch, flume-ng, hadoop, hadoop-0.20-mapreduce, hadoop-hdfs, hadoop-httpfs, hadoop-kms, hadoop-mapreduce, hadoop-yarn, hbase, hbase-solr, hive, hive-hcatalog, hue, hue-common, impala, llama, oozie, parquet, pig, sentry, solr, spark, sqoop, sqoop2, zookeeper
 
VM Guest Prerequisites
git (included in VM)
curl (included in VM)

Install sbt
curl https://bintray.com/sbt/rpm/rpm | sudo tee /etc/yum.repos.d/bintray-sbt-rpm.repo
sudo yum install sbt
 
joshua JobServer Installation & Configuration
Get Started
#----------- Skip -----------
# This first commented out bit can be used to install the vanilla jobserver
# create a tmp directory for build spark-jobserver 
# mkdir /tmp/spark-jobserver-build   
# navigate to the above directory 
# cd /tmp/spark-jobserver-build  
# clone the latest jobserver repository
# git clone https://github.com/spark-jobserver/spark-jobserver.git    
# After git clone, navigate to the spark-jobserver directory
# cd spark-jobserver
#----------- Skip -----------
 
# hive-site.xml must be on the classpath for the HiveContext to pick it up and be able to hit the CDH hive metastore and data.
# For now we put it in job-server-extras/src/main/resources but that needs to change.
 
# <PROJECT_DIR> represents the dir on the VM guest which is the root of the project
     
# cd <PROJECT_DIR>
# Create a config directory in bin folder
 
mkdir ./bin/config
    
# Copy settings template file from config to bin/config folder
 
cp ./job-server/config/local.sh.template bin/config/settings.sh   
     
# Make the following changes in the settings.sh file -- gedit bin/config/settings.sh
 
LOG_DIR=/var/log/job-server
SPARK_HOME=/opt/cloudera/parcels/CDH/lib/spark
   
# Copy the conf file from below location to bin/config location
 
cp ./job-server/src/main/resources/application.conf bin/config/settings.conf
    
# Edit the conf file ./bin/config/settings.conf, make sure of the following setting:
# We want to run with context-per-jvm=true but in development mode (running from sbt) it only works if context-per-jvm=false 
context-per-jvm = false
# Run sbt command in <PROJECT_ROOT>
 
sbt
  
# Type the below command in the sbt shell to simply test the Job Server
job-server/reStart
  
# If instead you want to use the HiveContext or want to test the JDBC Server use:
job-server-extras/reStart config/joshua-devl.conf
 
# Access http://localhost:8090 to see if the Spark job server UI is enabled
Testing the Job server
 
# In the shell, navigate to below location (login as root in case of permission issues)
 
cd <PROJECT_ROOT>/spark-jobserver/
 
# Run the below command to package job-server-tests
  
sbt job-server-tests/package
  
# It will give you a jar in job-server-tests/target/scala-2.10 directory
# Upload the jar to the server using the following curl command
  
curl --data-binary @job-server-tests/target/scala-2.10/job-server-tests_2.10-0.7.0-SNAPSHOT.jar localhost:8090/jars/test
  
# Submit the word count job on the server using the below command
 
curl -d "input.string = a b c a b see" 'localhost:8090/jobs?appName=test&classPath=spark.jobserver.WordCountExample'
  
# The status should be ‘STARTED’ when submitted
# Run  curl localhost:8090/jobs/<jobID> to get the output
[cloudera@quickstart spark-jobserver]$ curl localhost:8090/jobs/f148728f-8d94-41e-8e8f-a0dcc6d8b8c8
 
 
# The output should be something like this
 
{
  "duration": "1.7 secs",
  "classPath": "spark.jobserver.WordCountExample",
  "startTime": "2016-10-26T12:04:24.608-07:00",
  "context": "13d7fb1c-spark.jobserver.WordCountExample",
  "result": {
    "a": 2,
    "b": 2,
    "see": 1,
    "c": 1
  },
  "status": "FINISHED",
  "jobId": "f148728f-8d94-421e-8e8f-a0dcc6d8b8c8"
  
#------------------------------------------------------------------------------------------------------------------------------------
# Testing SQL Context
  
# Navigate to the spark-jobserver directory
  
cd <PROJECT_ROOT>
  
# run sbt command
  
sbt
  
# In sbt shell run the below command
  
job-server-extras/reStart
  
# This will start up the Job server with the job-server-extras jar packaged
 
  
  
#In a new shell run the below command to create an SQL context
  
curl -d "" '127.0.0.1:8090/contexts/sql-context?context-factory=spark.jobserver.context.SQLContextFactory'
  
# Upload the jar to the server
  
curl --data-binary @job-server-extras/target/scala-2.10/job-server-extras_2.10-0.7.0-SNAPSHOT.jar  localhost:8090/jars/sql
  
# Once the jar is uploaded, the following commands can be used to test
  
curl -d "" '127.0.0.1:8090/jobs?appName=sql&classPath=spark.jobserver.SqlLoaderJob&context=sql-context&sync=true'
  
  
curl -d "sql = \"select * from addresses limit 10\"" '127.0.0.1:8090/jobs?appName=sql&classPath=spark.jobserver.SqlTestJob&context=sql-context&sync=true'
 
# The output will be some thing like below
  
{
  "jobId": "b03c202f-ef77-44d0-8de0-cb1998907ca1",
  "result": ["[Bob,Charles,101 A St.,San Jose]", "[Sandy,Charles,10200 Ranch Rd.,Purple City]", "[Randy,Charles,101 A St.,San Jose]"]
}
  
References
http://gethue.com/get-started-with-spark-deploy-spark-server-and-compute-pi-from-your-web-browser/
https://community.cloudera.com/t5/Hadoop-101-Training-Quickstart/spark-jobserver-and-spark-jobs-in-Hue-on-CDH5-2/td-p/22397
http://nishutayaltech.blogspot.com/2016/05/how-to-run-spark-job-server-and-spark.html
https://github.com/spark-jobserver/spark-jobserver/blob/master/doc/contexts.md#example


************************************************************************************************************************************
Sparklez Spark Thrift Server

Description
Installation & Configuration
Custom CDH Spark
Custom CDH Spark Build
Custom CDH Spark Installation and Configuration
Sparklez
Spaklez Thrift Server Installation
Starting the Sparklez Thrift Server
Connecting to Sparklez Thrift Server
Note this project has been abandoned, This documentation is retained for reference.
Description
Sparklez is an implementation of the Spark Thrift Server. Instead of the normal Spark Thrift Server, Sparklez creates its own HiveContext and starts the normal HiveThriftServer2 using this context. This HiveContext contains UDFs (User Defined Functions) and UDAFs (User Defined Aggregate Functions) which can execute custom code, provide caching, etc.. Normal JDBC clients can connect to the Sparkez server and make use of the custom UDFs and UDAFs.
Installation & Configuration
Custom CDH Spark
Custom CDH Spark Build
Cloudera CDH 5.7.1, the latest development version as of this writing, does not ship with the Spark Thrift Server therefore we need to build a custom version of the CDH version of Spark that includes the Spark Thrift Server. Cloudera may include the Spark Thrift Server in future releases.
To build a custom version of CDH Spark follow these instructions:  http://blog.clairvoyantsoft.com/2016/06/how-to-rebuild-clouderas-spark/ or see them below (copied from the page).
 Note that this was successfully done on a Linux laptop, but not on an IMS Windows laptop.
(copied from  http://blog.clairvoyantsoft.com/2016/06/how-to-rebuild-clouderas-spark/ )
Make sure that you have the following software installed on your local workstation:
Virtual Box
Vagrant 1.3+
Git
vagrant-librarian-puppet
vagrant-puppet-install (optional)
Installation of these components are documented at their respective links.
Get Started
Clone the vagrant-sparkbuilder git repository to your local workstation:
git clone https://github.com/teamclairvoyant/vagrant-sparkbuilder.git
cd vagrant-sparkbuilder
Start the Vagrant instance that comes with the vagrant-sparkbuilder repository. This will boot a CentOS 7 virtual machine, install the Puppet agent, and instruct Puppet to configure the virtual machine with Oracle Java and the Cloudera Spark git repository. Then it will log you in to the virtual machine.
vagrant up
vagrant ssh
Inside the virtual machine, change to the spark directory:
cd spark 
The automation has already checked out the branch/tag that corresponds to the target CDH version (presently defaulting to cdh5.7.0-release). Now you just need to build Spark with the Hive Thriftserver while excluding dependencies that are shipped as part of CDH. The key options in this example are the -Phive -Phive-thriftserver. Expect the compilation to take 20-30 minutes depending upon your Internet speed and workstation CPU and disk speed.
patch -p0 </vagrant/undelete.patch
./make-distribution.sh -DskipTests \
  -Dhadoop.version=2.6.0-cdh5.7.0 \
  -Phadoop-2.6 \
  -Pyarn \
  -Phive -Phive-thriftserver \
  -Pflume-provided \
  -Phadoop-provided \
  -Phbase-provided \
  -Phive-provided \
  -Pparquet-provided
If the above command fails with a ‘Cannot allocate memory’ error, either run it again or increase the amount of memory in the Vagrantfile.
Copy the resulting distribution back to your local workstation:
rsync -a dist/ /vagrant/dist-cdh5.7.0-nodeps
If you want to build against a different CDH release, then use git to change the code:
git checkout -- make-distribution.sh
git checkout cdh5.5.2-release
patch -p0 </vagrant/undelete.patch
Log out of the virtual machine with the exit command, then stop and/or destroy the virtual machine:
vagrant halt
vagrant destroy
 

Custom CDH Spark Installation and Configuration
Copy (sftp/scp) the spark distribution under dist/ to somewhere on the cluster edge node. E.g. /development/pbi/data/esnyder/spark-dist-cdh5.7.1-nodeps 
Copy the following from the official CDH Spark version to spark-dist-cdh5.7.1-nodeps/conf:  
/opt/cloudera/parcels/CDH/lib/spark/conf/classpath.txt
/opt/cloudera/parcels/CDH/lib/spark/conf/log4j.properties
/opt/cloudera/parcels/CDH/lib/spark/conf/spark-env.sh
Edit the copied spark-env.sh to match the attached spark-env.sh. The changes are to SPARK_HOME, HADOOP_CONF_DIR and SPARK_DIST_CLASSPATH.
Sparklez
Spaklez Thrift Server Installation
Checkout the project from gitblt. The remote repo is (change esnyder to your id): ssh://esnyder@gitblit.rxcorp.com:29418/USSupplierservices/com.rxcorp.bdf.sparklez.git
cd to sparklez-thrift-server and build with "mvn clean package". This assumes that maven is installed and configured.
Make a directory on the CDH cluster edge node, e.g. /development/pbi/data/esnyder/sparklez-thrift-server
Copy (sftp/scp) the jar file under /target to this new directory.
Copy devl.properties from the project root to this new directory. devl.properties contains spark configuration properties as well as the properties that specify the kerberos principal and keytab path under which Sparklez Thrift Server will run (probably the hive service principal and keytab). Note that the hive principal specified in devl.properties must be a 3-part service principal (principal/host@domain), e.g. "sparklez.hive.server2.authentication.kerberos.principal=hive/cdts10hdbe02d.rxcorp.com@INTERNAL.IMSGLOBAL.COM" and the host must be the host name of the edge node under which Sparklez Thrift Server will run.
Obtain hive service and pbidusr keytabs and copy them to the new directory. pbidusr.keytab and hive.keytab should exist in the directory. The hive service keytab must match the principal specified in devl.properties.
Make a copy of pbidusr.keytab somewhere locally and name it impala.user.keytab.
Create an HDFS dir /user/pbidusr/sparklez and upload to it impala.properties and impala.user.keytab. These files are used by UDTFs that connect to Impala. Verify that the url and principal properties are valid (see below).
Copy run.sh from the root of the project to the new directory and make it executable "chmod +x run.sh".

devl.properties
#spark.driver.cores=8
spark.driver.memory=16g
spark.dynamicAllocation.enabled=false
#spark.dynamicAllocation.minExecutors=1
#spark.dynamicAllocation.maxExecutors=24
spark.executor.cores=8
spark.executor.instances=4
# Always update spark.yarn.executor.memoryOverhead (below) whenever this value is changed.
spark.executor.memory=16g
# Kryo is the recommended alternative to the default of Java serialization.
spark.serializer=org.apache.spark.serializer.KryoSerializer
# If using dynamic allocation (spark.dynamicAllocation.enabled), set this to true.
spark.shuffle.service.enabled=false
 
# This is the kerberos principal and keytab which the thrift server will use to authenticate.
# The keytab must be a 3 part host based service principal, e.g. hive/somehost.rxcorp.com@INTERNAL.IMSGLOBAL.COM.
sparklez.hive.server2.authentication.kerberos.principal=hive/cdts10hdbe02d.rxcorp.com@INTERNAL.IMSGLOBAL.COM
sparklez.hive.server2.authentication.kerberos.keytab=/home/pbidusr/sparklez/sparklez-thrift-server/hive.keytab
impala.properties
impala.url=jdbc:impala://cdts1hdpun01d.rxcorp.com:21051;AuthMech=1;KrbRealm=INTERNAL.IMSGLOBAL.COM;KrbHostFQDN=cdts1hdpun01d.rxcorp.com;KrbServiceName=impala;SSL=1;AllowSelfSignedCerts=1;CAIssuedCertNamesMismatch=1
impala.principal=pbidusr
Starting the Sparklez Thrift Server
Execute ./run.sh in the Sparklez thrift server dir on the edge node. This will write stdout and stderr to nohup.out. Note that log rotation is not in place.
This will create a Spark job that runs indefinitely and creates a thrift server that listens on the port specified in run.sh.
"ps -ef|grep sparklez" will show the Spark Job / Sparklez Thrift Server.
To kill the Sparklez Thrift Server get the pid with "ps -ef|grep Sparklez" and send a TERM signal to it with "kill <the pid>". This will shut it down both the thrift server and Spark job gracefully.
Connecting to Sparklez Thrift Server
From the beeline client (just execute 'beeline'):
The principal must specify the host on which Sparklez Thrift Server is running.
You will be prompted for a username and password, these can be any values.
Execute any valid Spark SQL/Hive dialect query, including the execution of any of our custom UDTFs.
Connecting Via Beeline
!connect jdbc:hive2://localhost:60010/default;principal=hive/cdts10hdbe02d.rxcorp.com@INTERNAL.IMSGLOBAL.COM
  
Enter username for jdbc:hive2://localhost:60010/default;principal=hive/cdts10hdbe02d.rxcorp.com@INTERNAL.IMSGLOBAL.COM: foo
Enter password for jdbc:hive2://localhost:60010/default;principal=hive/cdts10hdbe02d.rxcorp.com@INTERNAL.IMSGLOBAL.COM: ***
