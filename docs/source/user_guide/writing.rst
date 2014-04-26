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

A critical aspect of any naming schema is to consider the cardinality of your time series. Cardinality is defined as the number of unique items in a set. In OpenTSDB's case, this means the number of items associated with a metric, i.e. all of the possible tag name and value combinations, as well as the number of unique metric names, tag names and tag values. Cardinality is important for two reasons outlined below.

**唯一 ID (UIDs)的限制** 

There is a limited number of unique IDs to assign for each metric, tag name and tag value. By default there are just over 16 million possible IDs per type. If, for example, you ran a very popular web service and tried to track the IP address of clients as a tag, e.g. ``web.app.hits clientip=38.26.34.10``, you may quickly run into the UID assignment limit as there are over 4 billion possible IP version 4 addresses. Additionally, this approach would lead to creating a very sparse time series as the user at address ``38.26.34.10`` may only use your app sporadically, or perhaps never again from that specific address.

The UID limit is usually not an issue, however. A tag value is assigned a UID that is completely disassociated from its tag name. If you use numeric identifiers for tag values, the number is assigned a UID once and can be used with many tag names. For example, if we assign a UID to the number ``2``, we could store timeseries with the tag pairs ``cpu=2``, ``interface=2``, ``hdd=2`` and ``fan=2`` while consuming only 1 tag value UID (``1``) and 4 tag name UIDs (``cpu``, ``interface``, ``hdd`` and ``fan``).

If you think that the UID limit may impact you, first think about the queries that you want to execute. If we look at the ``web.app.hits`` example above, you probably only care about the total number of hits to your service and rarely need to drill down to a specific IP address. In that case, you may want to store the IP address as an annotation. That way you could still benefit from low cardinality but if you need to, you could search the results for that particular IP using external scripts. (Note: Support for annotation queries is expected in a *future* version of OpenTSDB.)

If you desperately need more than 16 million values, you can increase the number of bytes that OpenTSDB uses to encode UIDs from 3 bytes up to a maximum of 8 bytes. This change would require modifying the value in source code, recompiling, deploying your customized code to all TSDs which will access this data, and maintaining this customization across all future patches and releases.

.. Warning:: It is possible that your situation requires this value to be increased.  If you choose to modify this value, you must start with fresh data and a new UID table. Any data written with a TSD expecting 3-byte UID encoding will be incompatible with this change, so ensure that all of your TSDs are running the same modified code and that any data you have stored in OpenTSDB prior to making this change has been exported to a location where it can be manipulated by external tools.  See the ``TSDB.java`` file for the values to change.

**查询速度**

Cardinality also affects query speed a great deal, so consider the queries you will be performing frequently and optimize your naming schema for those. OpenTSDB creates a new row per time series per hour. If we have the time series ``sys.cpu.user host=webserver01,cpu=0`` with data written every second for 1 day, that would result in 24 rows of data. However if we have 8 possible CPU cores for that host, now we have 192 rows of data. This looks good because we can get easily a sum or average of CPU usage across all cores by issuing a query like ``start=1d-ago&m=avg:sys.cpu.user{host=webserver01}``.

However what if we have 20,000 hosts, each with 8 cores? Now we will have 3.8 million rows per day due to a high cardinality of host values. Queries for the average core usage on host ``webserver01`` will be slower as it must pick out 192 rows out of 3.8 million. 

The benefits of this schema are that you have very deep granularity in your data, e.g., storing usage metrics on a per-core basis. You can also easily craft a query to get the average usage across all cores an all hosts: ``start=1d-ago&m=avg:sys.cpu.user``. However queries against that particular metric will take longer as there are more rows to sift through.  

Here are some common means of dealing with cardinality:

**Pre-Aggregate** - In the example above with ``sys.cpu.user``, you generally care about the average usage on the host, not the usage per core. While the data collector may send a separate value per core with the tagging schema above, the collector could also send one extra data point such as ``sys.cpu.user.avg host=webserver01``. Now you have a completely separate timeseries that would only have 24 rows per day and with 20K hosts, only 480K rows to sift through. Queries will be much more responsive for the per-host average and you still have per-core data to drill down to separately.

**Shift to Metric** - What if you really only care about the metrics for a particular host and don't need to aggregate across hosts? In that case you can shift the hostname to the metric. Our previous example becomes ``sys.cpu.user.websvr01 cpu=0``. Queries against this schema are very fast as there would only be 192 rows per day for the metric. However to aggregate across hosts you would have to execute mutliple queries and aggregate outside of OpenTSDB. (Future work will include this capability).

