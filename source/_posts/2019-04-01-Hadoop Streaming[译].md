---
layout: post
title: Hadoop Streaming手册[译]
date: 2019-04-01
categories: hadoop
tags: [hadoop,streaming]
abstract: hadoop streaming是hadoop发行版附带的功能. 该功能让我们可以使用任何程序或脚本作为mapper和reducer 来创建Map/Reduce作业.
---

<!-- toc -->

## 概述

hadoop streaming是hadoop发行版附带的功能. 该功能让我们可以使用任何程序或脚本作为mapper和reducer 来创建Map/Reduce作业.例如：

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
	-input inputDirs \
	-output outputDirs \
	-mapper /bin/cat \
	-reducer /usr/bin/wc 
```

## Streaming运行原理

上述例子中，mapper和reducer都是执行程序，该执行程序从标准输入(stdin)逐行读取数据并将数据吐到标准输出(stdout)中.hadoop streaming创建Map/Reduce作业、提交到对应的集群中并监控作业运行进度.

指定一个可执行程序作为mapper并初始化成功后，每个mapper任务将启动该执行程序.　当mapper作业运行时，它讲输入数据转化成行并逐行写入到该执行程序进程的标准输入中.同时mapper作业面向行收集该执行程序进程的标准输出(以\n作为分割符切分数据)并将每行数据转成成key/value格式，将转化后的kv数据作为mapper作业的输出. 默认配置下，以tab作为分割符，每行数据分割后第一部分作为key其余部分作为value.如果行数据中没有制表符那么整行数据作为key, value为空. 我们可以通过设置-inputformat命令行参数来个性化定制key value格式．

这是Map/Reduce框架与streaming mapper/reduce通信协议的基础.

用户可以通过设置 `stream.non.zero.exit.is.failure`参数（true|false）来确定streaming作业是以非零状态退出表示作业运行成功或失败．默认情况下,streaming　作业进程以非零状态退出表示任务失败.

## Streaming 命令行选项

Streaming 支持hadoop通用命令行选项的同时也支持streaming命令行选项，下面展示了常规命令行选项的语法格式.

**注意：**　一定要把通用命令行选项设置放置在streaming命令行选项之前，否则命令行选项设置将无效. 

```bash
hadoop command [genericOptions] [streamingOptions]
```

**Ｈadoop streaming 命令行选项列表如下.**

| 参数                                   | 必选 | 描述                                                         |
| :------------------------------------- | :--: | ------------------------------------------------------------ |
| -input 目录名或文件名                  |  是  | mapper作业输入数据位置                                       |
| -output 目录名                         |  是  | reducer作业输出位置                                          |
| -mapper 执行程序或Java类名             |  否  | Mapper执行程序，若未设置则默认使用IdentityMapper             |
| -reducer 执行程序或Java类名            |  否  | Reducer执行程序，若未设置则默认使用IdentityReducer           |
| -file 文件名                           |  否  | 让mapper,reducer,combiner作业能够在对应节点上本地化读取该文件 |
| -inputformat JavaClassName             |  否  | 设置的类必须返回 key(Text类型)/Values(Text类型)键值对．若未设置该选项则默认使用TextInputFormat |
| -outputformat JavaClassName            |  否  | 同上                                                         |
| -partioner JavaClassName               |  否  | 该类确定输出数据数据写入到哪一个reduce分区                   |
| -combiner streaming命令或JavaClassName |  否  |                                                              |
| -cmdenv name=value                     |  否  | 给streaming作业传递环境变量                                  |
| -inputreader                           |  否  |                                                              |
| -verbose                               |  否  | verbose输出                                                  |
| -lazyOutput                            |  否  |                                                              |
| -numReduceTasks                        |  否  | reducer作业数量(mapreduce.job.reduces)                       |
| -mapdebug                              |  否  | map作业失败时调用的脚本                                      |
| -reducedebug                           |  否  | reduce作业失败时调用的脚本                                   |

###指定Java类作为Mapper/Reducer

我们可以提供一个Java类作为mapper或者reducer

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
-input inputdirs \
-output outputdir \
-inputformat org.apache.hadoop.mapred.KeyValueTextInputFormat \
-mapper org.apache.hadoop.mapred.lib.IdentifyMapper \
-reducer /usr/bin/wc
```

