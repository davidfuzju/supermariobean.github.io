---
layout: post
category: iOS
---

##版本配置##

+ Ubuntu

	系统版本
	
		Ubuntu 12.04.1 LTS (GNU/Linux 3.2.0-37-generic-pae i686)
		
	通过ssh登陆的主机
	
		$ssh username@10.200.187.27
		
	创建并配置用户组
		
		hadoop
		
	创建用户
		
		hadoop
		
	密码
		
		huaathadoop
		
	用户权限
	
		root
		
	修改主机名称
		
		Hadoop-Ubuntu-00
		
+ ssh

	版本
	
		OpenSSH_5.9p1 Debian-5ubuntu1, OpenSSL 1.0.1 14 Mar 2012
		
	需要配置为无需密码登陆本机，将在后文中配置完成
		
+ Hadoop
 
	版本
	
		hadoop_1.0.4-1_i386.deb
	
	关于版本的选择，截止日期为2013.3.7 Amazon EC2支持的Hadoop版本为：
	
		Hadoop 1.0.3 (Amazon Distribution)
		Hadoop 0.20.205 (MapR M5 Edition v1.2.8)
		Hadoop 0.20.205 (MapR M3 Edition v1.2.8)
	
    这里选择的版本为Apache提供的开源hadoop版本1.0.4。

    你可以点击[这里](http://mirror.bjtu.edu.cn/apache/hadoop/common/hadoop-1.0.4/)进入下载列表查看其他版本

+ Mahout
	
	版本
	
		mahout-distribution-0.7.tar.gz
	
	你可以点击[这里](http://mirror.bjtu.edu.cn/apache/mahout/)选择你想要的版本下载
	
+ Java

	版本
	
		java version "1.6.0_24"
		OpenJDK Runtime Environment (IcedTea6 1.11.5) (6b24-1.11.5-0ubuntu1~12.04.1
		OpenJDK Client VM (build 20.0-b12, mixed mode, sharing)
		
			
+ maven

		$ sudo apt-get install maven
		
		$ mvn -v
		Apache Maven 3.0.4
		Maven home: /usr/share/maven
		Java version: 1.6.0_26, vendor: Sun Microsystems Inc.
		Java home: /usr/lib/jvm/java-6-sun-1.6.0.26/jre
		Default locale: en_US, platform encoding: UTF-8
		OS name: "linux", version: "3.2.0-38-generic-pae", arch: "i386", family: "unix"

		
+ GroupLens DataSet

	数据集版本
	
		MovieLens 100k - Consists of 100,000 ratings from 1000 users on 1700 movies.
		MovieLens 1M - Consists of 1 million ratings from 6000 users on 4000 movies.
		MovieLens 10M - Consists of 10 million ratings and 100,000 tag applications applied to 10,000 movies by 72,000 users.

	你可以点击[这里](http://www.grouplens.org/node/12)进入数据集列表并获得你想要的数据集
		
		
##开始部署##

###部署目标###

+ 在ubuntu系统之上搭建单机模式（伪分布式）的hadoop分布式应用程序框架，并在此之上运行mahout推荐引擎，使用来自GroupLens的dataset进行mahout推荐引擎测试。

###部署####
+ Ubuntu

	+ 群组及用户设置
	
		创建hadoop用户组
		
			$ sudo addgroup hadoop
		
		创建hadoop用户
		
			$ sudo adduser -ingroup hadoop hadoop
			
		为新创建的hadoop用户赋予root权限
		
			$ sudo vim /etc/sudoers
			
		找到vim编辑器如下内容
			
			# User privilege specification
			root    ALL=(ALL:ALL) ALL
		
		添加一行命令，是hadoop用户获得root用户同样的权限
		
			hadoop  ALL=(ALL:ALL) ALL
		
		完成后的结果应该如下所示
		
			# User privilege specification
			root    ALL=(ALL:ALL) ALL
			hadoop  ALL=(ALL:ALL) ALL
			
		保存退出即可  
		**若提示文件只读，在vim保存命令加添加!**
		
			:w!
		
	+ 修改主机名称
		
		直接编辑文件
		
			$ sudo vim /etc/hostname
			
		修改主机名为
		
			Hadoop-Ubuntu-00	
	
	+ ssh无密码登陆
		
		安装openssh-server;

			$ sudo apt-get install ssh openssh-server
		
		如果你当前用户就是hadoop用户则不需要切换，如果不是，你需要切换至hadoop用户用以完成所有的hadoop部署
		
			$ su - hadoop
		
		使用rsa方式生成ssh-key
		
			$ ssh-keygen -t rsa -P ""

		**当回应以下输入时**

			Enter file in which to save the key (/home/hadoop/.ssh/id_rsa): 
		**默认回车后会在~/.ssh/下生成两个文件：id_rsa和id_rsa.pub这两个文件是成对出现的**
		
		追加授权文件
		
			$ cd ~/.ssh
			$ cat id_rsa.pub >> authorized_keys
		
		至此，实现无需输入密码ssh登陆本机
		
		可以通过命令
		
			$ ssh localhost
		
		来验证
	
	+ Java环境
		
		一般本版本的Ubuntu系统下Java环境无需安装，安装可由以下命令完成
		
			$ sudo apt-get install openjdk-6-jre
		
		**这里犯了一个低级错误，安装jre是可以保证hadoop的运行，但是当你在后面使用jps这样的开发调试命令的时候就会有问题**
		>提示如下
		>		
		>		hadoop@ubuntu:/usr/lib/jvm/java-6-openjdk$ jps
		>		The program 'jps' can be found in the following packages:
		>		* openjdk-6-jdk
		>		* openjdk-7-jdk
		>		Ask your administrator to install one of them
		>这是由于只安装了上面提示的openjdk-6-jre，也就是java的运行时环境，但是并没有安装jdk从而获取不到开发调试命令的路径，运行下面的命令以获得相应版本的jdk
		>
		>		$ sudo apt-get install openjdk-6-jdk
		**解决**
+ Hadoop

	+ 获取hadoop安装包
		
		我是在自己的Air中下载了hadoop的安装包，然后通过scp命令，将我本地的hadoop安装包通过ssh传输给了远程主机上
		
			$ scp /Users/DavidFu/Desktop/文件名 hadoop@10.200.187.132:~/
			
			
		如果你下载的是源码安装，则如下步骤
		
		hadoop-1.0.4.tar.gz在home目录下，将它复制到安装目录 /usr/local/下

			$ sudo cp hadoop-1.0.4.tar.gz /usr/local/

		解压hadoop-0.20.203.tar.gz；

			$ cd /usr/local
			$ sudo tar -zxf hadoop-1.0.4.tar.gz
			
		将解压出的文件夹改名为hadoop;

			$ sudo mv hadoop-1.0.4 hadoop
			
		将该hadoop文件夹的属主用户设为hadoop，

			$ sudo chown -R hadoop:hadoop hadoop
	
	+ 设置配置文件
			
		打开hadoop/conf/hadoop-env.sh文件;

			$ sudo vim hadoop/conf/hadoop-env.sh
			
		配置conf/hadoop-env.sh（找到#export JAVA_HOME=...,去掉#，然后加上本机jdk的路径）;

			export JAVA_HOME=/usr/lib/jvm/java-6-sun
			
		**如果不知道自己的JAVA_HOME，可以使用如下bash命令**
		
			$ echo $JAVA_HOME	
			
		>>添加HADOOP_HOME
		>>	
		>>		export HADOOP_HOME=/usr/local/hadoop
        >>
        >
        >**此版本的HADOOP_HOME不能通过bash命令修改，所以这条内容废除**
        
        >>添加hadoop可执行文件目录到系统PATH，以便方便的调用
        >>
        >>	    export PATH=$PATH:/usr/local/hadoop/bin
        >>
        >
        >**上面的bash命令无法达到效果修改为如下bash命令可行**
		>
		>		export PATH=/usr/local/hadoop/bin:$PATH
		>
		
		打开conf/core-site.xml文件;

			$ sudo vim hadoop/conf/core-site.xml
		
		编辑如下：

			<?xml version="1.0"?>
			<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
			<!-- Put site-specific property overrides in this file. -->
			<configuration>
				<property>  
  					<name>fs.default.name</name>  
 					<value>hdfs://localhost:9000</value>   
 				</property>  
			</configuration>
			
		打开conf/mapred-site.xml文件;

			$ sudo vim hadoop/conf/mapred-site.xml

		编辑如下：

			<?xml version="1.0"?>
			<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
			<!-- Put site-specific property overrides in this file. -->
    		        <configuration>  
     			        <property>   
      				        <name>mapred.job.tracker</name>  
      				        <value>localhost:9001</value>   
     			        </property>  
    		        </configuration>

		打开conf/hdfs-site.xml文件;

			$ sudo vim hadoop/conf/hdfs-site.xml
		
		编辑如下：

			<configuration>
				<property>
					<name>dfs.name.dir</name>
					<value>/usr/local/hadoop/datalog1,/usr/local/hadoop/datalog2</value>
				</property>
				<property>
					<name>dfs.data.dir</name>
					<value>/usr/local/hadoop/data1,/usr/local/hadoop/data2</value>
				</property>
				<property>
					<name>dfs.replication</name>
					<value>2</value>
				</property>
			</configuration>
			
		打开conf/masters文件，添加作为secondarynamenode的主机名，作为单机版环境，这里只需填写 localhost 就Ok了。

			$ sudo vim hadoop/conf/masters

		打开conf/slaves文件，添加作为slave的主机名，一行一个。作为单机版，这里也只需填写 localhost就Ok了。

			$ sudo vim hadoop/conf/slaves
		
	+ 使用伪分布模式上运行hadoop

		进入hadoop目录下，格式化hdfs文件系统，初次运行hadoop时一定要有该操作，

			$ cd /usr/local/hadoop/
			$ bin/hadoop namenode -format

		出现下面的Log时，就说明hdfs文件系统格式化成功了。
			
			13/03/08 16:21:40 INFO common.Storage: Storage directory /usr/local/hadoop/datalog2 has been successfully formatted.
			13/03/08 16:21:40 INFO namenode.NameNode: SHUTDOWN_MSG: 
			/************************************************************
			SHUTDOWN_MSG: Shutting down NameNode at Hadoop-Ubuntu-00/180.168.41.175
			************************************************************/
			
		**问题**  
		**第一次格式化hdfs文件的时候发生以下问题**
		
				************************************************************/
			13/03/08 16:19:48 INFO util.GSet: VM type       = 32-bit
			13/03/08 16:19:48 INFO util.GSet: 2% max memory = 19.33375 MB
			13/03/08 16:19:48 INFO util.GSet: capacity      = 2^22 = 4194304 entries
			13/03/08 16:19:48 INFO util.GSet: recommended=4194304, actual=4194304
			[Fatal Error] mapred-site.xml:12:1: XML document structures must start and end within the same entity.
			13/03/08 16:19:49 FATAL conf.Configuration: error parsing conf file: org.xml.sax.SAXParseException: XML document structures must start and end within the same entity.
			13/03/08 16:19:49 ERROR namenode.NameNode: java.lang.RuntimeException: org.xml.sax.SAXParseException: XML document structures must start and end within the same entity.
		
		**根据提示，修改了mapred-site.xml第12行1列的错误，格式化成功**
		
		启动bin/start-all.sh

			$ bin/start-all.sh

		检测hadoop是否启动成功

			$ jps
			
		如果有Namenode，SecondaryNameNode，TaskTracker，DataNode，JobTracker五个进程，就说明你的hadoop单机版环境配置好了！

		如下所示：

			hadoop@Hadoop-Ubuntu-00:/usr/local/hadoop$ jps
			4722 DataNode
			5298 Jps
			4929 SecondaryNameNode
			5226 TaskTracker
			5012 JobTracker
			4487 NameNode

	至此，一个hadoop的为分布式模式就搭建好了。
	
	+ 运行wordcount测试hadoop环境
	
	拷贝WordCount.java到我们的家目录，下载的hadoop里带有WordCount.java，路径为：
	
	/usr/local/hadoop/src/examples/org/apache/hadoop/examples/WordCount.java

将其进行拷贝操作：

	$ cp /usr/local/hadoop/src/examples/org/apache/hadoop/examples/WordCount.java ~


在当前目录下创建一个用来存放WordCount.class的文件夹：

	$ mkdir classes

编译WordCount.java：

	javac -classpath /usr/local/hadoop/hadoop-hadoop-core-1.0.4.jar -d classes WordCount.java


>如果出现下面的这种错误
>
>		WordCount.java:53: cannot access org.apache.commons.cli.Options
>		class file for org.apache.commons.cli.Options not found
>    	String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();

>原因是由于其中的一个类所在的包在另外路径中
>请用下面的方法：

>		javac -classpath /usr/local/hadoop/hadoop-core-1.0.4.jar:/usr/local/hadoop/lib/commons-cli-1.2.jar -d classes WordCount.java

编译成功，classes下会出现一个org的文件夹.再对编译好的class进行打包：

	jar -cvf wordcount.jar -C classes/ .

到此为止，java文件的编译工作已经完成.

现在为测试工作准备需要的测试文件，创建2个文件file01、file02，内容如下:

file01:

	I look some one that pee in the public

file02:

	I hear some body shit in his home

在hadoop中创建input文件夹

	hadoop dfs -ls
	hadoop dfs -mkdir input
	hadoop dfs -ls

把file01、file02上传input中

	hadoop fs -put file01 input  

	hadoop fs -put file02 input  

	hadoop fs -ls input

使用以下命令运行分布式命令

	$ hadoop jar wordcount.jar org.apache.hadoop.examples.WordCount input output

查看output中的内容

	$ hadoop fs -cat output/part-r-00000
	
内容为

	I	2
	body	1
	hear	1
	his	1
	home	1
	in	2
	look	1
	one	1
	pee	1
	public	1
	shit	1
	some	2
	that	1
	the	1
	
hadoop浏览器控制台

hdfs管理
[http://10.200.187.27:50070/dfshealth.jsp](http://10.200.187.27:50070/dfshealth.jsp)

tasktracker管理

[http://10.200.187.27:50060/tasktracker.jsp](http://10.200.187.27:50060/tasktracker.jsp)

jobtracker管理

[http://10.200.187.21:50030/jobtracker.jsp](http://10.200.187.21:50030/jobtracker.jsp)
	
+ mahout

	+ 获取安装包
	
	转移到服务器内
	
		$ scp /Users/DavidFu/Desktop/文件名 hadoop@10.200.187.27:~/
	
	解压压缩包
	
		$ tar -xvf mahout-distribution-0.7-src.tar.bz2
	
	通过该maven安装mahout
	**再次应该注意你下载的是否是src包，否则无需mvn安装**
	
		$ cd mahout-distribution-0.7/
		$ mvn install
	
	可以通过如下命令跳过单元测试过程，该过程确实持续时间太长
		
		$ mvn clean install -DskipTests=true
		
	正确执行完毕后会在/mahout/core/target目录下生成相应的jar文件
	
	**在执行mvn install的过程中可能会出现assenbly plugin无法完成目标任务的问题，此问题可能是在mvn执行的过程会从网络上获取所需要的编译和汇编插件，你会看见，有很多即时的download，这个问题可能和当前的网络状态有关，确保网络稳定，再mvn一次即可**
	
	
	下载grouplens的数据集使用unzip解压后
	
		ml-100k/
	
	里面的数据形式是
	
		943 	1044    	3   			888639903
		用户ID    ItemID    preference分值    Timestamp时间戳	
	
	jar的参数要求
	
		Job-Specific Options:                                                           
  		--input (-i) input                                  Path to job input         
                                                      directory.                
  		--output (-o) output                                The directory pathname    
                                                      for output.               
  		--recommenderClassName (-r) recommenderClassName    Name of recommender class 
                                                      to instantiate            
  		--numRecommendations (-n) numRecommendations        Number of recommendations 
                                                      per user                  
  		--usersFile (-u) usersFile                          File of users to          
                                                      recommend for             
  		--help (-h)                                         Print out help            
  		--tempDir tempDir                                   Intermediate output       
                                                      directory                 
  		--startPhase startPhase                             First phase to run        
  		--endPhase endPhase                                 Last phase to run     
	
	在文件目录中找到ua.base文件，将它放到HDFS文件系统中去作为输入文件

		$ hadoop fs -put ./ua.base input 
		
	为了对于指定客户来完成推荐，所以我们使用了-u参数，上文件usersfile
	
		$ hadoop fs -put ./usersfile input
		
	内容为
	
		1 2
		2 10
		3 200
		4 400
		5 900
	
	之后再/mahout/core/target目录下，执行下面的一串命令：
	
		hadoop jar mahout-core-0.7-job.jar org.apache.mahout.cf.taste.hadoop.pseudo.RecommenderJob -i input/ua.base -o output/ -r org.apache.mahout.cf.taste.impl.recommender.slopeone.SlopeOneRecommender -u input/usersfile

	运行之后我们将结果从hdfs导出到本地，
	
		$ hadoop fs -get output/part-r-00000.gz ~
		
	会到home目录，解压一下，然后vim一下
	
		1 2   [1467:5.273459,1449:5.269988,1293:5.0,1639:5.0,1642:4.961697,1398:4.917642,1080:4.8413863,868:4.8192577,1189:4.79314    57,1463:4.765127]
  		2 10  [1500:5.8595767,1463:5.519434,1642:5.227846,1467:5.2203684,1449:5.18005,1643:5.1038065,1368:5.0921597,1398:5.046423,    1639:4.9851227,114:4.907535]
  		3 200 [1500:6.1834316,1463:5.9748735,1449:5.643898,1398:5.6061945,1467:5.605993,1612:5.5716624,1643:5.271633,1064:5.256677    6,114:5.2549276,1642:5.21734]
  		4 400 [1500:6.5,1398:6.1464467,1080:5.8535533,1449:5.597149,868:5.55503,119:5.5,1463:5.3786798,851:5.3281155,1642:5.08088,    1631:5.068409]
  		5 900 [1500:7.5,1463:5.5,1293:5.0,1080:4.1464467,1642:4.025774,851:4.0199866,1191:4.0,1643:3.9585469,1398:3.880141,1449:3.    8667347]
~                 
		
	就可以看到生成的推荐数据.
	
	**在使用命令的时候发现了mahout中slopeone实例中的一个代码错误**
	
	错误日志如下
		
		hadoop@Hadoop-Ubuntu-00:~/mahout-distribution-0.7/core/target$ hadoop jar mahout-core-0.7-job.jar org.apache.mahout.cf.taste.hadoop.pseudo.RecommenderJob -Dmapred.input.dir=input/ua.base -Dmapred.output.dir=output/recommend/ --recommenderClassName org.apache.mahout.cf.taste.impl.recommender.slopeone.SlopeOneRecommender

		13/03/15 16:01:03 INFO common.AbstractJob: Command line arguments: {--endPhase=[2147483647], --numRecommendations=[10], --recommenderClassName=[org.apache.mahout.cf.taste.impl.recommender.slopeone.SlopeOneRecommender], --startPhase=[0], --tempDir=[temp]}

		Exception in thread "main" java.lang.IllegalArgumentException: Can not create a Path from a null string
		at org.apache.hadoop.fs.Path.checkPathArg(Path.java:78)
		at org.apache.hadoop.fs.Path.<init>(Path.java:90)
		at org.apache.mahout.cf.taste.hadoop.pseudo.RecommenderJob.run(RecommenderJob.java:122)
		at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:65)
		at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:79)
		at org.apache.mahout.cf.taste.hadoop.pseudo.RecommenderJob.main(RecommenderJob.java:148)
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
		at java.lang.reflect.Method.invoke(Method.java:597)
		at org.apache.hadoop.util.RunJar.main(RunJar.java:156)

	
	首先是在没有使用usersfile参数的时候会发生以下错误。错误显示在运行

		org.apache.mahout.cf.taste.hadoop.pseudo.RecommenderJob.run(RecommenderJob.java:122)
	
	
	时发生错误，此后会调用PATH，并在生成实例的过程中因为参数为NULL所以导致了错误，但是当制定了usersfile之后，得到的结果却是所有用户的推荐结果，这又与参数设置有冲突，很奇怪的现象。
	
	跟踪看代码之后定位到了如下代码行
			
		Path usersFile = hasOption("usersFile") ? inputFile : new Path(getOption("usersFile"));
		
	这里有逻辑错误，正确的写法应该如下
		
	    Path usersFile = hasOption("usersFile") ? new Path(getOption("usersFile")):inputFile;
	    
	运行后在没有指定usersfile的情况下会得到全用户的推荐列表，而制定usersfile的情况下只会得到就没有问题了。
	
	**问题解决**
