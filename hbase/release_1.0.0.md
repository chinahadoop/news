# Apache HBase 1.0.0 发布  
  
2015年2月24日，星期二  
  
## **开启一个新的时代：Apache HBase 1.0**  
  
## 社区的过去、现在及将来  
作者：Enis，Apache HBase 项目管理委员会（PMC）成员，HBase-1.0.0发布官  
  
Apache HBase社区已经发布了HBase 1.0.0版本。HBase经过七年发展，终于迎来这天，对于HBase项目的开发者来说，这意味着一个伟大的里程碑。
这个版本提供了一些让人激动的功能，并且，在不牺牲稳定性的前提下，引入了新的API。新版本在写数据及磁盘操作方面，与HBase 0.98.x完全兼容。  
  
在这篇文章中，让我们一同回忆HBase项目的过去、审视当前、展望未来。  
  
## 版本、版本、还是版本  
在细数这个版本的功能细节之前，让我们重温一下过去，看一看各个发布版本（release number）是如何产生的。HBase以Apache Hadoop的贡献项目
（contrib project）形式开始它的生命之旅，那大约是在2007年，还只存在于Hadoop项目的一个子目录中，与Hadoop一同发布。三年后，HBase
成为了独立的Apache顶级项目。由于HBase依赖于HDFS（Hadoop分布式文件系统，译者注），社区一直确保着HBase的主版本与Hadoop的主版本号的
同一性。比如，HBase 0.19.x就表示它是工作在Hadoop 0.19.x之上的，等等。  
  
