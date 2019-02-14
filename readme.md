## 安装，build和使用DataX:

https://github.com/alibaba/DataX/blob/master/userGuid.md

python /opt/datax/bin/datax.py xx.json

解压之后:

+ /bin:用于执行的python脚本
+ /lib:是core common两个库jar包的位置
+ /plugin：是各种插件的位置
+ /job:自带的存放各类job配置的地方

```shell 
hduser@master:/opt/datax/job$ python /opt/datax/bin/datax.py job.json
```

### 生成Job的Json文件
```shell
python datax.py -r {YOUR_READER} -w {YOUR_WRITER}
EG: python /opt/datax/bin/datax.py -r streamreader -w streamwriter
```

这个例子用来生成最简单的往console上打印的例子

https://github.com/alibaba/DataX
READER和WRITER就是代码仓库当中的那些reader和writer

python /opt/datax/bin/datax.py stream2stream.json
会打印出50次   hello，你好，世界-DataX

## 一个从mysql抽往hdfs的例子

https://github.com/alibaba/DataX/blob/master/mysqlreader/doc/mysqlreader.md
这个例子将Mysql当中的元素取出，并且在stream上输出出来

https://github.com/alibaba/DataX/blob/master/hdfswriter/doc/hdfswriter.md
这个例子是将数据写入到hdfs当中

hive conf当中有hive-site.conf这个配置文件当中会指定hive在hdfs上保存文件的位置。
/user/hive/warehouse/

https://cwiki.apache.org/confluence/display/Hive/LanguageManual

```sql
create table text_table(
col1  SMALLINT,
col2  VARCHAR(255),
col3  VARCHAR(255)
)
row format delimited
fields terminated by "\t"
STORED AS TEXTFILE;
```

最后的job配置如下
```json
{
    "job": {
        "setting": {
            "speed": {
                 "channel": 3
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        },
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "123",
                        "column": [
                            "film_id",
                            "title",
							"description"
                        ],
                        "splitPk": "film_id",
                        "connection": [
                            {
                                "table": [
                                    "film"
                                ],
                                "jdbcUrl": [
     "jdbc:mysql://127.0.0.1:3306/sakila"
                                ]
                            }
                        ]
                    }
                },
               "writer": {
                    "name": "hdfswriter",
                    "parameter": {
                        "defaultFS": "hdfs://192.168.13.128:9000",
                        "fileType": "text",
                        "path": "/user/hive/warehouse/db_hive_edu.db/text_table",
                        "fileName": "hdfsText.txt",
                        "column": [
                            {
                                "name": "col1",
                                "type": "SMALLINT"
                            },
                            {
                                "name": "col2",
                                "type": "STRING"
                            },
                            {
                                "name": "col3",
                                "type": "STRING"
                            }
                        ],
                        "writeMode": "append",
                        "fieldDelimiter": "\t",
                        "compress":"GZIP"
                    }
                }
            }
        ]
    }
}
```

一个简单的抽取job设置完成，当中hive表的建立现在DATAX是完成不了的

## dataX相关概念

+ Job: Job是DataX用以描述从一个源头到一个目的端的同步作业，是DataX数据同步的最小业务单元。比如：从一张mysql的表同步到odps的一个表的特定分区。
+ Task: Task是为最大化而把Job拆分得到的最小执行单元。比如：读一张有1024个分表的mysql分库分表的Job，拆分成1024个读Task，用若干个并发执行。
+ TaskGroup: 描述的是一组Task集合。在同一个TaskGroupContainer执行下的Task集合称之为TaskGroup
+ JobContainer: Job执行器，负责Job全局拆分、调度、前置语句和后置语句等工作的工作单元。类似Yarn中的JobTracker
+ TaskGroupContainer: TaskGroup执行器，负责执行一组Task的工作单元，类似Yarn中的TaskTracker。

运行模式：
+ Standalone: 单进程运行，没有外部依赖。（默认的模式）
+ Local: 单进程运行，统计信息、错误信息汇报到集中存储。
+ Distrubuted: 分布式多进程运行，依赖DataX Service服务。

主要有三类产生脏数据的原因：

+ Reader读到不支持的类型、不合法的值。
+ 不支持的类型转换，比如：Bytes转换为Date。
+ 写入目标端失败，比如：写mysql整型长度超长。

看两张图：图1：

