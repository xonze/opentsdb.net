配置
-------------

配置OpenTSDB可以通过本地系统文件，也可以通过命令行参数或组合，或者两者混合。

配置文件
^^^^^^^^^^^^^^^^^^

配置文件符合Java配置规范。配置名称使用没有空格的小写、带点的字符串。每个名称后面是一个等号，之后是该属性的值。所有的属性都是以``tsd.``开始，无效的配置行用 ``#``符号注释。例如::


  # List of Zookeeper hosts that manage the HBase cluster
  tsd.storage.hbase.zk_quorum = 192.168.1.100
  
这将配置TSD连接到 ``192.168.1.100`` 的Zookeeper。

当混合使用配置文件盒命令行参数，其处理顺序如下：

#. 加载默认值
#. 加载配置文件的值, 覆盖默认值
#. 加载命令行参数，覆盖配置文件盒默认值

文件位置
^^^^^^^^^^^^^^

可以使用 ``--config`` 命令行参数指定配置文件的完整路径，假如没有指定的话 OpenTSDB 和它的命令行工具将尝试从以下目录搜索有效的配置文件：

* ./opentsdb.conf
* /etc/opentsdb.conf
* /etc/opentsdb/opentsdb.conf
* /opt/opentsdb/opentsdb.conf

在不能发现一个有效的配置文件或者必选的参数没有设置TSD都无法启动。请参阅下面必选配置和属性列表。

属性
^^^^^^^^^^

下面的表格是所有工具的可选配置项，在适用场景下，符合命令行提供覆盖。请注意单独的命令行工具可能会有不同的值，因此要仔细查看他们的文档细节。

