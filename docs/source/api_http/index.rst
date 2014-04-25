HTTP API
========

OpenTSDB provides an HTTP based application programming interface to enable integration with external systems. Almost all OpenTSDB features are accessiable via the API such as querying timeseries data, managing metadata and storing data points. Please read this entire page for important information about standard API behavior before investigating individual endpoints.
OpenTSDB提供了一个基于HTTP的应用程序接口来实现与外部系统的集成。通过API可以很容易实现几乎所有OpenTSDB功能，如查询的时间序列数据，管理元数据和存储数据点，了解单个端点之前，请阅读整个页面，了解API标准用法的重要信息。
Overview
--------

The HTTP API is RESTful in nature but provides alternative access through various overrides since not all clients can adhere to a strict REST protocol. The default data exchange is via JSON though plugable ``formatters`` may be accessed, via the request, to send or receive data in different formats. Standard HTTP response codes are used for all returned results and errors will be returned as content using the proper format.
HTTP API本质上是满足REST协议（约束）的，但是它也提供了一个可以各种重写的方法，然而不是所有的客户端能够坚持严格的REST协议。 默认的数据交换是通过格式化的JSON进行的，访问该请求，发送或接收不同格式的数据。标准的HTTP响应代码用于将所有的结果和错误内容以正确的格式返回。
Version 1.X to 2.x
------------------

OpenTSDB 1.x had a simple HTTP API that provided access to common behaviors such as querying for data, auto-complete queries and static file requests. OpenTSDB 2.0 introduces a new, formalized API as documented here. The 1.0 API is still accessible though most calls are deprecated and may be removed in version 3. All 2.0 API calls start with ``/api/``.
OpenTSDB 1.x中已经提供一个简单的HTTP API来实现一些普遍的用法，如查询数据，自动完成查询和静态文件请求。OpenTSDB 2.0引入了一个新的，形式化的API文档。1.0 API是仍然可以访问，虽然大多数调用已被弃用，并可能在第3版中删除。所有2.0 API调用以/ API /开始。
Serializers
-----------

2.0 introduces plugable serializers that allow for parsing user input and returning results in different formats such as XML or JSON. Serializers only apply to the 2.0 API calls, all 1.0 behave as before. For details on Serializers and options supported, please read :doc:`serializers/index`
2.0引入plugable serializers，这就允许用户以不同格式输入、返回结果，如XML或JSON。序列化只适用于2.0 API调用，所有1.0使用和以前保持不变。有关serializers和支持选项的详细信息，请阅读:doc:`serializers/index`

All API calls use the default JSON serializer unless overridden by query string or ``Content-Type`` header. To override:
所有的API调用使用默认的JSON序列化，除非被‘query string’或者‘Content-Type’header重写。要覆盖：

