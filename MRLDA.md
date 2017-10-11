[TOC]

# MRLDA的Hadoop运行指南

中山大学数据传播实验室MRLDA项目

​	这是一份`hadoop`的`lda`运行指南, 阅读者假设已经对hadoop、LDA主题模型已经有基本了解，并且了解简单的linux终端命令。指南主要是通过两个实例指导熟悉。最后再附上基本的LDA分布式思想讲解。



### 1. 简单MRLDA使用

  	这是一个简单的MRLDA实例教程，比较完整的介绍了前期准备步骤以及运行MRLDA的基本步骤。



#### 1.1 前期准备工作

##### step 1. 拥有一个github账号并且能够复制项目到本地

​	将项目代码克隆到目标文件夹，克隆项目之前确保你拥有一个github的账号，在本机生成SSH密钥，并且将密钥保存至你的github账户上，能够可通一个项目到本地。具体可以参见[廖雪峰的github学习指南](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374385852170d9c7adf13c30429b9660d0eb689dd43a000)或者是[github-菜鸟教程](http://www.runoob.com/w3cnote/git-guide.html)

##### step 2. 克隆`MRLDA`项目

```shell
git clone git@github.com:lintool/Mr.LDA.git
```



#### 1.2  在Hadoop运行MRLDA代码

在Hadoop运行MRLDA的基本步骤为：

* 将训练文档上传hadoop的hdfs文件系统
* 将训练文档进行必要的预处理、格式化
* 运行MRLDA模型

请注意，在以上步骤里，文件的路径需要自己根据实验情况写明，除非有某些文件要求放在哪些文件夹下。



##### step 1. 将训练文档上传hadoop的hdfs文件系统

  在hadoop的文件系统hdfs创建 input 文件夹，并上传模型训练文件。这里使用项目自带的数据：`ap-sample.txt`

```shell
$ cd Mr.LDA  # 进入项目代码文件夹
$ hadoop fs -mkdir -p input # 在hafs文件系统上创建hadoop数据输入文件夹
$ hadoop fs -put ap-sample.txt input # 上传数据到input文件夹
$ hadoop fs -ls input # 查看文件夹内容

# ------------------------
# 结果输出为
# ------------------------
Found 1 items
-rw-r--r--   3 hdfs hdfs    3099675 2017-09-17 11:42 input/ap-sample.txt
# ------------------------
```

备注：ap-sample.txt的数据格式为： 文档id 文档单词 文档单词 文档单词 ......

因此，如果要使用自己的数据文件，只要保证文件为此格式即可。

> ap881218-0003   student privat baptist school allegedli kill ...
> ap880224-0195   bechtel group offer sell oil israel discount ...



##### step 2. 将训练文档进行必要的预处理、格式化

​	跑MRLDA模型之前，需要先进行数据的预处理，以及格式化数据。包括筛选停用词、统计不重复单词词库等等。

```shell
$ hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.ParseCorpus -input input/ap-sample.txt -output parse_output 
# 处理input文件夹下的ap-sample.txt文件，并将解析结果输出到parse_output文件
$ hadoop fs -ls parse_output # 查看输出文件内容

# ------------------------
# 显示内容如下
# ------------------------
parse_output/document
parse_output/term
parse_output/title
# ------------------------
```



#####  [错误警示] 

​	此处请注意， 路径下一定要却分路径以及文件，如果想要输入的是某个文件夹下的所有文件 一定要在文件夹名字后面加 `/*`!!!!! 否则很可能会出现如下报错：

> Exception in thread "main" org.apache.hadoop.mapred.InvalidInputException: Input path does not exist:hdfs://zdhdp00.zd:8020/user/hdfs/.input

&nbsp;&nbsp;

##### step 3. 运行MRLDA模型

  	接下来，根据输出文件夹`parse_output`的结果进行模型的迭代，命令为：

```shell
$ nohup hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar  cc.mrlda.VariationalInference  -input parse_output/document -output ap-sample-lda  -term 10000 -topic 20 -iteration 50 -mapper 50 -reducer 20 >& lda.log &`
```
* 命令`hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar  cc.mrlda.VariationalInference -input parse_output/document -output ap-sample-lda  -term 10000 -topic 20 -iteration 50 -mapper 50 -reducer 20` 设置了`hadoop`的基本参数，以及lda的基本参数。对于`MRLDA`模型的`hadoop`运行，`input`参数指明模型的输入文件，也就是上一步骤解析出来的`parse_output`文件夹下的`document`文件;  `output`参数代表模型输出文件夹; `term`参数设置了词库里一个单词的最大频率，`topic`为LDA生成的主题数，`iteration`是模型最大迭代数。

* `nohup`用于使程序在用户退出登陆、关闭终端之后仍能继续运行，最后的`&`命令是让程序放到后台执行；`>&lda.log`指的是将原本要输出正在控制台的各种信息重定向输出到lda.log文件里， 打开即可看见命令运行的全部过程。

* `lda.log`文件，记录了模型运行的整个进程。通过使用`tail`查看lda.log文件，可以不断地读入文件的更新内容，就像在控制台查看当前运行一样，命令如下：

  `tail -f lda.log`

&nbsp;&nbsp;

至此，一次简单的并行化MRLDA操作就已完成。



### 2. 复杂的MRLDA实例:20newsgroup 的提取与评估



&nbsp;&nbsp;接下来，我们使用另外的数据集：`20-new-groups`进行运行。在这一例子里涉及的基本操作包括：

* 将数据集导入集群HDFS文件系统

* 将数据集预处理与格式化

* 使用MRLDA运行提取主题

* 使用指标`held-out likelihood`以及`Topic coherence`衡量主题提取结果。

文件的路径需要自己根据实验情况写明，除非有某些文件要求放在哪些文件夹下。





#### 2.1. 下载数据集并进行初步处理

##### step 1. 下载数据集

​	下载数据集，并将解压后的`labels`文件夹、`test`数据文件夹、`train`数据文件夹一同放到 `parse_20news`文件夹下，也就是与 `createLdacMrlda.py`同一个根目录。

##### [错误警示] 

​	确保与parse_20news里的代码文件同一个目录下即可，否则会出现找不到文件的情况，或者最后生成的文件内容为空的情况 

```shell
$ git clone git@github.com:lintool/Mr.LDA-data.git  # 克隆数据文件到本地
$ cd Mr.LDA-data
$ ls

# ------------------------
# 以下是显示内容
# ------------------------
ap-sample.txt.gz  
ap.tgz  
parse_ap.py  # 文本预处理程序
#-------------------------
20news-labels.tgz # 包含 20new 数据集的train(训练集)、test(测试集)、dev(验证集) 的划分
parse_20news # 内含python文件，用于进行文本预处理
#-------------------------
README.md
#-------------------------

$ tar xvfz 20news-labels.tgz # 解压
# 将解压的label文件移到 parse_20news文件夹下  
$ mv 20news-labels parse_20news/ 
# 接下来下载原始数据集
$ wget http://qwone.com/~jason/20Newsgroups/20news-bydate.tar.gz  
$ tar -xzvf 20news-bydate.tar.gz # 解压20news原始数据集
$ mv 20news-bydate-test parse_20news/ # 将数据移到 parse_20news文件夹下
$ mv 20news-bydate-train parse_20news/ 
```



##### step 2. 初步处理格式化

​	初步处理原始数据集，执行createLdacMrlda.py文件，并传入参数20new-labels里的三个文件，得到训练集、验证集、测试集的划分数据; 10/500 是一个文档里单词最低和最高频率限制，可以自行设置。

```shell
$ python createLdacMrlda.py 20news-labels/train.labels 20news-labels/dev.labels 20news-labels/test.labels 10 500 # 默认python已安装

#-------------------------
# 以上代码执行的结果是产生以下文件,不同的文件格式不同
#-------------------------
# LDAC文件
20news.ldac.dev 
20news.ldac.train 
20news.ldac.test
# MRLDA文件
20news.mrlda.dev 
20news.mrlda.train 
20news.mrlda.test  
# RAW文件
20news.raw.dev 
20news.raw.train 
20news.raw.test 
# STAT文件
20news.stat.dev 
20news.stat.train 
20news.stat.test 
# 整个文档的词汇库：
20news.vocab.txt  
```





#### 2.2  将训练文档上传到hdfs文件系统，并进行预处理、格式化

##### step 1. 文件上传至hdfs文件系统

```shell
$ hadoop fs -mkdir mrlda_input # 创建就输入文件夹
$ hadoop fs -put 20news.mrlda.train mrlda_input  # 将20news.mrlda.train文件上传
$ hadoop fs -ls mrlda_input # 显示文件夹

#-------------------------
# 显示如下，说明上传成功
#-------------------------
Found 1 items
-rw-r--r--   3 hdfs hdfs    4939824 2017-09-28 15:51 mrlda_input/20news.mrlda.train
# ------------------------
```



##### step 2. 生成Mr.LDA格式的文件

```shell
$ hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.ParseCorpus    -input mrlda_input/20news.mrlda.train -output mrlda_output
$ hadoop fs -ls mrlda_output 

#-------------------------
# 查看output文件夹下结果，显示如下
#-------------------------
Found 3 items
drwxr-xr-x   - hdfs hdfs        0 2017-09-28 16:04 mrlda_output/document
-rw-r--r--   3 hdfs hdfs    68577 2017-09-28 16:04 mrlda_output/term
-rw-r--r--   3 hdfs hdfs   606353 2017-09-28 16:04 mrlda_output/title
# ------------------------
```



####  2.3 运行Mr.LDA

​	这一步的命令在 `1.2` 小节已介绍过，此处不再解释。完整命令为:	

```shell
$ nohup hadoop jar ldatrial/target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.VariationalInference -input mrlda_output/document -output mrlda_run_output -symmetricalpha 0.01 -topic 20 -term 100000 -iteration 1000  -mapper 50 -reducer 20 >& 20news.log &
```

​	在本次实验中，模型66次之后收敛，查看 `20news.log`的倒数几行文本，有如下显示内容：

```shell
INFO mrlda.VariationalInference: Model converged after 66 iterations...
```

​	在`mrlda_run_output`文件夹下，是每一次迭代产生的 $\alpha, \beta, \gamma$ 等蒂利克雷超参数记录。



#### 2.4 显示主题内容

以下命令用来查看模型收敛迭代后的每个主题下的单词分布

```shell
# 66为模型收敛的迭代数，20为一开始设置的主题数，这两个参数可以自行根据实验情况设定。结果文件输出为 20news.mrlda.train.20.topics
$ hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.DisplayTopic   -index mrlda_output/term  -input mrlda_run_output/beta-66  -topdisplay 20 > 20news.mrlda.train.20.topics

# 将文件复制到python处理文件所在的文件夹下
$ cp  20news.mrlda.train.20.topics ./parse_20news/ 

# 将 20news.mrlda.train.20.topics 文件转换为 20news.mrlda.train.20.ti.topics 文件，20为主题数
$ cd parse_20news
$ python convertMrldaTopicsToTopics.py 20news.mrlda.train.20.topics 20news.mrlda.train.20.ti.topics 20

# 使用cat 、tail或head三个命令可以显示文件的全部内容 / 文件倒数几行/n行内容 / 文件正数几行/n行内容，显示的文件即为主题下出现的单
$ cat[tail [-n]][head [-n]] 20news.mrlda.train.20.ti.topics 
```




#### 2.5 计算评估指标

​	这一小节介绍如何使用项目代码计算主题模型评价指标。

##### 2.5.1 计算 likelihood

​	该指标用来LDA模型生成的主题质量。计算会用到 [Blei's LDA implementation in C (LDAC) ](http://www.cs.princeton.edu/~blei/lda-c/) 提供的源码 ，然后根据在本步骤中生成`.beta`文件以及`.other`文件计算似然性。这两个文件的文件名前缀需要一样，可自行设定，在本次实验中，设为 `20news.mrlda.train.20.ldac`.

**step 1. 生成.beta以及.other文件** 

```shell
# 下载有关评估指标的代码，并编译
$ wget http://www.cs.princeton.edu/~blei/lda-c/lda-c-dist.tgz 
$ tar -xzvf lda-c-dist.tgz  # 解压
$ cd lda-c-dist 
$ make # 编译其中的c++文件

# 生成.beta文件，转换为LDAC格式,其中beta-66是模型收敛的迭代数，可自行更改
$ hadoop jar ldatrial/target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile mrlda_run_output/beta-66 > 20news.mrlda.train.20.beta 

# 显示文件的字节数、字符数、行数、文件名，在这里行数即为词库单词个数，文件的路径请自行写明更换
$ wc 20news.vocab.txt 
#-------------------------
# 显示内容如下：
#-------------------------
# 其中，82497就是行数，本实验中也是单词数
10283 10283 82497 Mr.LDA-data/parse_20news/20news.vocab.txt 
#-------------------------

# 生成ldac的.beta文件，82479 为 本次实验的词库单词数目VOCAB_SIZE
$ python parse_20news/convertMrldaBetaToBeta.py 20news.mrlda.train.20.beta 20news.mrlda.train.20.ldac.beta 82497

# 查看模型的alpha数值，66为迭代收敛的次数,本次实验结果显示约为0.018
$ hadoop jar  ldatrial/target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile mrlda_run_output/alpha-66

# 在.beta文件的目录下，使用nano文本编辑器生成.other文件，nano使用自行百度，与普通文档编译没有太大差异
$ sudo nano 20news.mrlda.train.20.ldac.other
# 在终端显示一个文本编辑器之后，输入以下信息，包括本次实验设置的主题个数、文档库的单词总数、alpha数值，然后保存退出：
num_topics 20
num_terms 82497
alpha 0.018
```

**step 2. 计算held-out-likehood**

```shell
# 各个文件的路径请写明，相对路径或是绝对路径均可，作为传入参数，相对路径请加上"./"符号.
# 其件lda与inf-settings.txt文件均在lda-c-list文件夹中，"./20news.mrlda.train.20.ldac" 为.beta、.other文件前缀及其路径，"./20news.mrlda.20.HL"  为生成的文件的前缀及其输出路径。
$ ./lda-c-list/lda inf ./lda-c-list/inf-settings.txt ./20news.mrlda.train.20.ldac ./20news.ldac.test ./20news.mrlda.20.HL 

#-------------------------
# 在目标目录下，会生成两个文件
#-------------------------
20news.mrlda.20.HL-gamma.dat  
20news.mrlda.20.HL-lda-lhood.dat
#-------------------------

# 将生成的 lhood.dat文件转移到 calculate_heldout_likelihood.py 所在的文件下，本次实验不做改动的话就是parse_news文件夹
$ cp 20news.mrlda.20.HL-lda-lhood.dat ./parse_20news/ 
# 执行计算，得到数值即为评价指标
$ python calculate_heldout_likelihood.py 20news.mrlda.20.HL-lda-lhood.dat 

#-------------------------
# 显示内容如下
#-------------------------
num_docs:  3646
avg likelihood:  -500.614934528
#-------------------------
```



##### 2.5.2 计算主题相关度

​	这一步可以计算`Topic coherence`评价指标。

```shell
# 获取计算代码
$ git clone https://github.com/jhlau/topic_interpretability 

# 生成文件夹
$ mkdir 20news_train_test_raws

# 复制文件至20news_train_test_raws文件夹下
$ cp -r 20news.raw.train 20news_train_test_raws/ 
$ cp -r 20news.raw.test 20news_train_test_raws/

# 生成 .oneline 文件
$ python parse_20news/createOneLineDict.py 20news.vocab.txt 

# 计算词库的单词频数，结果储存在 20news.train.test.wc 文件
$ python topic_interpretability/ComputeWordCount.py 20news.vocab.txt.oneline   20news_train_test_raws > 20news.train.test.wc

# 执行计算，输出结果到 20news.mrlda.20.oc 文件
$ python topic_interpretability/ComputeObservedCoherence.py 20news.mrlda.train.20.ti.topics npmi 20news.train.test.wc > 20news.mrlda.20.oc 
```



### 3. 简单的功能介绍 

​	这一章节介绍项目代码里提供的一些简单功能。



####  3.1查看文档ID号映射

​	MRLDA项目代码里提供了查看文档ID号映射的功能。以[1. 简单MRLDA使用](#1. 简单MRLDA使用 )小节的实验为例子，要查看文档的ID映射，使用一下命令

```shell
# 查看文档预处理生成的title文档，前5个document的id映射
$ hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile parse_output/title 5 

#-------------------------
# 输出结果
#-------------------------
 Record 0
 Key: 1
 Value: ap880212-0078
 Record 1
 Key: 2
 Value: ap880213-0055
 Record 2
 Key: 3
 Value: ap880213-0133
 Record 3
 Key: 4
 Value: ap880213-0186
 Record 4
 Key: 5
 Value: ap880216-0094
#-------------------------
```



#### 3.2 查看单词ID号映射

​	以[1. 简单MRLDA使用](#1. 简单MRLDA使用 )小节的实验为例子，要查看词库里不重复单词的ID映射，使用一下命令	

```shell
# 查看词库里前5个单词
$ hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile parse_output/term 5 

#-------------------------
# 输出结果
#-------------------------
Record 0
Key: 1
Value: said
Record 1
Key: 2
Value: year
Record 2
Key: 3
Value: would
Record 3
Key: 4
Value: one
Record 4
Key: 5
Value: also
5 records read.
#-------------------------
```



#### 3.3 查看主题内容 

​	项目代码里的提供主题显示功能能够查看所有主题下的单词，其使用方法已在[2.4 显示主题内容]()小节介绍，请返回查看。



#### 3.4 重新执行被迫中断的MRLDA模型

​	MRLDA模型的每一次迭代都会在输出文件夹下基本本次迭代产生的三个超参数 $\alpha, \beta, \gamma$  的数值。因此，如果MRLDA模型在未达到收敛状态之前由于其他原因被迫中断，我们可以使用 `modelindex`参数在这一步重新开始，继续迭代直到完成，以[1. 简单MRLDA使用](#1. 简单MRLDA使用 )实验为例。

```shell
# 查看当前迭代次数，或者自行查看lda.log查找,假设迭代了38次
$ hadoop fs -ls output
# 重新开始
$ nohup hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.VariationalInference -input parsed_output/document -output output -term 10000 -topic 20 -iteration 50 -mapper 50 -reducer 20 -modelindex 38 >& lda.log &




```
