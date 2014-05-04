数据写入
============

在你通过Telnet 或 HTTP API push数据之前，或者使用 OpenTSDB 所支持的工具如 'tcollector'，这个时候您可能想立即投入并把数据丢到你的TSD中，但是要真正利用OpenTSDB强大的功能和灵活的优势，这恐怕要暂停下来先想下您的命名模式。

命名模式
^^^^^^^^^^^^^

一般metrics 管理员会为他们的时间序列提供一个简单的名称。例如系统管理员用 RRD-style 系统命名他们的时间序列 ``webserver01.sys.cpu.0.user``。这个名称告诉我们这个时间序列记录的是在 ``webserver01`` 上的 cpu ``0`` 的用户空间数量。假如你想获取特别是 web server 有些时候cpu核心的user time，这个非常适合。

但是，如果web server有64核你又想获取他们所有的平均时间。有些系统允许你指定一个通配符，如 ``webserver01.sys.cpu.*.user`` 这样就可以读出所有64个并且聚合的结果。或则你可以记录一个叫 ``webserver01.sys.cpu.user.all`` 新的时间序列来表示相同的聚合，但你必须写一个跟'64 + 1' 不同的时间序列。如果你有一千台web server你想要他们所有一起的cup平均时间，你可以制定类似 ``*.sys.cpu.*.user`` 的查询需要打开64,000个文件，然后聚合结果返回数据。或者你可以设置一个预先做好聚合处理的数据存储到 ``webservers.sys.cpu.user.all``。

OpenTSDB 通过引入标签 'tags' 思想来处理不同的事情。每一个时间序列都可以有一个 'metric' 名称，但他们更通用的是通过一些唯一时间序列共享一些东西。相反，它的唯一性是通过 key/value 标签（tag）实现，并允许灵活快速地聚合查询。

.. NOTE:: 在OpenTSDB 中的每一个时间序列必须至少有一个tag.

拿上面的 ``webserver01.sys.cpu.0.user`` metric为例，在 OpenTSDB 中这可能会成为 ``sys.cpu.user host=webserver01, cpu=0``。现在假如我们需要单个核的数据，我们可以制作一个类似 ``sum:sys.cpu.user{host=webserver01,cpu=42}`` 的查询。假如我们想要所有核的，我们简单的使用 ``sum:sys.cpu.user{host=webserver01}``。这将为我们聚合64个核的结果给我们。假如我们需要1000个服务器的结果，我们简单的请求 ``sum:sys.cpu.user``。底层数据模式存储``sys.cpu.user`` 所有时间序列是彼此相邻的，使得聚合各个值非常快速高效。因为大多数用户一开始就在一个较高的水平，OpenTSDB 的目的就是使这些聚合查询尽可能快。然后再钻研详细信息。

集合
------------

从长远看，如果你不明白OpenTSDB 如何查询，灵活的tag系统也可能会出现一些问题。还是以 ``sum:sys.cpu.user{host=webserver01}`` 查询为例，一个时间序列对应一个 cpu 内核，我们为 ``webserver01`` 记录了64 个唯一的时间序列。当我们发出一个查询，所有metric ``sys.cpu.user`` 还有 tag ``host=webserver01`` 都会被检索，平均，并返回一个时间序列数值,比方说，由此产生的时间戳为 ``1356998400`` 的值为 ``50``. 现在，我们从另一个系统预先集合号64个核的平均值只写一个时间序列 ``sys.cpu.user host=webserver01`` 存到OpenTSDB以便我们迅速得到平均值，如果我们执行相同的查询，我们会得到一个在 ``1356998400`` 的值是 ``100``。这是发生了什么呢？OpenTSDB 汇总了所有64个时间序列相加后预聚合时间序列为100。在存储方面，我们是这样做的：
::

  sys.cpu.user host=webserver01        1356998400  50
  sys.cpu.user host=webserver01,cpu=0  1356998400  1
  sys.cpu.user host=webserver01,cpu=1  1356998400  0
  sys.cpu.user host=webserver01,cpu=2  1356998400  2
  sys.cpu.user host=webserver01,cpu=3  1356998400  0
  ...
  sys.cpu.user host=webserver01,cpu=63 1356998400  1
  