![image](https://github.com/chinahadoop/news/blob/master/hbase/materials/versions.png)  
  
然而，渐渐的HBase社区希望一个HBase版本可以工作于多个Hadoop版本之上——不仅仅只工作于与其主版本匹配的某一个Hadoop版本。于是，正如上图
所示，一个新的版本命名规则产生了，版本号从接近1.0的0.90主版本开始。我们也遵从了奇偶发布版本号的惯例，即奇数版本号为“开发人员预览版”
（developer previews），偶数版本号为可以产品化的“稳定版”（stable）。因此，稳定版的发布版本包括0.90、0.92、0.94、0.96和0.98（可以
翻看一下HBase的版本说明获取更多的信息：https://hbase.apache.org/book.html#hbase.versioning ）。  
  
在0.98版本之后，我们将项目的主分支（trunk）版本命名为0.99-SNAPSHOT，但实际，我们已经官方的将可用的版本号码用光了！于是在去年，HBase
社区同意，项目已经成熟并且足够稳定，1.0.0版本的发布被提上日程。在经过了三个0.99.x系列的“开发者预览版”以及六个Apache HBase 1.0.0
候选版本的发布之后，HBase 1.0.0现在正式呈现给大家！一起来看看以上这幅由Lars George（2007年加入HBase项目，HBase全职committer，
现在Cloudera工作，译者注）奉献给大家的图，其中列出了版本发展的时间历史。这幅图描述了每个发布版本的时间线及其支持的生命周期，如果
某个正式版本与开发者预览版相关联，这里也会列出来（比如0.99->1.0.0）。  
  
## HBase-1.0.0，开启全新的时代  
1.0.0版本有三个目标：  
1）为将来的1.x版本奠定一个稳定的基础版本。  
2）稳定HBase集群及客户端的运行（方式）。  
3）使版本与兼容性度量更为清晰。  
  
包括之前的0.99.x版本的修改，1.0.0包含了已经解决的1500多个jira问题（jira为apache项目用于提交和解决开源项目问题的系统及体系，译者注）。
一些主要的修改：  
### **API重构及修改**  
  
HBase的client级API已经发展了好几年，为了简化其语意，同时有益于未来的扩展性和易用性，我们重新审视了1.0以前的API。最终，在1.0.0版本中
引入新的API，同时将hbase client侧的一些经常使用的API（包括HTableInterface, HTable and HBaseAdmin）设置为将不再支持
（deprecates，开源项目中某些API被deprecates，表示这些API在系统再演进一段时间之后会被去掉，同时不再基于其开发新的功能支持，译者注）。  
  
我们建议您更新您的应用，使用新的API，因为不再支持的（deprecated）API将会在未来的2.x系列版本中被删除。更详细的说明，可以参看：
http://www.slideshare.net/xefyr/apache-hbase-10-release 以及 http://s.apache.org/hbase-1.0-api。  
  
如果一个类或者方法是官方的“客户API”，该客户端API会被InterfaceAudience.Public这个类（https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/classification/InterfaceAudience.Public.html ）
注解（这是个annotation，译者注）（更详细的信息，参考HBase手册http://hbase.apache.org/book.html 的“11.1.1. HBase API Surface”一章）。
同时，对于以注解方式声明为client侧公开的类，所有的1.x版本将会对其API兼容。  
  
### **用时间一致的region备份实现读可用性**  
  
作为第一阶段的一部分，这个版本包含了一个实验性质的功能“用时间一致的region备份实现读可用性”
（"Read availability using timeline consistent region replicas"）。即，一个region可以在多个region server中以只读模式存在。region的
一个备份为主备份，可写，而其他备份将共享相同的数据文件。读请求可以被region的任意备份满足，同时以时间一致的保障备份RPC以满足高可用性。
对具体的细节感兴趣，可以参考JIRA HBASE-10070（https://issues.apache.org/jira/browse/HBASE-10070 ）。  
  
### **在线配置修改及合并0.89-fb分支的一些功能**  
  
Apache HBase项目中的0.89-fb是Facebook用于放置他们的修改的分支。JIRA HBASE-12147（https://issues.apache.org/jira/browse/HBASE-12147 ）
合入了其中的补丁，可以在不用重启region server的情况下重新加载HBase服务器配置。  
  
除此之外，还有很多很多的性能优化（优化WAL（Write-ahead-Log，HBase写日志，译者注）管道，disruptor的使用，多WAL，更多的使用off-heap
内存（不在java heap中，不被GC管理的内存，译者注），等等）以及bug fix，还有很多很棒的特性，太多了，无法在这里一一列举。大家可以从
官方的release notes（http://markmail.org/message/u43qluenc7soxloe ）中获取更详细的描述。release note以及参考说明中也会包含二进制
发布包、源代码、以及兼容性需求，支持的Hadoop及Java版本说明，从0.94、0.96、0.98版本如何升级以及很多其他重要的细节。  
  
HBase-1.0.0也是使用“语义版本”（“semantic versioning”，http://semver.org/ ）的开始。简单的说，将来的HBase发布版本将采用MAJOR.MINOR.PATCH
这样的显式兼容性语义形式。HBase说明书中包含兼容性以及可预知的不同版本之间区别的所有内容描述。  
  
## 下一步  
我们已经将HBase-1.0.0标志为下一个HBase的稳定版本，这意味着所有的新用户应当开始使用这一版本。但是，作为数据库，我们深知，切换到一个
全新的版本需要耗费一些时间。我们将继续维护0.98.x版本，直到我们的用户已经准备好接受老版本的结束。1.0.x以及1.1.0、1.2.0等产品线将从
它们相应的分支中发布，同时，当2.0.0及其它的主版本随时代应运而生时，也将遵循这一规则。  
  
读备份第二阶段的功能、以列族（column family）为单位的flush、procedure V2、 在SSD上存储WAL或列族数据等等，已经在即将实现的功能
列表中。  
  
## 总结  
最后想说说，HBase 1.0.0的发布过程是如此的漫长，期间得到了大量令人敬佩的人们的贡献，committer以及contributor们的辛勤工作促成
这一伟大里程碑。我们由衷的感谢我们的用户以及所有在这些年中为HBase做出贡献的人们。  
  
继续与HBase共舞吧！


正是