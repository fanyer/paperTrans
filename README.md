#来自HeroKu的HTTP API 设计指南(fanyer译)


原文来自  [Heroku Dev Center](https://devcenter.heroku.com/articles/platform-api-reference)

由*fanyer*整理翻译，**非商业目的**自由转载

----

> ###目录
> 
> **基础**
> 
> - 总是使用TLS
> - 在Accepts头中带上版本号
> - 通过Etags支持缓存
> - 用Request-Ids追踪请求
> - 用Ranges来分页
> 
> **请求**
> 
> - 返回适当的状态码
> - 总是返回完整的资源
> - 在请求body中接收JSON序列
> - 使用一致的路径格式
> - 小写所有路径和属性
> - 支持非ID的参数作为快捷方式
> - 少用路径嵌套
> 
> **响应**
> 
> - 总是提供资源(UU)ID
> - 提供标准的时间戳
> - 使用ISO8601格式的UTC时间
> - 嵌入外键数据
> - 总是生成结构化的错误信息
> - 显示频率限制的状态
> - 在所有的响应中压缩JSON数据
> 
> **文档及其他**
> 
> - 提供机器可读的JSON格式
> - 提供人类可读的文档
> - 提供可执行的示例
> - 描述稳定性
> 
> 


---
##一、基础
**1.总是使用TLS**

总是使用TLS（就是https）来访问API，没有必要指出什么时候需要用，什么时候不需要用，只管任何时候都用它就好。

对所有非TLS的请求返回403 Forbidden，不要用重定向，这会允许一些不良的客户端行为，而又没有任何好处。依赖重定向的客户端会使流量翻倍，而让TLS毫无意义 —— 敏感数据已经在第一次请求时发送出来了。

**2.在Accepts头中带上版本号**

从一开始就为API分配版本。使用Accepts头来发送版本信息，可以使用自定义的内容类型，如：

    Accept: application/vnd.heroku+json; version=3

不要提供默认版本，而由客户端显式指定它使用哪一个特定的版本。

**3.通过Etags支持缓存**

在所有的请求中带上 ETag 头 ， 用于识别特定版本的返回资源。用户可以在随后的请求中通过提供If-None-Match头的值来检查内容是否过期。

**4.用Request-Ids追踪请求**

在每个API相应中提供Request-Id头，带上一个唯一的UUID值。如果服务器和客户端都记录了这些值，在跟踪和调试请求时会派上大用场。

**5.用Ranges来分页**

对所有可能产生大量数据的响应进行分页。使用Content-Range 头来标记分页请求。可以参考这个例子，来了解请求和响应头、状态码、Limit、排序和翻页：Heroku Platform API on Ranges



---
##二、请求
**返回适当的状态码**

为每个请求返回适当的状态码，成功的请求应该遵守如下规则：

    200: 当GET请求成功完成，DELETE或者PATCH请求同步完成。

    201: 同步方式成功完成POST请求。

    202: POST，DELETE或者PATCH请求提交成功，稍后将异步的进行处理。

    206: GET请求成功完成，但只返回了部分数据。参见用ranges分页

注意认证和认证错误的使用：

    401 Unauthorized: 请求失败，因为用户没有进行认证。
    403 Forbidden: 请求失败，因为用户被认定没有访问特定资源的权限。

返回合适的状态码可以为错误提供更多的信息：

    422 Unprocessable Entity: 你的请求服务器可以理解，但是其中包含了不合法的参数。
    429 Too Many Requests: 请求频率超配，稍后再试。
    500 Internal Server Error: 服务器出错了，检查网站的状态，或者报告问题。

根据HTTP response code 规范的指导来设计用户错误和服务器错误情况下的状态码。



**总是返回完整的资源**

对于200和201的响应，总是尽可能在响应中返回完整的资源（比如一个对象的所有属性），包括PUT，PATCH和DELETE请求，如：

    $ curl -X DELETE \  
      https://service.com/apps/1f9b/domains/0fd4
    
    HTTP/1.1 200 OK
    Content-Type: application/json;charset=utf-8
    ...
    {
      "created_at": "2012-01-01T12:00:00Z",
      "hostname": "subdomain.example.com",
      "id": "01234567-89ab-cdef-0123-456789abcdef",
      "updated_at": "2012-01-01T12:00:00Z"
    }

202响应则不用包含完整的资源，如:

    $ curl -X DELETE \  
      https://service.com/apps/1f9b/dynos/05bd
    
    HTTP/1.1 202 Accepted
    Content-Type: application/json;charset=utf-8
    ...
    {}


**在请求body中接收JSON序列**

不要将额外信息放到form-encoded里边，而是将其JSON序列放到PUT，PATCH或POST请求的Body中。这样才能和同为JSON序列的响应Body对称（作者你是处女座么），如：

    $ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'
    
    {
      "id": "01234567-89ab-cdef-0123-456789abcdef",
      "name": "demoapp",
      "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
      },
      ...
    }

**使用一致的路径格式**

- 资源名称——使用复数来命名资源，除非该资源在系统中是单件（比如，在绝大多数系统中，一个用户只能拥有一个账户）。这样在你引用特定资源时可以保持一致性。


- 动作——对独有的资源使用不需要特定动作的endpoint格式。这样当需要特定的动作，只需要把它们放到标准的actions前缀后边，就可以清晰的描述它们：

    /resources/:resource/actions/:action

如：

	/runs/{run_id}/actions/stop



**小写所有路径和属性**

使用小写字母和减号命名路径，这样Hostname可以对齐（作者你真的是处女座）：
	
	service-api.com/users
	service-api.com/app-setups