如果查询一个metric的时间序列没有提供tag OpenTSDB将自动聚合所有时间序列。如果指定了一个或多个tag，聚合不管其他tag 只将匹配相应的tag所对应的所有时间序列.做 ``sum:sys.cpu.user{host=webserver01}`` 查询，我们将包括 ``sys.cpu.user host=webserver01,cpu=0`` 还有 ``sys.cpu.user host=webserver01,cpu=0,manufacturer=Intel``, ``sys.cpu.user host=webserver01,foo=bar`` 和 ``sys.cpu.user host=webserver01,cpu=0,datacenter=lax,department=ops``。这个例子告诉我们：*小心你的命名模式*.

时间序列基数
-----------------------

每一个命名模式的一个关键方面就是要考虑你的时间序列基数。基数的意思就是项目中唯一的一组数。在 OpenTSDB 中，它的意思就是一个metric 所关联的多个项目，即所有可能的tag名称和tag值的组合，也就是一组唯一的metric名称、tag名称和tag值。基数的重要性有以下两个原因。

**唯一 ID (UIDs)的限制** 

为每一个 metric、tag 名称和 tag 值所分配的 UID 数量是有限的，默认情况下每个类型的 UID 可以有 1600万多个。假如，如果你运行这个一个非常受欢迎的 web service，并且试图将跟踪客户IP做为 tag，比如：``web.app.hits clientip=38.26.34.10``，你可能很快碰到可能有超过40亿的IPv4地址的 UID 分配限制。另外，这种方法会导致为客户地址 ``38.26.34.10`` 创建一个稀疏的时间序列，仅偶尔会被你的应用所用到，或者永远都不会被用到这个特定的地址。

但是，UID限制通常并不是一个问题。一个 tag值分配一个 UID，这完全与它的 Tag 名称没有关联。假如你使用数字标示符做 tag 值，这个数字被分配一次 UID 可以用于许多 Tag名称。举个例子，我们给数字 ``2`` 分配了一个UID，我们可以使用 tag对 ``cpu=2``、``interface=2``、 ``hdd=2`` 和 ``fan=2`` 存储时间序列，仅消耗一个 tag 值 (``2``)  UID 和 4 个tag名称（``cpu``, ``interface``, ``hdd`` and ``fan``）的 UID。

如果你认为 UID 的限制会影响到你，首先要想想你要执行的查询。如果我们来看上面关于 ``web.app.hits`` 的例子，你可能仅仅关心到达你的服务所命中的总数量，并不关心某个特定的IP地址。这种情况下，你可能想将IP地址作为注释存储，如果你需要，这样你任然可以受益于低基数，你可以为特定IP搜索结果使用扩展脚本。（注：OpenTSDB 2.0 开始支持注释查询。）

如果你迫切需要超过1600万的值，你可以增加 OpenTSDB UID 编码的字节数，从3个字节到最大8个字节。这个改变需要在源码中修改相应的值、重新编译、部署所有需要访问这个数据的TDS，并且需要为未来所有版本保持这个定制的补丁。

.. Warning:: 有可能你的情况需要增加这个值。如果你选择来修改这个值,你必须开始用新的数据和一个新的UID表。任何数据写一个预计用3字节UID编码的TSD都将不兼容这一修改，所有确保所有的TSD运行相同修改的代码，并且在修改之前已经存储到OpenTSDB任何数据已经使用外部导出工具导出到一个地方。这个值的修改是在 ``TSDB.java`` 文件中。

**查询速度**

基数也会对查询速度有很大的影响,所以要考虑你会经常执行的查询以优化你的命名模式。OpenTSDB 为每一个小时创建新的一行时间序列。假如我们有时间序列 ``sys.cpu.user host=webserver01,cpu=0``，每秒都写入写一天，那么就会有24行的数据。然而如果我们再这个主机上有8核的CPU，我们一天就会有192行数据。这看起来不错，因为我们可以通过类似这样的查询 ``start=1d-ago&m=avg:sys.cpu.user{host=webserver01}`` 轻松获得合计或者平均数。

然而，假如我们有20,000台8核主机，现在我们每天将会有380万行这样一个较高基数的主机值。要查询 ``webserver01`` 主机的平均核心的使用情况会很慢，因为它会从380万行中挑出192行进行计算。

这种模式的好处是，假如需要存储每个核心的使用指标，你可以有很深的数据颗粒度，你还可以轻松的获取所有主机的所有核心的平均值：``start=1d-ago&m=avg:sys.cpu.user``。然而特定的指标将需要更长的查询时间，因为需要从更多的行中过滤。


下面是处理基数的常见方法：

