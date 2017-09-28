# MRLDA
中山大学数据传播实验室MRLDA项目


## `MRLDA`的`Hadoop`运行指南
这是一份`hadoop`的`lda`运行指南,接下来会一步步的指导完成整个实验

### 步骤指导

#### 拥有一个github账号并且能够复制项目到本地
&nbsp;&nbsp;将项目代码克隆到目标文件夹，克隆项目之前确保你拥有一个github的账号，以及在本机生成可SSH密钥，并且将密钥保存至你的github账户上。具体可以参见[廖雪峰的github学习指南](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374385852170d9c7adf13c30429b9660d0eb689dd43a000)或者是
[github-菜鸟教程](http://www.runoob.com/w3cnote/git-guide.html)

#### 克隆`MRLDA`项目

	git clone git@github.com:lintool/Mr.LDA.git
  
  
#### 在`Hadoop`运行`MRLDA`代码

##### 1. 在hadoop的文件系统hdfs创建 input 文件夹
	hadoop fs -mkdir -p input

在hadoop的集群系统上创建了一个input文件夹，作为`hadoop`存放 `MRLDA` 输入文件的地方。
我们需要在input文件夹里放入数据，`ap-sample.txt`，假设`ap-sample.txt`的文件路径为` /home/hdfs/MRLDA_TEST/inputData/ap-sample.txt`，则输入命令：

	hadoop fs -put /home/hdfs/MRLDA_TEST/inputData/* input
可以使用`ls`命令查看是否上传文件成功：

	hadoop fs -ls input
结果输出为：
> [hdfs@zdhdp04 MRLDA_TEST]$ hadoop fs -ls input
Found 1 items
 -rw-r--r--   3 hdfs hdfs    3099675 2017-09-17 11:42 input/ap-sample.txt

b备注：ap-sample.txt的数据格式为：
> ap881218-0003   student privat baptist school allegedli kill ...
> ap880224-0195   bechtel group offer sell oil israel discount ...


##### 2. 将原始文本文件转换为`MRLDA`的标准格式文本
输入命令（执行环境为项目代码根目录下）：


    hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar cc.mrlda.ParseCorpus -input input/* -output ap-sample-parsed

将input文件里的所有文本输入到hadoop程序中，并将结果输出到`ap-sample-parsed`文件夹下。
请注意， 路径下一定要却分路径以及文件，如果想要输入的是某个文件夹下的所有文件 一定要在文件夹名字后面加 `/*`!!!!!，否则很可能会出现报错：

> 
Exception in thread "main" org.apache.hadoop.mapred.InvalidInputException: Input path does not exist:hdfs://zdhdp00.zd:8020/user/hdfs/.input


 运行完成后，可以通过ls命令查看输出的文件
 `hadoop fs -ls ap-sample-parsed`
 可以看到，`ap-sample-parsed`下正确的文件结构如下
> ap-sample-parsed/document
  ap-sample-parsed/term
  ap-sample-parsed/title
    
`target/mrlda-0.9.0-SNAPSHOT-fatjar.jar`文件是项目代码的文件.

可以使用以下命令来查看输出结果：

    hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile ap-sample-parsed/title 10 // 查看前10个document的id映射

  
  输出结果
 
 > Record 0
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
Record 5
Key: 6
Value: ap880216-0184
Record 6
Key: 7
Value: ap880216-0198
Record 7
Key: 8
Value: ap880217-0008
Record 8
Key: 9
Value: ap880217-0012
Record 9
Key: 10
Value: ap880217-0063
Record 10
Key: 11
Value: ap880217-0154
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
Record 5
Key: 6
Value: ap880216-0184
Record 6
Key: 7
Value: ap880216-0198
Record 7
Key: 8
Value: ap880217-0008
Record 8
Key: 9
Value: ap880217-0012
Record 9
Key: 10
Value: ap880217-0063
Record 10
Key: 11
Value: ap880217-0154

  hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar edu.umd.cloud9.io.ReadSequenceFile ap-sample-parsed/term 20 # 查看文档库里前20个单词
  
输出结果样例：
> Record 0
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
Record 5
Key: 6
Value: new
Record 6
Key: 7
Value: two
Record 7
Key: 8
Value: last
Record 8
Key: 9
Value: state
Record 9
Key: 10
Value: nation
10 records read.


##### 3. 运行MRLDA
接下来，根据输出文件夹`ap-sample-parsed`的结果进行主题提取，完整命令为：
`nohup hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar  cc.mrlda.VariationalInference  -input ap-sample-parsed/document -output ap-sample-lda  -term 10000 -topic 20 -iteration 50 -mapper 50 -reducer 20 >& lda.log &`
下面分部分解释：

` hadoop jar target/mrlda-0.9.0-SNAPSHOT-fatjar.jar  cc.mrlda.VariationalInference  -input ap-sample-parsed/document -output ap-sample-lda  -term 10000 -topic 20 -iteration 50 -mapper 50 -reducer 20`
这个命令，设置了`hadoop`的基本参数，以及lda的基本参数。为了让程序在后台自己跑，我们可以做其他的事情，并且就算我们退出当前的用户程序也不会被中断，就需要ubantu的`nohup`命令以及`&`命令。`nohup`用于使程序在用户退出登陆、关闭终端之后仍能继续运行，最后的`&`命令是让程序放到后台执行；`>&lda.log`指的是将原本要输出正在控制台的各种信息重定向输出到lda.log文件里， 打开即可看见命令运行的全部过程；`&`在最后指的是将程序。

要想查看lda.log文件，并不断地读入更新，就像在控制台查看一样，可以使用命令：
`tail -f lda.log`


至此，一次简单的并行化MRLDA操作就已完成。接下来，我们使用另外的数据集：20-new-groups来做MRLDA的测试。在这一例子里涉及的操作包括：将数据集格式化为MRLDA数据，将数据集导入集群的HDFS文件系统， 使用MRLDA运行提取主题，使用指标`held-out likelihood`以及`Topic coherence`衡量主题提取结果。


#### `20newsgroup`实例

##### 1 .下载数据集

	git clone git@github.com:lintool/Mr.LDA-data.git  # 克隆必要的处理文件项目
	wget http://qwone.com/~jason/20Newsgroups/20news-bydate.tar.gz  
	tar -xzvf 20news-bydate.tar.gz # 20news原始数据集
	
克隆项目到本地，进入Mr.LDA-data文件夹，可以看到以下文件夹以及压缩包

	cd Mr.LDA-data
	ls
	# 以下是显示内容
	# 上一个实例使用的数据集，以及处理文件
	ap-sample.txt.gz  
	ap.tgz  
	parse_ap.py 

	# 压缩包里有三个文件，分别是20new数据集的train(训练集)、test(测试集)、dev(验证集) 的划分，用标签表示； 
	# parse_20news文件夹内含python文件用于生成MRLDA所需的处理文本 
	20news-labels.tgz 
	parse_20news  
	README.md

处理原始数据集，生成训练文本、测试文本、验证文本
	
	$ tar xvfz 20news-labels.tgz # 解压
	$ python parse_20news/createLdacMrlda.py 20news-labels/train.labels 20news-labels/dev.labels 20news-labels/test.labels 10 500 # 执行其中的createLdacMrlda.py文件，并传入参数20new-labels里的三个文件， 10/500是可以自行设置的一个document里出现的单词的最低和最高频率。

以上代码执行的结果是产生以下文件：
		
	# LDAC文件
	20news.ldac.dev 20news.ldac.train 20news.ldac.test
	# MRLDA文件
	20news.mrlda.dev 20news.mrlda.train 20news.mrlda.test  
	# RAW文件
	20news.raw.dev 20news.raw.train 20news.raw.test 
	# STAT文件
	20news.stat.dev 20news.stat.train 20news.stat.test 
	# 整个文档的词汇库：
	20news.vocab.txt  

#### 2.  运行Mr.LDA

使用前一个步骤生成的文本 
