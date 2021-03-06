> 【小象科技】编者按：该篇文章作者是RedHat员工，Puppet专家。他用golang基于etcd开了个配管项目叫mgmt，本文介绍了开展这个项目的几个重要原因，也算是自身对配置系统的理解和思考。总结来说：要能并发执行，通过事件驱动节约资源和时间，通过分布式系统解决扩展性问题。

# 下一代配管工具mgmt，通过分布式系统解决扩展性问题

我对配管工具的关注对于这个博客的读者来说已经不是什么秘密了。用Puppet工作期间，我从其他黑客和社区那里学到了很多知识。

[我](https://ttboj.wordpress.com/2013/11/17/iteration-in-puppet)，[已经](https://ttboj.wordpress.com/2013/02/20/automatic-hiera-lookups-in-puppet-3-x)，[发布了](https://ttboj.wordpress.com/2012/11/07/preventing-duplicate-parameter-values-in-puppet-types)，[很多](https://ttboj.wordpress.com/2013/05/14/overriding-attributes-of-collected-exported-resources)，[相关](https://ttboj.wordpress.com/2012/08/23/how-to-avoid-cluster-race-conditions-or-how-to-implement-a-distributed-lock-manager-in-puppet)，[的文章](https://ttboj.wordpress.com/2014/07/24/hybrid-management-of-freeipa-types-with-puppet)，[来](https://ttboj.wordpress.com/2014/06/06/securely-managing-secrets-for-freeipa-with-puppet)，[分享](https://ttboj.wordpress.com/2014/06/04/hiera-data-in-modules-and-os-independent-puppet)，[自己的](https://ttboj.wordpress.com/2014/03/24/introducing-puppet-execagain)，[知识](https://ttboj.wordpress.com/2012/11/14/setting-timed-events-in-puppet)。[推动](https://ttboj.wordpress.com/2013/06/04/collecting-duplicate-resources-in-puppet)，[这个](https://ttboj.wordpress.com/2013/09/28/finite-state-machines-in-puppet)，[领域](https://ttboj.wordpress.com/2012/11/20/recursion-in-puppet-for-no-particular-reason)，[的进步](https://ttboj.wordpress.com/2013/11/27/advanced-recursion-and-memoization-in-puppet)。我花了很多个日日夜夜思考这些问题，但遗憾的是，目前没有一款配管工具可以简单有效地解决所有的问题。

对此，我想谈一下我对下一代配管工具概念的看法。我叫它mgmt。

##三重设计

mgmt和其他配管工具的不同之处，主要有以下三点：

1. 并行执行，尽可能地利用所有的资源。
2. 事件驱动，动态地监控并响应做出的改变。
3. 拓扑分布，这样集中式的问题可以被健壮的分布式系统代替。

代码可以在[这里](https://github.com/purpleidea/mgmt/)找到，但是你最好先听我讲一下这三种特性。

## 1）并行执行

基本上，所有的配管系统都用[图形](https://en.wikipedia.org/wiki/Graph_%28abstract_data_type%29)来表示资源之间的依赖关系，一般来说都是直接[关系的、非循环的](https://en.wikipedia.org/wiki/Directed_acyclic_graph)。

![](https://ttboj.files.wordpress.com/2016/01/graph1.png?w=584)

图中，黑色箭头表示依赖关系，红色箭头线性依赖关系。

不幸的是，上图中这种[拓扑排序](https://en.wikipedia.org/wiki/Topological_sorting)的版本通过一个单线程线性执行。而从理论上将，两个[互不相关的部分](https://en.wikipedia.org/wiki/Connectivity_%28graph_theory%29)完全可以并行执行。

![](https://ttboj.files.wordpress.com/2016/01/graph2.png?w=584)

上图红色箭头表示执行顺序。左右分别是两个不同的部分，可以并行执行。注意：2a和2b必须等1a执行完成之后才能执行，3a必须在1a、2a、2b成功执行之后才能执行。

一般来说，有一些节点会有共同的依赖。一旦共同的父节点条件满足，所有的子节点可以同时执行。

这是mgmt实现的一个主要特性。现在，在像安装包这种流程长的操作中，处理互不相关的部分时，可以大大提高性能。此外，它的好处还有很多。

由于现在的很多使用配管工具的服务器一般都使用了不同的模块，而这些模块又没有内部的依赖，因此在实践中性能的提升非常显著。

下面是一个明显的例子。我设置了四项处理任务，其中三项有依赖关系，采用线性执行，第四项和其他的没有依赖关系，系统设置为在所有处理完成后5s退出。显而易见，最长的处理周期是35s。运行结果如下图所示：

	$ time ./mgmt run --file graph8.yaml --converged-timeout=5 --graphviz=example1.dot
	22:55:04 This is: mgmt, version: 0.0.1-29-gebc1c60
	22:55:04 Main: Start: 1452398104100455639
	22:55:04 Main: Running...
	22:55:04 Graph: Vertices(4), Edges(2)
	22:55:04 Graphviz: Successfully generated graph!
	22:55:04 State: graphStarting
	22:55:04 State: graphStarted
	22:55:04 Exec[exec4]: Apply //exec4 start
	22:55:04 Exec[exec1]: Apply //exec1 start
	22:55:14 Exec[exec4]: Command output is empty! //exec4 end
	22:55:14 Exec[exec1]: Command output is empty! //exec1 end
	22:55:14 Exec[exec2]: Apply //exec2 start
	22:55:24 Exec[exec2]: Command output is empty! //exec2 end
	22:55:24 Exec[exec3]: Apply //exec3 start
	22:55:34 Exec[exec3]: Command output is empty! //exec3 end
	22:55:39 Converged for 5 seconds, exiting! //converged for 5s
	22:55:39 Interrupted by exit signal
	22:55:39 Exec[exec4]: Exited
	22:55:39 Exec[exec1]: Exited
	22:55:39 Exec[exec2]: Exited
	22:55:39 Exec[exec3]: Exited
	22:55:39 Goodbye!
	
	real    0m35.009s
	user    0m0.008s
	sys     0m0.008s
	$
	
上面的输出中，我稍作了修改，删除了一些不必要的log，添加了一些注释，但输出都是真实内容。这个工具还能产生一个[图表](https://en.wikipedia.org/wiki/Graphviz)，来帮助你理解问题：

![](https://ttboj.files.wordpress.com/2016/01/example1-dot.png?w=584)

实际应用中，还有更多典型的例子。

##2）事件驱动

所有的配管工具都有幂等操作的概念。简单来说，[幂等操作](https://en.wikipedia.org/wiki/Idempotence)可以被多次调用产生的结果和第一次调用的结果相同。在实际应用中，每一份独立的资源都会检查元素的状态，一旦状态发生改变，就会采取一系列的操作。

当下的配管工具，基本上都是每30分钟检查一次状态，或多或少，有一些甚至只依靠手动来检查。这些都是成本很高的操作，每一次检查操作都会带来成本。归根结底，还是因为流程不能并行操作。

![](https://ttboj.files.wordpress.com/2016/01/graph3.png?w=584)

在上面这张图中，时间进程从左到右。三个元素从上到下，状态应该是a，b，c。初始化时，第一、二个元素状态需要改变，第三个已经正确。t1时改变第一、二个元素的状态。t2的时候,元素被外力改变，系统不再保持正常的状态。直到t3再次检查的时候，我们并不知道这个改变，在t3时才再次修复。在这个过程中，有30分钟的时间系统不在正常状态。

除了性能问题，当系统改变的时候，最多需要30分钟才能发现改变！

mgmt系统特殊在，可以基于系统，采用事件驱动的方式，提供更加有效、迅速的解决方案。这就是mgmt配管工具的第二个特性。

我们所说的事件是指：文件改变的[inode事件](https://en.wikipedia.org/wiki/Inotify)，服务改变的[系统事件](https://en.wikipedia.org/wiki/Systemd)，包改变的[包管理工具](https://en.wikipedia.org/wiki/PackageKit)事件，以及调用、计时、网络操作等事件。在下面inode事件中，第一次运行mgmt系统的时候，先检查要管理的文件的inode，将状态初始化为系统要求的状态。之后除非inode事件发生，否则我们都不要再检查状态了。

![](https://ttboj.files.wordpress.com/2016/01/graph4.png?w=584)

在上图中，初始化之后，所有的元素都修改为系统要求的状态，如果其中发生改变，mgmt会立即做出反应并及时修正。

mgmt使用高手可能会发现下面三点有趣的地方：

1. 如果我们不想要mgmt程序一直监听事件的话，可以让它在初始化之后退出，30分钟之后运行。这可以通过 --converged-timeout=1标志来实现。这和当前的配管工具是一样的，如果实在不想体验这种全新的特性的话可以选择这种方式。换句话说，现在的系统就是mgmt的一个特殊模式！
2. 如果一些资源并没有提供事件监听机制的话，使用轮询的方法也是可行的。虽然现在没有已知的遗漏API。
3. 使用这种架构建立一个监听系统是可行的，事实上，我在讨论的配管工具就是一个监听系统的概念，有着一样的规律。

下面这个特性的一个小例子。开始，我建立了三个文件f1，f2和f3，可以验证它们的文件内容，并且确认文件f4不存在。然后我们可以发现，mgmt的反应是如此之快：

	james@computer:/tmp/mgmt$ ls
	f1  f2  f3
	james@computer:/tmp/mgmt$ cat *
	i am f1
	i am f2
	i am f3
	james@computer:/tmp/mgmt$ rm -f f2 && cat f2
	i am f2
	james@computer:/tmp/mgmt$ echo blah blah > f2 && cat f2
	i am f2
	james@computer:/tmp/mgmt$ touch f4 && file f4
	f4: cannot open `f4' (No such file or directory)
	james@computer:/tmp/mgmt$ ls
	f1  f2  f3
	james@computer:/tmp/mgmt$
	
实在是太快了！

##3）拓扑分布

所有的软件从某种意义上说都是以某种拓扑结构运行的。Puppet和Chef是普通的[客户端/服务器](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)结构，一个服务器对应多个客户端，每一个都有一个代理。它们都有一种独立的模式，但是将它们聚集起来，采用一种多机器的架构，就更有意思了。

![](https://ttboj.files.wordpress.com/2016/01/graph5.png?w=584)

上图描述的是一台服务器响应三个客户端的模式。

传统的模式广为人知，理解起来也非常简单。简单说，就是将代码都放到一个地方（服务器），客户端或代理通过简单的配置就能工作。然而，这存在着性能和扩展性的问题，如果配管工具中心出了问题，就不能开启新的机器或者修改现有的机器。这可能导致严重的后果。

其他像[Ansible](https://en.wikipedia.org/wiki/Ansible_%28software%29)这种在我眼里更像是[协调器](https://en.wikipedia.org/wiki/Orchestration_%28computing%29[)，而不是配管工具。这就是说，他们实际上不共享很多问题空间，而是通过幂等加入了很多传统配管工具的特性，是非常实用、重要的工具。

![](https://ttboj.files.wordpress.com/2016/01/graph6.png?w=584)

通过推送模型的方式工作，是协调器的主要不同。当一台服务器（系统管理的顶层）想要控管理一台机器的时候就要主要和它建立连接。这样的优点是，很容易推出多机器的计算机结构，然而也有单点错误的通病。此外，当面对大型的集群时，性能又成了一个问题。实际应用中，这些集群通常会分成组或逻辑单元来降低压力，但是这又降低了前面提到的简单的优点。

更糟的是，以上的两种拓扑结构，在发生问题时都不能不借助第三方监视器快速定位问题。采用协调器方式来管理的，无法定位问题并提供反馈，也无法告诉服务器或者聚集器做出响应的反应。

好消息是，现在以及将来像Paxos family和Raft这种拓扑结构的算法已经被广泛接受，实现比较成熟，并且开放为自由软件了。Mgmt基于三种算法形成网状代理。不存在客户端和服务器的区别，都是平等的。每一个节点都可以从其他分布的数据存储中输出或收集数据。现在的这种软件实现是一项叫[etcd](https://en.wikipedia.org/wiki/Etcd)的了不起的工程。

![](https://ttboj.files.wordpress.com/2016/01/graph7.png?w=584)

从上图中可以看到，一个完全互相连接的拓扑结构是怎样的。连接数很大，试着想象一下完全连接128个节点所需要的边数吧。

实际应用中，要完成所有点对点的连接需要的边数很大，所以，先让集群总的划分成分布式的阵列，每一个阵列中选取一个作为主管机处理etcd事务，集群中的代理通过主管机和其他机器连接。分布的数据存储可以轻松地处理错误，如果和主管机的连接发生错误，也可以方便地切换到一个临时的主管机。随着处理事务的增多和减少，集群可以通过向中心询问etcd进度自动的升高和降低级别。

![](https://ttboj.files.wordpress.com/2016/01/graph8.png?w=584)

在上图中，可以看到一个紧密相连的节点中心，同时运行着配管事务和etcd管理。其他的节点都与其相连。这样很容易聚集节点。将来的算法将朝着主机数增加的方向设计和优化。将etcd的管理机运行在不同的故障域无疑是聪明的做法。

节点可以从分布的数据存储中输出和收集数据，可以用一个像Puppet中叫数据输出的装置实现。我认为，这种装置和数据的交换是一个很好的方案，但是这种实现也有明显的缺点。比如，如果一个集群有N个节点，每一个都想要和其他的节点交换数据，那么puppet就必须运行N-1次，每次检查N遍来查看所有的数据。每一次检查的开销都很大。

而在mgmt中，图表只有在有etcd事件发生的时候才刷新，此时，只有相关的成员节点运行。实际中，一个小的集群只需要很少的数据处理时间，可以忽略不计。

下面是一个很好的例子：

- 在系统中有三个节点：A，B，C
- 每个都有四个文件，其中两个输出
- 在节点A中，文件是/tmp/mgmtA/f1a 和 /tmp/mgmtA/f2a
- 在节点A中，输出/tmp/mgmtA/f3a 和 /tmp/mgmtA/f4a
- 在节点A中，收集所有可能的数据，输出到tmp/mgmtA/
- B和C做同样的操作，只不过在上面的路径中，A改为B
- 为了演示目的，我现在A，然后B，最后C上面启动mgmt，同时运行一些终端的指令来进行更新。

此外，我对输出的log做了一些整理和注释：

	james@computer:/tmp$ rm -rf /tmp/mgmt* # clean out everything
	james@computer:/tmp$ mkdir /tmp/mgmt{A..C} # make the example dirs
	james@computer:/tmp$ tree /tmp/mgmt* # they're indeed empty
	/tmp/mgmtA
	/tmp/mgmtB
	/tmp/mgmtC
	
	0 directories, 0 files
	james@computer:/tmp$ # run node A, it converges almost instantly
	james@computer:/tmp$ tree /tmp/mgmt*
	/tmp/mgmtA
	├── f1a
	├── f2a
	├── f3a
	└── f4a
	/tmp/mgmtB
	/tmp/mgmtC
	
	0 directories, 4 files
	james@computer:/tmp$ # run node B, it converges almost instantly
	james@computer:/tmp$ tree /tmp/mgmt*
	/tmp/mgmtA
	├── f1a
	├── f2a
	├── f3a
	├── f3b
	├── f4a
	└── f4b
	/tmp/mgmtB
	├── f1b
	├── f2b
	├── f3a
	├── f3b
	├── f4a
	└── f4b
	/tmp/mgmtC
	
	0 directories, 12 files
	james@computer:/tmp$ # run node C, exit 5 sec after converged, output:
	james@computer:/tmp$ time ./mgmt run --file examples/graph3c.yaml --hostname c --converged-timeout=5
	01:52:33 main.go:65: This is: mgmt, version: 0.0.1-29-gebc1c60
	01:52:33 main.go:66: Main: Start: 1452408753004161269
	01:52:33 main.go:203: Main: Running...
	01:52:33 main.go:103: Etcd: Starting...
	01:52:33 config.go:175: Collect: file; Pattern: /tmp/mgmtC/
	01:52:33 main.go:148: Graph: Vertices(8), Edges(0)
	01:52:38 main.go:192: Converged for 5 seconds, exiting!
	01:52:38 main.go:56: Interrupted by exit signal
	01:52:38 main.go:219: Goodbye!
	
	real    0m5.084s
	user    0m0.034s
	sys    0m0.031s
	james@computer:/tmp$ tree /tmp/mgmt*
	/tmp/mgmtA
	├── f1a
	├── f2a
	├── f3a
	├── f3b
	├── f3c
	├── f4a
	├── f4b
	└── f4c
	/tmp/mgmtB
	├── f1b
	├── f2b
	├── f3a
	├── f3b
	├── f3c
	├── f4a
	├── f4b
	└── f4c
	/tmp/mgmtC
	├── f1c
	├── f2c
	├── f3a
	├── f3b
	├── f3c
	├── f4a
	├── f4b
	└── f4c
	
	0 directories, 24 files
	james@computer:/tmp$
	
惊人的事情发生了，集群在不到1秒的时间内就做了响应。而且并没有太多的IO事件，虽然这些都是常量，但也显示出了处理速度之快。你也可以自己做一下测试。

##代码

代码已经公开很长时间了。我想早一点发布，但是直到我对这三个特性满意之前，不想在博客上谈起这个。程序完全由go语言写成，我认为这个语言能很好地满足我的需求。这是我第一个公开的go语言项目，所以这个项目的很多地方还有待提高。欢迎大家提出意见和补丁。目前这个项目属于自由软件，并且我坚持在以后也不改变这种方式。

##社区

我们有一个线上频道：[#mgmtconfig on Freenode](https://webchat.freenode.net/?channels=#mgmtconfig)

如果将来发展的不错，我们会建立一个邮件列表。

##总结

感谢你读到了这里！我希望你喜欢我的工作，我保证将来会发更多有关mgmt设计的东西，欢迎评论。

Happy hacking!

James

译者/赖信涛

原文：[Next generation configuration mgmt](https://ttboj.wordpress.com/2016/01/18/next-generation-configuration-mgmt/)