**聚合前** - 在 ``sys.cpu.user`` 这个例子中, 你通常关心的主机平均使用情况,而不是每核心使用的使用情况。
虽然在上面的Tag模式中数据收集器可能会分散发送每个核的值，收集器也可以发送一个类似 ``sys.cpu.user.avg host=webserver01``额外的数据点。现在20k台主机每个每天有24行独立的时间序列，这样只有480k 行要筛选，查询每个主机的平均值更快，并且与单个核的数据完全独立开了。

**转变 metric** - 如果你真的只关心特定主机的metric而不是总的主机情况，这种情况下，你可以把hostname作为metric。再回到上面的例子 ``sys.cpu.user.websvr01 cpu=0`` ，这种模式查询非常快，每天只有192行metric。然而主机汇总则不得不在 OpenTSDB 之外做更多的查询和聚合。（未来将包含此功能）

命名经验总结
-----------------

在设计命名模式时，把这些铭记在心：

* 保持一致的命名，减少重复。总是使用相同的 metrics、 tag name 和 values 的名称。
* 每个metric使用相同数量和类型的tag，如不要用 ``my.metric host=foo`` 和 ``my.metric datacenter=lga``。
* 考虑你最常用的查询，以及根据你的查询优化模式
* 考虑查询时如何更深入
* 不要使用太多的Tag，保持在一个相对小的数量。通常4~5个tag（默认OpenTSDB最多支持8个Tag）。

数据规范
^^^^^^^^^^^^^^^^^^

每一个时间序列数据都需要以下几点组成：

* metric - 一个通用的时间序列名称，比如 ``sys.cpu.user``, ``stock.quote`` 或 ``env.probe.temp``.
* 时间戳 - Unix 时间戳秒和毫秒，仅支持正数的时间戳。
* 值 - 时间序列给定时间点存储的数值，支持整数和浮点数。
* Tag - 由 key/value  ``tagk``  和 ``tagv`` 键值对组成。每个数据点必须至少要有一个Tag

时间戳
----------

数据可以以秒或毫秒的精度写入 OpenTSDB。时间戳必须是不能大于13个数。毫秒时间戳必须以 ``1364410924250`` 这样的格式，最终以末尾以三个数字来标示毫秒。应用程序如果生成的时间戳多于13个数字（即：高于毫秒精度）必须最多截取前13个数字，否则会产生错误。

时间戳以秒的精度存储占用2个字节，如果以毫秒的精度存储占4个字节。因此如果你不需要毫秒精度或你所有的数据点的边界是1秒，我们推荐你在提交秒精度的10个数字的时间戳，这样你可以节省存储空间。这对于一个给定的时间序列来说避免混合使用秒和毫秒精度是有好处的。混合时间戳查询迭代时间要比只记录一种类型要快得多。


.. NOTE:: 当以telnet接口写入时，时间戳可以被写成 ``1364410924.250`` 点后三个数字代表毫秒。 通过 ``/api/put`` HTTP接口写入的时间戳必须是整数，不能带点符号。仅在通过  ``/api/query`` 接口和命令行时可以使用毫秒精度的时间戳。具体细节参考 :doc:`query/index` 。

.. NOTE:: 提供毫秒精度并不意味着 OpenTSDB 支持可以每毫秒一个数据点为很多时间序列快速写入。虽然一个TSD可以支持每秒上千的写入，这样假如你尝试每毫秒存储一个点这只会覆盖一些时间序列。相反，OpenTSDB 旨在提供更高的精度，同时还是要避免以这样的速度记录数据，特别是长时间运行的时间序列。

Metric 与 Tags
----------------

以下规则适用于 metric 和 tag 值:

* 字符串大小写敏感，比如"Sys.Cpu.User" 和 "sys.cpu.user" 会分别被存储
* 不允许有空格
* 只允许以下字符: ``a`` 到 ``z``, ``A`` 到 ``Z``, ``0`` 到 ``9``, ``-``, ``_``, ``.``, ``/`` 或 Unicode 字符 (按照规范)

虽然 Metric和Tag不限制长度，但还是应该保持短一些。

整数值
--------------

假如用 ``put`` 命令解析没有带小数点 (``.``)的值，它会被认为是有符号整型。整数存储，无符号，使用可白长度编码，一个数据点需要少则1字节，多则8字节存储空间。这个意思是一个暑假点可以有一个最小值 -9,223,372,036,854,775,808 和最大值  9,223,372,036,854,775,807（包含）。整数不能有逗号或其它不是数字和负号（负值）的字符。举个例子，为了存储最大值它必须在 ``9223372036854775807`` 之内提供。