我们可以通过设置`stream.non.zero.exit.is.failure`为`false`或`true`来确定是否以执行程序退出非零状态码来表示任务成功或失败. 默认程序非零退出状态表示任务失败.

###文件打包到作业提交

我们可以执行任意执行程序作为mapper或reducer.可执行程序并不需要预先保存在集群上;若未提前保存在集群节点中,则需要通过`-file`选项通知框架打包相应执行程序作为作业提交的一部分. 例如:

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
-input inputdirs \
-output outputdir \
-mapper mapper.py \
-reducer /usr/bin/wc \
-file mapper.py
```

上如例子中指定了一个用户定义的python 可执行脚本作为mapper. `-file mapper.py` 将python可执行文件作为作业提交的一部分传输到集群.除了打包可执行文件,还可以打包其他辅助文件(比如字典文件,配置文件等)供mapper作业使用. 如下所示:

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
  -input myInputDirs \
  -output myOutputDir \
  -mapper myPythonScript.py \
  -reducer /usr/bin/wc \
  -file myPythonScript.py \
  -file myDictionary.txt
```
###为作业指定其他插件

跟其他普通Map/Reduce作业一样,我们可以指定其他插件到streaming作业.

```bash
 -inputformat JavaClassName
 -outputformat JavaClassName
 -partitioner JavaClassName
 -combiner streamingCommand or JavaClassName
```

给输入格式指定的Java类必须返回Text 格式的键值对类型. 若没有特别指定输入格式处理类,那么`TextInputFormat`将作为默认处理类.由于`TextInputFormat` 返回LongWriteable类型key, 并且key不是输入数据的一部分,key会被丢弃.只有值会通过管道传递给streaming mapper.

为输出格式指定的类应该采用`Text`类型对键值对类型.如果没有指定输出格式处理类,`TextOutputFormat`作为默认处理类.

###设置环境变量

通过下述方式再streaming命令行中设置环境变量

```bash
-cmdenv EXAMPLE_DIR=/home/example/dic/
```



##  通用命令行选项

Hadoop streaming支持streaming命令行选项也支持hadoop通用命令行选项.通用命令行选项设置方式如下

**注意:**  一定要把通用命令行选项设置放置在streaming命令行选项之前，否则命令行选项设置将无效. 
```bash
hadoop command [genericOptions] [streamingOptions]
```

Streaming可以使用的hadoop命令行选项如下表所示:

| 参数                   | 必须 | 描述                                                    |
| ---------------------- | :--: | ------------------------------------------------------- |
| -conf configfile       |  否  | 指定特定的配置文件                                      |
| -D                     |  否  | 设置指定的属性(Dkey=value)                              |
| -fs host:port or local |  否  | 指定namenode                                            |
| -files                 |  否  | 指定需要copy到集群的文件,多个文件用逗号分开             |
| -libjars               |  否  | 指定需要copy到集群classpath的jar文件,多个文件用逗号分开 |
| -archives              |  否  | 指定要上传到集群并解压到压缩文件,多个文件用逗号分开     |

###使用-D选项设置配置变量

我们可以使用`-D<property>=<value>来设置额外的配置.

#### 设置目录

修改节点本地临时目录路径

```bash
-D dfs.data.dir=/tmp
```

设置额外的本地临时目录

```bash
 -D mapred.local.dir=/tmp/local
 -D mapred.system.dir=/tmp/system
 -D mapred.temp.dir=/tmp/temp