.. csv-table::
   :header: "属性", "类型", "必要性", "描述", "默认", "命令行"
   :widths: 20, 5, 10, 50, 5, 10

   "tsd.core.auto_create_metrics", "Boolean", "可选", "无论是否新的，如果传入的metric不存在就自动创建一个新的UID。如果 false, 在写入数据时 metric 没有匹配到已存在的 UID 将会被拒绝写入. Tag 名称和 Tag 值总是会被自动创建的.", "False", "--auto-metric"
   "tsd.core.meta.enable_realtime_ts", "Boolean", "可选", "是否启用 TSMeta 对象实时创建。 See :doc:`../user_guide/metadata`", "False", ""
   "tsd.core.meta.enable_realtime_uid", "Boolean", "可选", "是否启用 UIDMeta 对象实时创建。. See :doc:`../user_guide/metadata`", "False", ""
   "tsd.core.meta.enable_tsuid_incrementing", "Boolean", "可选", "是否启用自增计数器跟踪每一个 TSUIDs 记录. See :doc:`../user_guide/metadata` (覆盖 ""tsd.core.meta.enable_tsuid_tracking"")", "False", ""
   "tsd.core.meta.enable_tsuid_tracking", "Boolean", "可选", "Whether or not to enable tracking of TSUIDs by storing a ``1`` with the current timestamp every time a data point is recorded. See :doc:`../user_guide/metadata`", "False", ""
   "tsd.core.plugin_path", "String", "可选", "TSD启动时会搜索插件目录，如果目录无效TSD将不能启动。 Plugins can still be enabled if they are in the class path.", "", ""
   "tsd.core.timezone", "String", "可选", "执行查询时，转换绝对时间为UTC时用来覆盖本地系统时区本地化时区标识字符串使用。这并不会影响输入数据时间戳。
   E.g. America/Los_Angeles", "系统配置", ""
   "tsd.core.tree.enable_processing", "Boolean", "可选", "是否启用通过树规则集处理新建和编辑 TSMeta", "false", ""
   "tsd.http.cachedir", "String", "必选", "临时文件写入的完整路径。
   E.g. /tmp/opentsdb", "", "--cachedir"
   "tsd.http.request.cors_domains", "String", "可选", "用于配置跨域请求（CORS）指定域名列表。A comma separated list of domain names to allow access to OpenTSDB when the ``Origin`` header is specified by the client. If empty, CORS requests are passed through without validation. The list may not contain the public wildcard ``*`` and specific domains at the same time.", "", ""
   "tsd.http.request.enable_chunked", "Boolean", "可选", "是否启用HTTP RPC chunk支持", "false", ""
   "tsd.http.request.max_chunk", "Integer", "可选", "接收HTTP请求的最大块", "4096", ""
   "tsd.http.show_stack_trace", "Boolean", "可选", "是否在API调用发生异常时返回trace信息", "false", ""
   "tsd.http.staticroot", "String", "必选", "本地静态文件路径，例如web界面所用到的JavaScript.
   E.g. /opt/opentsdb/staticroot", "", "--staticroot"
   "tsd.network.async_io", "Boolean", "可选", "使用非阻塞 NIO 或者传统的阻塞IO", "True", "--async-io"
   "tsd.network.backlog", "Integer", "可选", "完成或未完成的连接请求队列深度取决于操作系统。 默认取内核kernel的'somaxconn' 设置，或把Netty设置为3072.", "See Description", "--backlog"
   "tsd.network.bind", "String", "可选", "绑定一个IPv4地址，默认是监听所有IP
   E.g. 127.0.0.1", "0.0.0.0", "--bind"
   "tsd.network.keep_alive", "Boolean", "可选", "是否允许长连接", "True", ""
   "tsd.network.port", "Integer", "必选", "TCP端口", "", "--port"
   "tsd.network.reuse_address", "Boolean", "可选", "是否启用绑定复用 Netty 端口", "True", ""
   "tsd.network.tcp_no_delay", "Boolean", "可选", "是否在传送数据之前禁用TCP缓冲", "True", ""
   "tsd.network.worker_threads", "Integer", "可选", "Netty异步IO线程数", "*#CPU cores \* 2*", "--worker-threads"
   "tsd.rpc.plugins", "String", "可选", "TSD启动时加载的，以逗号风格的 RPC 插件列表. 必须是完整的class名称。", "", ""
   "tsd.rtpublisher.enable", "Boolean", "可选", "是否启用实时发布插件。如果为true，则必须提供一个有效的 ``tsd.rtpublisher.plugin`` class 名称", "False", ""
   "tsd.rtpublisher.plugin", "String", "可选", "实时发布插件的实例 class 名称。假如 ``tsd.rtpublisher.enable`` 设置为 false, 该项将忽略。
   E.g. net.opentsdb.tsd.RabbitMQPublisher", "", ""
   "tsd.search.enable", "Boolean", "可选", "是否启用搜索功能， 如果为 true, 你必须提供一个有效的 ``tsd.search.plugin`` class 名称", "False", ""
   "tsd.search.plugin", "String", "可选", "搜索插件的class 名称. 如果 ``tsd.search.enable`` 为False时，该项将忽略。
   E.g. net.opentsdb.search.ElasticSearch", "", ""
   "tsd.stats.canonical", "Boolean", "可选", "Whether or not the FQDN should be returned with statistics requests. The default stats are returned with ``host=<hostname>`` which is not guaranteed to perform a lookup and return the FQDN. Setting this to true will perform a name lookup and return the FQDN if found, otherwise it may return the IP. The stats output should be ``fqdn=<hostname>``", "false", ""
   "tsd.storage.enable_compaction", "Boolean", "可选", "是否启用压缩", "True", ""
   "tsd.storage.flush_interval", "Integer", "可选", "多长时间将新的数据Flush一次，以毫秒为单位", "1000", "--flush-interval"
   "tsd.storage.hbase.data_table", "String", "可选", "数据存储在HBase里的表名称", "tsdb", "--table"
      "tsd.storage.hbase.meta_table", "String", "可选", "存储元数据HBase的表名称", "tsdb-meta", ""
   "tsd.storage.hbase.tree_table", "String", "可选", "存储树数据HBase的表名称", "tsdb-tree", ""
   "tsd.storage.hbase.uid_table", "String", "可选", "存储UID信息的HBase表名称", "tsdb-uid", "--uidtable"
   "tsd.storage.hbase.zk_basedir", "String", "可选", " -ROOT- region 在 znode 里的位置", "/hbase", "--zkbasedir"
   "tsd.storage.hbase.zk_quorum", "String", "可选", "用逗号分隔的ZooKeeper主机列表，可以指定端口。
   E.g. ``192.168.1.1:2181, 192.168.1.2:2181``", "localhost", "--zkquorum"
   
数据类型
^^^^^^^^^^

有些配置值需要特别考虑:

* Booleans - 下面的文字将解析为 ``True``:

  * ``1``
  * ``true``
  * ``yes``
  
  其它值都被认为是 ``False``。 解析不区分大小写
  
* Strings - 字符串， 即使是有空格也不是必须使用引号， 但一些注意事项应用:

  * 特殊字符必须用一个反斜杠转义: ``#``, ``!``, ``=``, and ``:``
    E.g.::
    
      my.property = Hello World\!
      
  * Unicode 字符必须转义它们的十六进制标示方法, e.g.::
  
      my.property = \u0009