同样小写属性，但使用下划线来分割，这样属性名在JavaScript中可以不用加引号：

	service_class: "first"


**支持非ID的参数作为快捷方式**

有时候要求最终用户提供ID来表示资源会比较麻烦，比如，用户可能只想得起Heroku的Appname，而应用本身却是由UUID来区分的。在这种情况下，我们可以同时接收ID和Name：

	$ curl https://service.com/apps/{app_id_or_name}
	$ curl https://service.com/apps/97addcf0-c182
	$ curl https://service.com/apps/www-prod

绝不要只接收名称来排除某些ID。


**少用路径嵌套**

在嵌套了父子资源的数据模型中，路径可能深度嵌套：

	/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}

可以通过从根路径定位来限制嵌套层数。使用嵌套来标识作用域内部的数据集。比如，上边那个dyno属于一个app，而app又属于一个org的例子：

	/orgs/{org_id}
	/orgs/{org_id}/apps
	/apps/{app_id}
	/apps/{app_id}/dynos
	/dynos/{dyno_id}

---
##响应
**总是提供资源(UU)ID**

为每个资源提供默认的ID属性。除非有特殊理由，总是使用UUID。不要用那些在服务的实例间或资源间不全局唯一的ID，特别是自增ID。

以8-4-4-4-12的格式小写UUID：

	"id": "01234567-89ab-cdef-0123-456789abcdef"

**提供标准的时间戳**

为资源提供默认的 created_at 和 updated_at 时间戳：

	{
	  ...
	  "created_at": "2012-01-01T12:00:00Z",
	  "updated_at": "2012-01-01T13:00:00Z",
	  ...
	}

如果这些时间戳对某些资源真的没有意义，那么你也可以去掉它。

**使用ISO8601格式的UTC时间**

只接受和返回UTC时间，以ISO8601格式显示：

	"finished_at": "2012-01-01T12:00:00Z"

**嵌入外键数据**

将外键引用通过序列化的嵌入对象显示：

	{
	  "name": "service-production",
	  "owner": {
	    "id": "5d8201b0..."
	  },
	  ...
	}

而不是这样：

	{
	  "name": "service-production",
	  "owner_id": "5d8201b0...",
	  ...
	}

这使得我们可以在inline使用相关的数据，而不需要改变响应的格式，或者引入更多高层的响应字段：

	{
	  "name": "service-production",
	  "owner": {
	    "id": "5d8201b0...",
	    "name": "Alice",
	    "email": "alice@heroku.com"
	  },
	  ...
	}

**总是生成结构化的错误信息**

为错误生成一致的，结构化的响应Body。包含机器可读的id，人类可读的message，以及可选的url指向关于错误的更多信息，还有如何解决它：

    //HTTP/1.1 429 Too Many Requests

    {
      "id":  "rate_limit",
      "message": "Account reached its API rate limit.",
      "url": "https://docs.service.com/rate-limits"
    }

为客户端常见的错误的格式和id撰写文档。

**显示频率限制的状态**

对客户端的频率限制可以保护服务的健康，并对其他的客户端提供高质量的服务。你可以使用token bucket 算法 来量化请求限制。

在每次请求的响应头中，通过RateLimit-Remaining 返回剩余的请求次数。
在所有的响应中压缩JSON数据

额外的空格增大了响应的大小，而很多人性化的客户端可以自动美化JSON输出。所以最好将JSON响应进行压缩：

	{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}

不要这样：

	{
	  "beta": false,
	  "email": "alice@heroku.com",
	  "id": "01234567-89ab-cdef-0123-456789abcdef",
	  "last_login": "2012-01-01T12:00:00Z",
	  "created_at": "2012-01-01T12:00:00Z",
	  "updated_at": "2012-01-01T12:00:00Z"
	}

你可以考虑提供一个可选的方式来为客户端输出更长的响应，比如通过请求参数（如?pretty=true）或者通过 Accept头（如Accept: application/vnd.heroku+json; version=3; indent=4;）。

##五、文档及其他
**提供机器可读的JSON格式**

提供机器可读的schema来描述你的API，可以用prmd来管理你的schema，用过prmd verify来确保它通过验证。

**提供人类可读的文档**

提供人类可读的文档帮助客户端开发者们理解你的API。

如果你使用了prmd来创建schema，那么你可以简单的通过prmd doc命令来生成Markdown的endpoint级别的文档。

除了endpoint级别的描述，还要提供概要级别的信息，比如：

- 授权，包括获得和使用授权Token。
- API的稳定性和版本，包括如何选择现有的API版本。
- 通用请求和响应头。
- 错误的序列化格式。
- 各种语言的客户端如何使用API的例子。

**提供可执行的示例**

提供可执行的例子，这样用户可以直接在终端输入并看到可以用的API请求。最好的情况是，这些例子可以直接复制粘贴，以最小化用户试用API的成本，如：

	$ export TOKEN=... # acquire from dashboard
	$ curl -is https://$TOKEN@service.com/users

如果你使用prmd来生成Markdown文档，你就免费获得了可执行的示例。

**描述稳定性**

描述你API的稳定性，以及哪些endpoint依赖于其成熟度，比如使用prototype，development或者production的标识。

可参考 Heroku API compatibility policy 了解哪些接口是稳定的，哪些可能有变动。

一旦你的API宣布为 production-ready 和 稳定版，不要在该API版本上做任何不向前兼容的修改。如果你需要做不向前兼容的修改，创建一个新的版本号。