```

**注释:** 作业配置参数跟多细节请参考`mapred-default.xml`

####设置仅运行Map任务作业

我们经常需要仅运行map函数来处理输入数据,仅仅需要把`mapreduce.job.reduces`设置为0即可达到此目的.设置为0后,Map/Reduce框架不再创建reduce任务,并且mapper任务等输出会作为作业的最终输出.

```bash
-D mapreduce.job.reduces=0
```

####设置指定数量的Reducers

设置指定数量的reducers,例如设置2个:

```bash
hadoop jar hadoo-streaming-2.9.2.jar \
-D mapreduce.job.reduces=2\
-input inputdir \
-output outputdir \
-mapper /bin/cat \
-reducer /usr/bin/wc
```

#### 自定义输入行分割键值对的方式

如前面讲到,Map/Reduce框架从mapper任务的标准输出读取行数据时,它会把行数据分割成呢键值对形式.默认配置下,到第一个制表符的数据作为键,其余的数据作为值(不包含制表符tab).

但是,我们可以修改默认配置.我们可以指定一个分割符号来替换制表符.并且也可以值指定分割后的哪一部分作为key.如下所示

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
  -D stream.map.output.field.separator=. \
  -D stream.num.map.output.key.fields=4 \
  -input myInputDirs \
  -output myOutputDir \
  -mapper /bin/cat \
  -reducer /bin/cat
```

上面的例子中,通过`-D stream.map.output.field.separator=.`设置`.`作为map输出的字段分割符.并且指定到第四个分割符的前缀部分作为key,其余部分作为值.如果一行数据中的风格符号`.`不够4个那么整行数据讲作为key,值部分为空.

类似的,我们可以使用`-D stream.reduce.output.field.separator=SEP`和 `-D stream.num.reduce.output.fields=NUM` 来确定reduce的输出数据中哪一部分作为key哪一部分作为value.

类似上述方式,可以通过`stream.map.input.field.separato` 和`stream.reduce.input.field.separator`来设置Map/Reduce作业的输入数据分割符号.默认设置是以制表符作为字段分割符号.

### 使用大文件和归档文件

`-files`和`-archives`选项让我们可以在任务中使用普通文件和归档文件.它们的参数是指向上传到hdfs的文件或归档文件到URI.这些文件缓存在作业中.我们可以通过fs.default.name配置变量获取host和fs_port.

**注意:** -files和-archives选项是hadoop通用命令行选项,一定要把它们放置在hadoop streaming选项之前,否则设置无效. 

#### 让任务可访问文件

-files选项在当前任务的工作目录创建指向本地文件副本的符号连接.下面例子中hadoop在当前工作目录中自动创建了名为testfile.txt的符号链接.链接指向了本地testfile.txt副本.

```bash
-files hdfs://host:fs_port/usr/testfile.txt
```

也可以使用`#`来为-files 指定的文件创建一个不同名称的符号链接

```bash
-files hdfs://host:fs_port/user/testfile.txt#cachefile.txt
```

如果有多个文件可以用逗号分开,如下所示

```bash
-files /home/usr/testfile1.txt, /home/user/estfile2.txt
```

#### 任务访问归档文件

-archives选项让我们可以把本地jars文件复制到当前任务的工作目录并unjar(解压).

下面例子中,hadoop在任务当前工作目录中创建了一个名为testfile.jar的符号链接.这个符号链接指向了上传的jar文件解压后的目录.

```bash
-archives hdfs://host:fs_port/user/testfile.jar
```

使用`#`符号为-archives上传文件、解压后的目录创建一个不同名字的符号链接.

```bash
-archives hdfs://host:fs_port/user/testfile.tgz#tgzdir
```
下面例子中,input.txt包含两行数据,分别表示两个文件: cachedir.jar/cache.txt 和 cacheddir.jar/cache2.txt.  cacheddir.jar是指向包含cache.txt和cache2.txt文件的归档目录的一个符号链接.

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
                -archives 'hdfs://hadoop-nn1.example.com/user/me/samples/cachefile/cachedir.jar' \
                -D mapreduce.job.maps=1 \
                -D mapreduce.job.reduces=1 \
                -D mapreduce.job.name="Experiment" \
                -input "/user/me/samples/cachefile/input.txt" \
                -output "/user/me/samples/cachefile/out" \
                -mapper "xargs cat" \
                -reducer "cat"