![](https://github.com/zhougit86/dataX_output/blob/master/plugin_dev_guide_1.png)

图2：

![](https://github.com/zhougit86/dataX_output/blob/master/plugin_dev_guide_2.png)

重编plugin，在项目总目录下运行下列的命令（可以根据需要注释掉不需要的module）
```shell
mvn clean package -DskipTests assembly:assembly
```

### standalone模式代码走读

在dataX.py当中查找startCommand，将其取消注释，可以看到究竟是怎样启动DataX的
```shell
python ../bin/datax.py ./mysql2stream.json > ~/cmd.txt

java -server -Xms1g -Xmx1g 
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/opt/datax/log -Xms1g -Xmx1g 
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/opt/datax/log 
-Dloglevel=info -Dfile.encoding=UTF-8 
-Dlogback.statusListenerClass=ch.qos.logback.core.status.NopStatusListener 
-Djava.security.egd=file:///dev/urandom -Ddatax.home=/opt/datax 
-Dlogback.configurationFile=/opt/datax/conf/logback.xml -classpath /opt/datax/lib/*:.  
-Dlog.file.name=ob_mysql2stream_json 
com.alibaba.datax.core.Engine -mode standalone -jobid -1 -job /opt/datax/job/mysql2stream.json

```

Job有一个Json配置
dataX本身有一个配置是在  conf/core.json
每个plugin也有json配置文件
最后的配置文件应该是将三个配置文件加在一起来使用的（todo待验证）

回到程序启动上：

最后一行的三个参数mode,jobid,job是会传到java当中去的，core/Engine当中会解析这几个参数
然后会调用Engine.start()，在start当中会有：
```java
        // 绑定column转换信息
        ColumnCast.bind(allConf);

        /**
         * 初始化PluginLoader，可以获取各种插件配置
         */
        LoadUtil.bind(allConf);
```

这两个方法，会将column 转换和plugin加载的工作完成

jobContainer里面有个initStandaloneScheduler，会保存下containerCommunicator

最后完成JobContainer的初始化之后，会调用container.start将container跑起来
分以下几个阶段

对应上一节图1

### preHandle:默认配置下为空，不需要关注

### init：初始化reader和writer

classLoaderSwapper完成plugin类的热加载
DefaultJobPluginCollector   会使用这个去初始化一些plugin之类的东西，在一开始的时候里面的那个Communicator是空的，
```
2019-02-12 23:23:46.997 [job-0] INFO  JobContainer - Xiaogang: reader init
mysqlreader
{"column":["film_id","title","description"],"connection":[{"jdbcUrl":["jdbc:mysql://127.0.0.1:3306/sakila"],
"table":["film"]}],"password":"123","splitPk":"film_id","username":"root"}
{"print":true}
null
```


以上是用parameter的配置加到reader和writer当中，而Commnicater实际上是空的。会在后面才完成初始化，应该是用来读取配置的

classLoaderSwapper.setCurrentThreadClassLoader(LoadUtil.getJarLoader(
        PluginType.READER, this.readerPluginName));
classLoaderSwapper.restoreCurrentThreadClassLoader();

每次运行一次reader,writer中的某个阶段，就要切换一下环境。


### prepare:
进行prepare阶段，貌似mysql和stream的prepare都为空


### split:
adjustChannelNumber： 当中一次对于byte,record进行限制，如果都没有设置，至少要针对Channel的速度进行限制，
拿当前的例子，就是返回了一个channel的number，用于后续计算reader数量的时候算出来一个adviceNumber

transformer在从mysql读Stream写的例子当中也不存在

reader task 和 writer task的数量永远是相等。

### schedule:
List<Configuration> taskGroupConfigs = JobAssignUtil.assignFairly(this.configuration,
                this.needChannelNumber, channelsPerTaskGroup);
是将task分配到不同的task group当中去

task configure的json比较长，参考
https://github.com/zhougit86/dataX_output/blob/master/taskGroupConfiguration.json

在schedule函数当中有一句
```java
scheduler.schedule(taskGroupConfigs);
```

是将上述task config作为入参传到schedule当中

在ProcessInnerScheduler当中有startAllTaskGroup这个方法，这里是完成所有数据转换的核心
```java
    @Override
    public void startAllTaskGroup(List<Configuration> configurations) {
        this.taskGroupContainerExecutorService = Executors
                .newFixedThreadPool(configurations.size());

        for (Configuration taskGroupConfiguration : configurations) {
            TaskGroupContainerRunner taskGroupContainerRunner = newTaskGroupContainerRunner(taskGroupConfiguration);
            this.taskGroupContainerExecutorService.execute(taskGroupContainerRunner);
        }

        this.taskGroupContainerExecutorService.shutdown();
    }
```