命名经验总结
-----------------

在设计命名模式时，把这些铭记在心：

* Be consistent with your naming to reduce duplication. Always use the same case for metrics, tag names and values.
* Use the same number and type of tags for each metric. E.g. don't store ``my.metric host=foo`` and ``my.metric datacenter=lga``.
* Think about the most common queries you'll be executing and optimize your schema for those queries
* Think about how you may want to drill down when querying
* Don't use too many tags, keep it to a fairly small number, usually up to 4 or 5 tags (By default, OpenTSDB supports a maximum of 8 tags).

数据规范
^^^^^^^^^^^^^^^^^^

Every time series data point requires the following data:

* metric - A generic name for the time series such as ``sys.cpu.user``, ``stock.quote`` or ``env.probe.temp``.
* timestamp - A Unix/POSIX Epoch timestamp in seconds or milliseconds defined as the number of seconds that have elapsed since January 1st, 1970 at 00:00:00 UTC time. Only positive timestamps are supported at this time.
* value - A numeric value to store at the given timestamp for the time series. This may be an integer or a floating point value.
* tag(s) - A key/value pair consisting of a ``tagk`` (the key) and a ``tagv`` (the value). Each data point must have at least one tag.

时间戳
----------

Data can be written to OpenTSDB with second or millisecond resolution. Timestamps must be integers and be no longer than 13 digits (See first [NOTE] below).  Millisecond timestamps must be of the format ``1364410924250`` where the final three digits represent the milliseconds.  Applications that generate timestamps with more than 13 digits (i.e., greater than millisecond resolution) must be rounded to a maximum of 13 digits before submitting or an error will be generated.

Timestamps with second resolution are stored on 2 bytes while millisecond resolution are stored on 4. Thus if you do not need millisecond resolution or all of your data points are on 1 second boundaries, we recommend that you submit timestamps with 10 digits for second resolution so that you can save on storage space. It's also a good idea to avoid mixing second and millisecond timestamps for a given time series. Doing so will slow down queries as iteration across mixed timestamps takes longer than if you only record one type or the other. OpenTSDB will store whatever you give it.

.. NOTE:: When writing to the telnet interface, timestamps may optionally be written in the form ``1364410924.250``, where three digits representing the milliseconds are placed after a period.  Timestamps sent to the ``/api/put`` endpoint over HTTP *must* be integers and may not have periods. Data with millisecond resolution can only be extracted via the ``/api/query`` endpoint or CLI command at this time. See :doc:`query/index` for details.

.. NOTE:: Providing millisecond resolution does not necessarily mean that OpenTSDB supports write speeds of 1 data point per millisecond over many time series. While a single TSD may be able to handle a few thousand writes per second, that would only cover a few time series if you're trying to store a point every millisecond. Instead OpenTSDB aims to provide greater measurement accuracy and you should generally avoid recording data at such a speed, particularly for long running time series.

Metrics 与 Tags
----------------

The following rules apply to metric and tag values:

* Strings are case sensitive, i.e. "Sys.Cpu.User" will be stored separately from "sys.cpu.user"
* Spaces are not allowed
* Only the following characters are allowed: ``a`` to ``z``, ``A`` to ``Z``, ``0`` to ``9``, ``-``, ``_``, ``.``, ``/`` or Unicode letters (as per the specification)

Metric and tags are not limited in length, though you should try to keep the values fairly short.

整数值
--------------

If the value from a ``put`` command is parsed without a decimal point (``.``), it will be treated as a signed integer. Integers are stored, unsigned, with variable length encoding so that a data point may take as little as 1 byte of space or up to 8 bytes. This means a data point can have a minimum value of -9,223,372,036,854,775,808 and a maximum value of 9,223,372,036,854,775,807 (inclusive). Integers cannot have commas or any character other than digits and the dash (for negative values).  For example, in order to store the maximum value, it must be provided in the form ``9223372036854775807``.

浮点值
---------------------

If the value from a ``put`` command is parsed with a decimal point (``.``) it will be treated as a floating point value. Currently all floating point values are stored on 4 bytes, single-precision, with support for 8 bytes planned for a future release.  Floats are stored in IEEE 754 floating-point "single format" with positive and negative value support.  Infinity and Not-a-Number values are not supported and will throw an error if supplied to a TSD. See `Wikipedia <https://en.wikipedia.org/wiki/IEEE_floating_point>`_ and the `Java Documentation <http://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.2.3>`_ for details.

排序
--------