$ ls test_jar/
cache.txt  cache2.txt

$ jar cvf cachedir.jar -C test_jar/ .
added manifest
adding: cache.txt(in = 30) (out= 29)(deflated 3%)
adding: cache2.txt(in = 37) (out= 35)(deflated 5%)

$ hdfs dfs -put cachedir.jar samples/cachefile

$ hdfs dfs -cat /user/me/samples/cachefile/input.txt
cachedir.jar/cache.txt
cachedir.jar/cache2.txt

$ cat test_jar/cache.txt
This is just the cache string

$ cat test_jar/cache2.txt
This is just the second cache string

$ hdfs dfs -ls /user/me/samples/cachefile/out
Found 2 items
-rw-r--r-* 1 me supergroup        0 2013-11-14 17:00 /user/me/samples/cachefile/out/_SUCCESS
-rw-r--r-* 1 me supergroup       69 2013-11-14 17:00 /user/me/samples/cachefile/out/part-00000

$ hdfs dfs -cat /user/me/samples/cachefile/out/part-00000
This is just the cache string
This is just the second cache string
```

## 使用范例

### Hadoop分区类

Hadoop有个对很多应用程序非常有用的类`KeyFieldBasePartitioner`.通过该类我们可以使Map/Reduce框架通过key中的一些字段来对输出进行分区,而不需要使用整个key域.例如:

```bash
hadoop jar hadoop-streaming-2.9.2.jar \
  -D stream.map.output.field.separator=. \
  -D stream.num.map.output.key.fields=4 \
  -D map.output.key.field.separator=. \
  -D mapreduce.partition.keypartitioner.options=-k1,2 \
  -D mapreduce.job.reduces=12 \
  -input myInputDirs \
  -output myOutputDir \
  -mapper /bin/cat \
  -reducer /bin/cat \
  -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner
```

上面例子里,*-D stream.map.output.field.separator=.* 和 *-D stream.num.map.output.key.fields=4* 用法含义已经在前面解释过.整个两个变量主要用来确定streaming中的key和value.

Map/Reduce作业中map阶段的输出包含四个字段并用`.`隔开. Map/Reduce框架通过 `-Dmapred.text.key.partitioner.option=k1,2` 选项确定使用key中前两个字段将map输出分区. `-Dmap.output.key.field.separator=.`指定输出数据字段间的分割符号.从而保证了key中前两个字段相同的数据将分配给同一个reducer任务(写入到容一个分区中).

这相当于将key分成4个字段,前两个字段为联合主键其余部分为次主键. 主键部分用于分区(确定写到哪个分区上),主键与次主键组合用于字典排序.通过下面例子可以诠释一下:

map输出数据的key部分
```bash
11.12.1.2
11.14.2.3
11.11.4.1
11.12.1.1
11.14.2.2
```

写入到三个分区中(前两个字段用于分区键)

```bash
11.11.4.1
-----------
11.12.1.2
11.12.1.1
-----------
11.14.2.3
11.14.2.2
```

每个分区中使用整个全部字段进行排序
```
11.11.4.1
-----------
11.12.1.1
11.12.1.2
-----------
11.14.2.2
11.14.2.3
```

### Hadoop Comparator Class

Hadoop类库中,KeyFieldBasedComparator也是一个非常有用的类库.它实现了Unix/GNU sort特定的一个子集.例如
```bash
hadoop jar hadoop-streaming-2.9.2.jar \
  -D mapreduce.job.output.key.comparator.class=org.apache.hadoop.mapreduce.lib.partition.KeyFieldBasedComparator \
  -D stream.map.output.field.separator=. \
  -D stream.num.map.output.key.fields=4 \
  -D mapreduce.map.output.key.field.separator=. \
  -D mapreduce.partition.keycomparator.options=-k2,2nr \
  -D mapreduce.job.reduces=1 \
  -input myInputDirs \
  -output myOutputDir \
  -mapper /bin/cat \
  -reducer /bin/cat