浮点值
---------------------

如果从 ``put`` 命令解析出的值带有小数点 (``.``)，它将被认为是一个浮点类型的值。当前所有浮点值存储占用4个字节，单精度，未来的发行版计划支持8字节。浮点数字存储在 支持正负值的IEEE 754 浮点单一格式中。无穷和非数值是不支持的，如果提供给TSD会抛出一个错误。具体参考 `维基百科 <https://en.wikipedia.org/wiki/IEEE_floating_point>`_ 和 `Java 文档 <http://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.2.3>`_ 

排序
--------

与其它方案相比，OpenTSDB 允许写入任何你想要的时间序列数据。这使得写入数据到TSD中有非常好的灵活性，允许从你的系统填充当前的数据稍后再将历史数据导入。

.. WARNING:: 唯一要注意的是不能以一个不同的值来改写现有的值。写入是幂等的，这意味着你可以在时间戳 ``1356998400`` 把值 ``42`` 写入，然后再一次再在时间戳 ``1356998400`` 把值 ``42`` 写入，这什么都被不会发生。但是如果你想把 ``42.5`` 写入同一个时间戳这行数据将无效，并且任何包含这一行的查询都会抛出异常。如果有这种情况使用 ``fsck`` 工具可以修复这行。

输入方法
^^^^^^^^^^^^^

目前OpenTSDB有三种方法获取数据： Telnet API、HTTP API 和从文件批量导入。作为二选一，你可以使用一个提供OpenTSDB支持的工具或者非常冒险的使用Java库。

.. WARNING:: 不要试图直接写底层存储系统，比如HBase。真的，它很快会变地混乱。

Telnet
------

开始OpenTSDB最简单的方法是打开一个终端或telnet客户端，连接到你的TSD并使用 ``put`` 命令并按回车键。如果你正在写一个程序，只需打开一个socket，用一个新行打印字符串命令并且发送数据包。telnet命令的格式：

::

  put <metric> <timestamp> <value> <tagk1=tagv1[ tagk2=tagv2 ...tagkN=tagvN]>
  
举一个例子:

::

  put sys.cpu.user 1356998400 42.5 host=webserver01 cpu=0
 
每一个 ``put`` 只能发送一个数据点。在你的命令结尾不要忘了换行符，即： ``\n`` 。

Http API
--------

从2.0开始，数据通过HTTP发送的数据格式支持 'Serializer' 插件。多个相关数据点可以发送一个HTTP POST请求以节约带宽，详情查看 :doc:`../api_http/put` .

批量导入
------------

如果你从另一个系统导入数据或者回填历史数据，你可以使用 ``import`` 命令行工具。详情查看  :doc:`cli/import` 。

写入性能
^^^^^^^^^^^^^^^^^

OpenTSDB 可以扩展到在使用普通机械硬盘的商业服务器上每秒写入上百万数据点。然而用户启用一个虚拟机的单机模式的HBase，尝试数百万的数据点猛烈地写入一个TSD时只能每秒写入数百点数据而感到失望时，这时你就需要新安装或测试扩大现有系统规模。


UID 分配
--------------

大伙遇到的第一个关键点就是 ''uid 分配'' 。在数据点存储前，每一个metric字符串、tag key 和tag 值都必须分别一个UID。例如，metric ``sys.cpu.user`` 可以在首次被TSD遇到时分配一个UID  ``000001`` ，这个任务需要大量的时间，因为它必须取得一个可用的UID，写入一个UID到名称的映射和名称到UID的映射，然后使用UID写数据点的 rowkey。TSD将UID存储到TSD的缓存中，这样下次相同的metric 它很快就可以找到UID。

因此，我们推荐你在有很多metric、tag keys和tag值时可用预分配UID。如果你已经如上文所建议设计了一个命名方案，你就会知道大部分值的分配。你可以使用命令行工具 :doc:`cli/mkmetric`、 :doc:`cli/uid` 或HTTP API :doc:`../api_http/uid/index` 执行预分配。任何时候你要发送一批新的metric或tag到运行OpenTSDB的集群，当得到新数据试图预分配或者TSD被拖垮。

.. NOTE:: 如果你重启一个TSD，它会查找每一个metric和tag的UID，这样会有点慢，直到缓存被填充。