Unlike other solutions, OpenTSDB allows for writing data for a given time series in any order you want.  This enables significant flexibility in writing data to a TSD, allowing for populating current data from your systems, then importing historical data at a later time. 

.. WARNING:: The only caveat when writing is that you cannot overwrite an existing value with a different value. Writing is idempotent, meaning you can write the value ``42`` at timestamp ``1356998400`` and then write ``42`` again for the same time, nothing bad will happen. However if you try to write ``42.5`` to the same timestamp, the row of data will become invalid (due to vagaries of the underlying schema) and any queries that include that row will throw an exception. Use the ``fsck`` utility to fix the row if this happens.

输入方法
^^^^^^^^^^^^^

There are currently three main methods to get data into OpenTSDB: Telnet API, HTTP API and batch import from a file. Alternatively you can use a tool that provides OpenTSDB support, or if you're extremely adventurous, use the Java library. 

.. WARNING:: Don't try to write directly to the underlying storage system, e.g. HBase. Just don't. It'll get messy quickly.

Telnet
------

The easiest way to get started with OpenTSDB is to open up a terminal or telnet client, connect to your TSD and issue a ``put`` command and hit 'enter'. If you are writing a program, simply open a socket, print the string command with a new line and send the packet. The telnet command format is:

::

  put <metric> <timestamp> <value> <tagk1=tagv1[ tagk2=tagv2 ...tagkN=tagvN]>
  
For example:

::

  put sys.cpu.user 1356998400 42.5 host=webserver01 cpu=0
 
Each ``put`` can only send a single data point. Don't forget the newline character, e.g. ``\n`` at the end of your command.

Http API
--------

As of version 2.0, data can be sent over HTTP in formats supported by 'Serializer' plugins. Multiple, un-related data points can be sent in a single HTTP POST request to save bandwidth. See the :doc:`../api_http/put` for details.

批量导入
------------

If you are importing data from another system or you need to backfill historical data, you can use the ``import`` CLI utility. See :doc:`cli/import` for details.

Write Performance
^^^^^^^^^^^^^^^^^

OpenTSDB can scale to writing millions of data points per 'second' on commodity servers with regular spinning hard drives. However users who fire up a VM with HBase in stand-alone mode and try to slam millions of data points at a brand new TSD are disappointed when they can only write data in the hundreds of points per second. Here's what you need to do to scale for brand new installs or testing and for expanding existing systems.

UID 分配
--------------

The first sticking point folks run into is ''uid assignment''. Every string for a metric, tag key and tag value must be assigned a UID before the data point can be stored. For example, the metric ``sys.cpu.user`` may be assigned a UID of ``000001`` the first time it is encountered by a TSD. This assignment takes a fair amount of time as it must fetch an available UID, write a UID to name mapping and a name to UID mapping, then use the UID to write the data point's row key. The UID will be stored in the TSD's cache so that the next time the same metric comes through, it can find the UID very quickly.

Therefore, we recommend that you 'pre-assign' UID to as many metrics, tag keys and tag values as you can. If you have designed a naming schema as recommended above, you'll know most of the values to assign. You can use the CLI tools :doc:`cli/mkmetric`, :doc:`cli/uid` or the HTTP API :doc:`../api_http/uid/index` to perform pre-assignments. Any time you are about to send a bunch of new metrics or tags to a running OpenTSDB cluster, try to pre-assign or the TSDs will bog down a bit when they get the new data.

.. NOTE:: If you restart a TSD, it will have to lookup the UID for every metric and tag so performance will be a little slow until the cache is filled.

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

A single TSD can handle thousands of writes per second. But if you have many sources it's best to scale by running multiple TSDs and using a load balancer (such as Varnish or DNS round robin) to distribute the writes. Many users colocate TSDs on their HBase region servers when the cluster is dedicated to OpenTSDB. 

长连接
----------------------

Enable keep alives in the TSDs and make sure that any applications you are using to send time series data keep their connections open instead of opening and closing for every write. See :doc:`configuration` for details.

禁用元数据和实时发布
------------------------------------------

OpenTSDB 2.0 introduced meta data for tracking the kinds of data in the system. When tracking is enabled, a counter is incremented for every data point written and new UIDs or time series will generate meta data. The data may be pushed to a search engine or passed through tree generation code. These processes require greater memory in the TSD and may affect throughput. Tracking is disabled by default so test it out before enabling the feature.

2.0 also introduced a real-time publishing plugin where incoming data points can be emitted to another destination immediately after they're queued for storage. This is diabled by default so test any plugins you are interested in before deploying in production.