```

如前面讲使用`.`分割字段等等. `-k2,2nr`表示以key中第二个字段作为排序字段, -n表示数字排序, -r表示逆序.如下所示,map的输出内容为
```
11.12.1.2
11.14.2.3
11.11.4.1
11.12.1.1
11.14.2.2
```
排序后的内容为(使用第二个字段排序,并且是将序[14,14,12,12,11])
```
11.14.2.3
11.14.2.2
11.12.1.2
11.12.1.1
11.11.4.1
```
### Hadoop Aggregate Package

	Hadoop有一个类库Aggregate. Aggregate提供一些特定reducer类,combiner类,以及一些简单的聚合操操作,如"sum","max","min"等等. 使用Aggregate我们可以定义一些mapper插件类用于生成相应的聚合选项.

一个聚合例子(-reducer aggregate)
```bash
hadoop jar hadoop-streaming-2.9.2.jar \
  -input myInputDirs \
  -output myOutputDir \
  -mapper myAggregatorForKeyCount.py \
  -reducer aggregate \
  -file myAggregatorForKeyCount.py \
```
myAggregatorForKeyCount.py代码如下
```python
#!/usr/bin/python

import sys;

def generateLongCountToken(id):
    return "LongValueSum:" + id + "\t" + "1"

def main(argv):
    line = sys.stdin.readline();
    try:
        while line:
            line = line&#91;:-1];
            fields = line.split("\t");
            print generateLongCountToken(fields&#91;0]);
            line = sys.stdin.readline();
    except "end of file":
        return None
if __name__ == "__main__":
     main(sys.argv)
```

### Hadoop Field Selection Class
Hadoop has a library class, FieldSelectionMapReduce, that effectively allows you to process text data like the unix “cut” utility. The map function defined in the class treats each input key/value pair as a list of fields. You can specify the field separator (the default is the tab character). You can select an arbitrary list of fields as the map output key, and an arbitrary list of fields as the map output value. Similarly, the reduce function defined in the class treats each input key/value pair as a list of fields. You can select an arbitrary list of fields as the reduce output key, and an arbitrary list of fields as the reduce output value. For example:

```
hadoop jar hadoop-streaming-2.9.2.jar \
  -D mapreduce.map.output.key.field.separator=. \
  -D mapreduce.partition.keypartitioner.options=-k1,2 \
  -D mapreduce.fieldsel.data.field.separator=. \
  -D mapreduce.fieldsel.map.output.key.value.fields.spec=6,5,1-3:0- \
  -D mapreduce.fieldsel.reduce.output.key.value.fields.spec=0-2:5- \
  -D mapreduce.map.output.key.class=org.apache.hadoop.io.Text \
  -D mapreduce.job.reduces=12 \
  -input myInputDirs \
  -output myOutputDir \
  -mapper org.apache.hadoop.mapred.lib.FieldSelectionMapReduce \
  -reducer org.apache.hadoop.mapred.lib.FieldSelectionMapReduce \
  -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner
```

The option “-D mapreduce.fieldsel.map.output.key.value.fields.spec=6,5,1-3:0-” specifies key/value selection for the map outputs. Key selection spec and value selection spec are separated by “:”. In this case, the map output key will consist of fields 6, 5, 1, 2, and 3. The map output value will consist of all fields (0- means field 0 and all the subsequent fields).

The option “-D mapreduce.fieldsel.reduce.output.key.value.fields.spec=0-2:5-” specifies key/value selection for the reduce outputs. In this case, the reduce output key will consist of fields 0, 1, 2 (corresponding to the original fields 6, 5, 1). The reduce output value will consist of all fields starting from field 5 (corresponding to all the original fields).

## FAQ

- How do I use Hadoop Streaming to run an arbitrary set of (semi) independent tasks?
- How do I process files, one per map?
- 应该使用多少个reducer
- 在脚本里设置的别名,-mapper结束后还能用么?
- 能使用unix 管道么?
...