* **Query String** - Supply a parameter such as ``serializer=<serializer_name>`` where ``<serializer_name>`` is the hard-coded name of the serializer as shown in the ``/api/serializers`` ``serializer`` output field.
  * **Query String** 提供了一个参数  像``serializer=<serializer_name>``，  ``<serializer_name>``是 ``/api/serializers` ``serializer``输出字段的编码格式名称
  .. WARNING:: If a serializer isn't found that matches the ``<serializer_name>`` value, the query will return an error instead of processing further.
   警告：如果没有找到一个serializer与<serializer_name>相匹配的值，则查询将返回一个错误，而不是进一步的处理。
  
* **Content-Type** - If a query string is not given, the TSD will parse the ``Content-Type`` header from the HTTP request. Each serializer may supply a content type and if matched to the incoming request, the proper serializer will be used. If a serializer isn't located that maps to the content type, the default serializer will be used.
* **Content-Type** - 如果一个‘query string ’没有给出，TSD将从HTTP请求中解析的Content-Type头。每个serializer可提供一个内容类型，如果找到和传入请求相匹配serializer，这个serializer将被使用。如果没有匹配到serializer，默认的serializer将被使用。
* **Default** - If no query string parameter is given or the content-type is missing or not matched, the default JSON serializer will be used.
* **默认**  -如果没有指定‘query string ’参数或content-type 缺少或不匹配，则默认JSON serializer将被使用。
The API documentation will display requests and responses using the JSON serializer. See plugin documentation for the ways in which serializers alter behavior.
API文档会使用JSON serializers去显示请求和响应。见插件文档中改变serializers方式。
.. NOTE:: The JSON specification states that fields can appear in any order, so do not assume the ordering in given examples will be preserved. Arrays may be sorted and if so, this will be documented.
JSON规范中fields没有固定的顺序，所有不一定要采用例子中所保留下来的顺序。即便数组可以排序，也将会在文档中有所记录。

Authentication/Permissions
--------------------------

As of yet, OpenTSDB lacks an authentication and access control system.
Therefore no authentication is required when accessing the API. If you wish to
limit access to OpenTSDB, use network ACLs or firewalls to block access.
We do not recommend running OpenTSDB on a machine with a public IP Address.
OpenTSDB缺少一个权限或者访问控制系统，因此无需任何权限就可以访问API，
如果你希望限制OpenTSDB的访问，可以用网络ACL或者防火墙去限制访问。
我们不推荐使用用公网ip的机器去运行OpenTSDB。
Response Codes
响应代码
--------------

Every request will be returned with a standard HTTP response code. Most responses will include content, particularly error codes that will include details in the body about what went wrong. Successful codes returned from the API include:
每个请求将会返回一个标准的http响应代码。大部分的反应将包括内容，特别是错误代码，将会包括出错部分的详细信息。从API返回成功的代码包括：
.. csv-table::
   :header: "Code", "Description"
   :widths: 10, 90
   
   "200", "The request completed successfully请求成功完成"
   "204", "The server has completed the request successfully but is not returning content in the body. This is primarily used for storing data points as it is not necessary to return data to caller服务器完成请求成功但没有返回内容。这是主要用于存储数据点，因为它是没有必要将数据返回给调用者"
   "301", "This may be used in the event that an API call has migrated or should be forwarded to another server 找不到API call"
   
Common error response codes include常见的错误代码包括:

.. csv-table::
   :header: "Code", "Description"
   :widths: 10, 90
   
   "400", "Information provided by the API user, via a query string or content data, was in error or missing. This will usually include information in the error body about what parameter caused the issue. Correct the data and try again.
            查询条件或者内容错误或丢失，会有错误提示信息，更正再试一次"
   "404", "The requested endpoint or file was not found. This is usually related to the static file endpoint.请求端点或者文件不存在，一般是存储文件的端点的问题"
   "405", "The requested verb or method was not allowed. Please see the documentation for the endpoint you are attempting to access请求的连接或者方法不允许访问，请查看你尝试连接端点的文档"
   "406", "The request could not generate a response in the format specified. For example, if you ask for a PNG file of the ``logs`` endpoing, you will get a 406 response since log entries cannot be converted to a PNG image (easily)访问文件格式不匹配"
   "408", "The request has timed out. This may be due to a timeout fetching data from the underlying storage system or other issues请求超时"
   "413", "The results returned from a query may be too large for the server's buffers to handle. This can happen if you request a lot of raw data from OpenTSDB. In such cases break your query up into smaller queries and run each individually超出缓存"
   "500", "An internal error occured within OpenTSDB. Make sure all of the systems OpenTSDB depends on are accessible and check the bug list for issues OpenTSDB内部错误"
   "501", "The requested feature has not been implemented yet. This may appear with formatters or when calling methods that depend on plugins请求未被执行"
   "503", "A temporary overload has occurred. Check with other users/applications that are interacting with OpenTSDB and determine if you need to reduce requests or scale your system.临时过载"
   
Errors错误
------

If an error occurs, the API will return a response with an error object formatted per the requested response type. Error object fields include:
如果发生错误，API将返回一个与请求响应相匹配的error 对象，Error对象fields包括：
.. csv-table::
   :header: "Field Name", "Data Type", "Always Present", "Description描述", "Example"
   :widths: 10, 10, 10, 50, 20
   
   "code", "Integer", "Yes", "The HTTP status code ", "400"
   "message", "String", "Yes", "A descriptive error message about what went ", "Missing required parameter"
   "details", "String", "Optional", "Details about the error, often a stack trace", "Missing value: type"
   "trace", "String", "Optional", "A JAVA stack trace describing the location where the error was generated. This can be enabled via the ``tsd.http.show_stack_trace`` configuration option. The default for TSD is to hide the stack trace.", "`See below`"

All errors will return with a valid HTTP status error code and a content body with error details. The default formatter returns error messages as JSON with the ``application/json`` content-type. If a different formatter was requested, the output may be different. See the formatter documentation for details.
   所有的错误都将返回一个有效的HTTP状态错误代码和错误的详细信息。默认的格式化程序返回错误消息为JSON格式``application/json``的内容类型。如果被请求的格式不用，输出可以是不同的。请参阅 formatter documentation。
Example Error Result
^^^^^^^^^^^^^^^^^^^^
.. code-block :: javascript

  {
      "error": {
          "code": 400,
          "message": "Missing parameter <code>type</code>",
          "trace": "net.opentsdb.tsd.BadRequestException: Missing parameter <code>type</code>\r\n\tat net.opentsdb.tsd.BadRequestException.missingParameter(BadRequestException.java:78) ~[bin/:na]\r\n\tat net.opentsdb.tsd.HttpQuery.getRequiredQueryStringParam(HttpQuery.java:250) ~[bin/:na]\r\n\tat net.opentsdb.tsd.SuggestRpc.execute(SuggestRpc.java:63) ~[bin/:na]\r\n\tat net.opentsdb.tsd.RpcHandler.handleHttpQuery(RpcHandler.java:172) [bin/:na]\r\n\tat net.opentsdb.tsd.RpcHandler.messageReceived(RpcHandler.java:120) [bin/:na]\r\n\tat org.jboss.netty.channel.SimpleChannelUpstreamHandler.handleUpstream(SimpleChannelUpstreamHandler.java:75) [netty-3.5.9.Final.jar:na]\r\n\tat org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:565) [netty-3.5.9.Final.jar:na]
          ....\r\n\tat java.lang.Thread.run(Unknown Source) [na:1.6.0_26]\r\n"
      }
  }
  
Note that the stack trace is truncated. Also, the trace will include system specific line endings (in this case ``\r\n`` for Windows). If displaying for a user or writing to a log, be sure to replace the ``\n`` or ``\r\n`` and ``\r`` characters with new lines and tabs.
请注意，堆栈跟踪将被截断。此外，跟踪将包括系统特定的行结束符（在这种情况下\ r \ n适用于Windows）。如果显示用户或写入日志，一定要更换\ n或\ r \ n和\ r新行和制表符字符。
Verbs请求动作
-----
   
The HTTP API is RESTful in nature, meaning it does it's best to adhere to the REST protocol by using HTTP verbs to determine a course of action. For example, a ``GET`` request should only return data, a ``PUT`` or ``POST`` should modify data and ``DELETE`` should remove it. Documentation will reflect what verbs can be used on an endpoint and what they do. 
http是遵守rest协议的，这就意味着使用http请求时候最好依附rest协议去确定功能的进程。例如，一个get请求应该只返回数据，put或者post请求应该是修改数据，delete请求用于删除数据，文档会介绍在终端怎么使用http请求和这些请求能做什么。
However in some situations, verbs such as ``DELETE`` and ``PUT`` are blocked by firewalls, proxies or not implemented in clients. Furthermore, most developers are used to using ``GET`` and ``POST`` exclusively. Therefore, while the OpenTSDB API supports extended verbs, most requests can be performed with just ``GET`` by adding the query string parameter ``method_override``. This parameter allows clients to pass data for most API calls as query string values instead of body content. For example, you can delete an annotation by issuing a ``GET`` with a query string ``/api/annotation/start_time=1369141261&tsuid=010101&method_override=delete``. The following table describes verb behavior and overrides.
然而在一些情况下，delete和put请求可能会被防火墙拦截，代理服务器或者客户端不执行。此外，大部分开发者只习惯性的使用‘GET’和‘POST’。因此，当OpenTSDB支持一些请求扩展时，大部分请求都可以用“GET”请求加上“method_override”查询参数去执行。这个参数允许客户端传递数据作为大部分API请求查询字符串的之，而不是主题内容。例如，你可以 用‘GET’请求加上``/api/annotation/start_time=1369141261&tsuid=010101&method_override=delete``字符串去实现删除一个注释的功能。下表描述了请求动作的用法和覆盖。
.. csv-table::
   :header: "Verb", "Description", "Override"
   :widths: 10, 70, 20
   
   "GET", "Used to retrieve data from OpenTSDB. Overrides can be provided to modify content.用于从OpenTSDB检索数据。可以通过重写去修改内容。 **Note**: Requests via GET can only use query string parameters; see the note below.GET只能用查询字符串参数", "N/A"
   "POST", "Used to update or create an object in OpenTSDB using the content body from the request. Will use a formatter to parse the content body 用请求内容区修改或者创建OpenTSDB对象。它将用格式化程序去解析内容", "method_override=post"
   "PUT", "Replace an entire object in the system with the provided content 用提供的内容去替换系统的一整个对象", "method_override=put"
   "DELETE", "Used to delete data from the system 用于删除系统的数据", "method_override=delete"
   
If a method is not supported for a given API call, the TSD will return a 405 error.
如果不支持API调用的一种方法，TSD将返回一个405错误。

.. NOTE:: The HTTP specification states that there shouldn't be an association between data passed in a request body and the URI in a ``GET`` request. Thus OpenTSDB's API does not parse body content in ``GET`` requests. You can, however, provide a query string with data and an override for updating data in certain endpoints. But we recommend that you use ``POST`` for anything that writes data.
.. 注:: http规范规定，请求数据和‘GET’请求url之间不需要关联。因此OpenTSDB不需要再‘GET’请求中去解析主题内容。当然，你可以再指定终端通过带数据的查询字符串和方法重写去实现数据更改，但是我们建议用``POST``去写任何数据。
API版本
---------------

OpenTSDB 2.0's API call calls are versioned so that users can upgrade with gauranteed backwards compatability. To access a specific API version, you craft a URL such as ``/api/v<version>/<endpoint>`` such as ``/api/v2/suggest``. This will access version 2 of the ``suggest`` endpoint. Versioning starts at 1 for OpenTSDB 2.0.0. Requests for a version that does not exist will result in calls to the latest version. Also, if you do not supply an explicit version, such as ``/api/suggest``, the latest version will be used.
OpenTSDB 2.0的API调用方法被注释了，所以用户可以做兼容的升级。要访问指定的API版本，你可以添加一个url像 ``/api/v<version>/<endpoint>``像 ``/api/v2/suggest``，它就会访问``suggest``终端的版本2。OpenTSDB 2.0.0的版本是从1开始的，如果请求的版本不存在则会调用最接近的一个版本。如果你想``/api/suggest``这样不指定一个明确的版本，最近的版本将会被调用。
查询字符串 Vs. 内容体
-----------------------------

Most of the API endpoints support query string parameters, particularly those that fetch data from the system. However due to the complexities of encoding some characters, and particularly Unicode, all endpoints also support access via POST content using formatters. The default format is JSON so clients can use their favorite means of generating a JSON object and send it to the OpenTSDB API via a ``POST`` request. ``POST`` requests will generally provided greater flexibility in the fields offered and fully Unicode support than query strings. 
大多数API的端点支持查询字符串参数，特别是那些从系统数据的请求。然而，由于一些编码字符，特别是统一的复杂性，所有端 ​​点也可以通过使用格式化POST内容支持访问
CORS
----

OpenTSDB provides simple and preflight support for Cross-Origin Resource Sharing (CORS) requests. To enable CORS, you must supply either a wild card ``*`` or a comma separated list of specific domains in the ``tsd.http.request.cors_domains`` configuration setting and restart OpenTSDB. For example, you can supply a value of ``*`` or you could provide a list of domains such as ``beeblebrox.com,www.beeblebrox.com,aurtherdent.com``. The domain list is case insensitive but must fully match any value sent by clients.

When a ``GET``, ``POST``, ``PUT`` or ``DELETE`` request arrives with the ``Origin`` header set to a valid domain name, the server will compare the domain against the configured list. If the domain appears in the list or the wild card was set, the server will add the ``Access-Control-Allow-Origin`` and ``Access-Control-Allow-Methods`` headers to the response after processing is complete. The allowed methods will always be ``GET, POST, PUT, DELETE``. It does not change per end point. If the request is a CORS preflight, i.e. the ``OPTION`` method is used, the response will be the same but with an empty content body and a 200 status code.

If the ``Origin`` domain did not match a domain in the configured list, the response will be a 200 status code and an Error (see above) for the content body stating that access was denied, regardless of whether the request was a preflight or a regular request. The request will not be processed any further.

By default, the ``tsd.http.request.cors_domains`` list is empty and CORS is diabled. Requests are passed through without appending CORS specific headers. If an ``Options`` request arrives, it will receive a 405 error message.

.. NOTE:: Do not rely on CORS for security. It is exceedingly easy to spoof a domain in an HTTP request and OpenTSDB does not perform reverse lookups or domain validation. CORS is only implemented as a means to make it easier JavaScript developers to work with the API.

Documentation
-------------

The documentation for each endpoint listed below will contain details about how to use that endpoint. Eahc page will contain a description of the endpoint, what verbs are supported, the fields in a request, fields in a respone and examples. 

Request Parameters are a list of field names that you can pass in with your request. Each table has the following information:

* Name - The name of the field
* Data Type - The type of data you need to supply. E.g. ``String`` should be text, ``Integer`` must be a whole number (positive or negative), ``Float`` should be a decimal number. The data type may also be a complex object such as an array or map of values or objects. 
  If you see ``Present`` in this column then simply adding the parameter to the query string sets the value to ``true``, the actual value of the parameter is ignored. For example ``/api/put?summary`` will effectively set ``summary=true``. If you request ``/api/put?summary=false``, the API will still consider the request as ``summary=true``.
* Required - Whether or not the parameter is required for a successful query. If the parameter is required, you'll see ``Required`` otherwise it will be ``Optional``. 
* Description - A detailed description of the parameter including what values are allowed if applicable.
* Default - The default value of the ``Optional`` parameter. If the data is required, this field will be blank.
* QS - If the parameter can be supplied via query string, this field will have a ``Yes`` in it, otherwise it will have a ``No`` meaning the parameter can only be supplied as part of the request body content.
* RW - Describes whether or not this parameter can result in an update to data stored in OpenTSDB. Possible values in this column are:

  * *empty* - This means that the field is for queries only and does not, necessarily, represent a field in the response.
  * **RO** - A field that appears in the response but is read only. The value passed along with a request will not alter the output field. 
  * **RW** or **W** - A field that **will** result in an update to the data stored in the system
  
* Example - An example of the parameter value

Deprecated API
--------------

Read :doc:`deprecated`

API Endpoints
-------------

.. toctree::
   :maxdepth: 1
   
   s
   aggregators
   annotation/index
   config
   dropcaches
   put
   query/index
   search
   serializers
   stats
   suggest
   tree/index
   uid/index
   version