预先分隔 HBase Regions
-----------------------

For brand new installs you will see much better performance if you pre-split the regions in HBase regardless of if you're testing on a stand-alone server or running a full cluster. HBase regions handle a defined range of row keys and are essentially a single file. When you create the ``tsdb`` table and start writing data for the first time, all of those data points are being sent to this one file on one server. As a region fills up, HBase will automatically split it into different files and move it to other servers in the cluster, but when this happens, the TSDs cannot write to the region and must buffer the data points. Therefore, if you can pre-allocate a number of regions before you start writing, the TSDs can send data to multiple files or servers and you'll be taking advantage of the linear scalability immediately. 

The simplest way to pre-split your ``tsdb`` table regions is to estimate the number of unique metric names you'll be recording. If you have designed a naming schema, you should have a pretty good idea. Let's say that we will track 4,000 metrics in our system. That's not to say 4,000 time series, as we're not counting the tags yet, just the metric names such as "sys.cpu.user". Data points are written in row keys where the metric's UID comprises the first bytes, 3 bytes by default. The first metric will be assigned a UID of ``000001`` as a hex encoded value. The 4,000th metric will have a UID of ``000FA0`` in hex. You can use these as the start and end keys in the script from the `HBase Book <http://hbase.apache.org/book/perf.writing.html>`_ to split your table into any number of regions. 256 regions may be a good place to start depending on how many time series share each metric.

TODO - include scripts for pre-splitting.

The simple split method above assumes that you have roughly an equal number of time series per metric (i.e. a fairly consistent cardinality). E.g. the metric with a UID of ``000001`` may have 200 time series and ``000FA0`` has about 150. If you have a wide range of time series per metric, e.g. ``000001`` has 10,000 time series while ``000FA0`` only has 2, you may need to develop a more complex splitting algorithm.

But don't worry too much about splitting. As stated above, HBase will automatically split regions for you so over time, the data will be distributed fairly evenly.

分布式 HBase
-----------------

HBase will run in stand-alone mode where it will use the local file system for storing files. It will still use multiple regions and perform as well as the underlying disk or raid array will let it. You'll definitely want a RAID array under HBase so that if a drive fails, you can replace it without losing data. This kind of setup is fine for testing or very small installations and you should be able to get into the low thousands of data points per second.

However if you want serious throughput and scalability you have to setup a Hadoop and HBase cluster with multiple servers. In a distributed setup HDFS manages region files, automatically distributing copies to different servers for fault tolerance. HBase assigns regions to different servers and OpenTSDB's client will send data points to the specific server where they will be stored. You're now spreading operations amongst multiple servers, increasing performance and storage. If you need even more throughput or storage, just add nodes or disks.

There are a number of ways to setup a Hadoop/HBase cluster and a ton of various tuning tweaks to make, so Google around and ask user groups for advice. Some general recomendations include:

* Dedicate a pair of high memory, low disk space servers for the Name Node. Set them up for high availability using something like Heartbeat and Pacemaker.
* Setup Zookeeper on at least 3 servers for fault tolerance. They must have a lot of RAM and a fairly fast disk for log writing. On small clusters, these can run on the Name node servers.
* JBOD for the HDFS data nodes
* HBase region servers can be collocated with the HDFS data nodes
* At least 1 gbps links between servers, 10 gbps preferable.
* Keep the cluster in a single data center

多个 TSD
-------------

单一的TSD每秒可以处理数千写入，但是你要是有多个数据源最好运行多个TSD，使用复制均衡器（如 Varnish或DNS轮询调度）分发写入。很多用户将OpenTSDB 集群的TSD部署在HBase region 服务器上。

长连接
----------------------

在TSD中启用长连接，并确保任何应用程序使用长连接发送时间序列数据，而不是每次打开关闭连接。具体参考 :doc:`configuration` 。

禁用元数据和实时发布
------------------------------------------

OpenTSDB 2.0 引入元数据用来跟踪系统中的各种类型数据。当跟踪启用时，计数器递增为每一个数据点写入新的UID或时间序列生成元数据，该数据可以被推倒一个搜索引擎或者通过tree生成代码。这些方法需要TSD有跟多的内存，并且会影响吞吐量。默认情况下禁用跟踪，测试之前启用该功能。

2.0还引用了实时发布插件，当数据点传入可以发送到另一目标立即排队存储。这是默认禁用的，任何插件在部署到生产环境之前要测试好。
