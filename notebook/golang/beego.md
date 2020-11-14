## 介绍

- 特性
- 

## 环境安装及初始化配置

- beego
- bee 工具

## controller 设计

### 参数配置

- app.conf 文件
  - App
  - web
  - 监听
  - session
  - log



### 路由配置

- 基础路由：URI + 闭包函数
  - beego.Get(router, beego.FilterFunc)
  - beego.Post(router, beego.FilterFunc)
  - beego.Put(router, beego.FilterFunc)
  - beego.Patch(router, beego.FilterFunc)
  - beego.Head(router, beego.FilterFunc)
  - beego.Options(router, beego.FilterFunc)
  - beego.Delete(router, beego.FilterFunc)
  - beego.Any(router, beego.FilterFunc)
  - 支持自定义 Handler 实现
- RESTful 路由
  - 固定路由（全匹配）
  - 正则路由
    - 自定义方法
  - 自动匹配
  - 注解路由
  - NameSpace



### 控制器函数

- Init(ct *context.Context, childName string, app interface{})

  这个函数主要初始化了 Context、相应的 Controller 名称，模板名，初始化模板参数的容器 Data，app 即为当前执行的 Controller 的 reflecttype，这个 app 可以用来执行子类的方法。

- **Prepare()**

  这个函数主要是为了用户扩展用的，这个函数会在下面定义的这些 Method 方法之前执行，用户可以重写这个函数实现类似用户验证之类。

- Get()

  如果用户请求的 HTTP Method 是 GET，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Get 请求。

- Post()

  如果用户请求的 HTTP Method 是 POST，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Post 请求。

- Delete()

  如果用户请求的 HTTP Method 是 DELETE，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Delete 请求。

- Put()

  如果用户请求的 HTTP Method 是 PUT，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Put 请求.

- Head()

  如果用户请求的 HTTP Method 是 HEAD，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Head 请求。

- Patch()

  如果用户请求的 HTTP Method 是 PATCH，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Patch 请求.

- Options()

  如果用户请求的HTTP Method是OPTIONS，那么就执行该函数，默认是 405，用户继承的子 struct 中可以实现了该方法以处理 Options 请求。

- Finish()

  这个函数是在执行完相应的 HTTP Method 方法之后执行的，默认是空，用户可以在子 struct 中重写这个函数，执行例如数据库关闭，清理数据之类的工作。

- Render() error

  这个函数主要用来实现渲染模板，如果 beego.AutoRender 为 true 的情况下才会执行。

#### 提前终止运行

- this.stopRun()
  - 调用 StopRun 之后，如果你还定义了 Finish 函数就不会再执行，如果需要释放资源，那么请自己在调用 StopRun 之前手工调用 Finish 函数。
- 

### XSRF 过滤（跨站请求伪造）

### 请求数据处理

- 获取参数
  - GetString(key string) string
  - GetStrings(key string) []string
  - GetInt(key string) (int64, error)
  - GetBool(key string) (bool, error)
  - GetFloat(key string) (float64, error)

- 表单内容直接解析到 struct

- 获取 Request Body 里的内容

  - 配置文件里设置 `copyrequestbody = true`
  - 在 controller 获取

- 文件上传

  -  form 表单中增加这个属性 `enctype="multipart/form-data"`

  - 处理文件上传的两个方法

    - **GetFile**(key string) (multipart.File, *multipart.FileHeader, error)

      该方法主要用于用户读取表单中的文件名 `the_file`，然后返回相应的信息，用户根据这些变量来**处理文件上传：过滤、保存文件等**。

    - **SaveToFile**(fromfile, tofile string) error

      该方法是在 GetFile 的基础上实现了快速保存的功能
      fromfile 是提交时候的 html 表单中的 name

- 请求绑定

### Json、XML、Jsonp 输出

- JSON 数据直接输出：

```go
func (this *AddController) Get() {
    mystruct := { ... }
    this.Data["json"] = &mystruct
    this.ServeJSON()
}
```

​	调用 ServeJSON 之后，会设置 `content-type` 为 `application/json`，然后同时把数据进行 JSON 序列化输出。



### session 控制

- 使用 session

  - 在 main 入口函数中设置

    ```go
    beego.BConfig.WebConfig.Session.SessionOn = true
    ```

  - 通过配置文件设置

    ```properties
    sessionon = true
    ```

- session 方法

  - **SetSession**(name string, value interface{})
  - **GetSession**(name string) interface{}
  - **DelSession**(name string)
  - SessionRegenerateID()
  - DestroySession()

- session 模块使用的参数设置（配置文件）



### 过滤器——InsertFilter

- beego 支持自定义过滤中间件，例如安全验证，强制跳转等。

- 过滤器函数如下所示：

  ```go
  beego.InsertFilter(pattern string, position int, filter FilterFunc, params ...bool)
  ```

  - InsertFilter 函数的三个必填参数，一个可选参数
    - **pattern 路由规则**，可以根据一定的规则进行路由，全匹配可以用 `*`
    - **position 执行 Filter 的地方**，五个固定参数如下，分别表示不同的执行过程
      - BeforeStatic 静态地址之前

      - BeforeRouter 寻找路由之前

      - BeforeExec 找到路由之后，开始执行相应的 Controller 之前

      - AfterExec 执行完 Controller 逻辑之后执行的过滤器

        ```go
        // 例如：
        beego.InsertFilter("/v1/*", beego.AfterExec, filterAction, false, false)
        ```

      - FinishRouter 执行完逻辑之后执行的过滤器
    - **filter** filter 函数 type FilterFunc func(*context.Context)
    - params
      1. 设置 returnOnOutput 的值(默认 true), 如果在进行到此过滤之前已经有输出，是否不再继续执行此过滤器,默认设置为如果前面已有输出(参数为true)，则不再执行此过滤器
      2. 是否重置 filters 的参数，默认是 false，因为在 filters 的 pattern 和本身的路由的 pattern 冲突的时候，可以把 filters 的参数重置，这样可以保证在后续的逻辑中获取到正确的参数，例如设置了 `/api/*` 的 filter，同时又设置了 `/api/docs/*` 的 router，那么在访问 `/api/docs/swagger/abc.js` 的时候，在执行 filters 的时候设置 `:splat` 参数为 `docs/swagger/abc.js`，但是如果不清楚 filter 的这个路由参数，就会在执行路由逻辑的时候保持 `docs/swagger/abc.js`，如果设置了 true，就会重置 `:splat` 参数.

- **然后在 main 函数中进行初始化。**



 ### flash 函数

- 它主要用于在两个逻辑间传递临时数据，flash 中存放的所有数据会在紧接着的下一个逻辑中调用后清除。一般用于传递提示和错误消息。它适合 [Post/Redirect/Get](http://en.wikipedia.org/wiki/Post/Redirect/Get) 模式。
- flash 对象有三个级别的设置：
  - Notice 提示信息
  - Warning 警告信息
  - Error 错误信息



### URL 构建

- UrlFor() 函数就是用于构建指定函数的 URL 的。它把对应控制器和函数名结合的字符串作为第一个参数，其余参数对应 URL 中的变量。未知变量将添加到 URL 中作为查询参数。



### 表单数据验证

- 表单验证是用于数据验证和错误收集的模块。





## model 设计





## beego 模块设计

### session 模块

[session 模块](https://beego.me/docs/module/session.md)



### grace  模块——热升级



### cache 模块



### logs 模块

[logs 模块](https://beego.me/docs/module/logs.md) 

- 支持的引擎：console、file、conn、SMTP、ElasticSearch、multifile
- 可输出文件名和行号
- 可异步输出日志
- 

### httplib 模块——模拟客户端发送 http 请求

[httplib 模块](https://beego.me/docs/module/httplib.md)

-  支持的方法：
  - Get(url)
  - Post
  - Put
  - Delete
  - Dead
- 支持 Debug 输出
- 支持 HTTPS 请求
- 支持超时设置
- 设置
  - 请求参数
  - 使用 Body 发送大片的数据
  - 设置 header 信息
  - 设置 transport
- 支持文件直接上传接口
- 获取返回结果：
  - req.Response()
  - req.Bytes()
  - req.String()
  - req.ToFile(filename)
  - req.ToJson(&result)
  - req.ToXml(&result)



### context 模块——上下文模块

[context 模块](https://beego.me/docs/module/context.md)

- 作用：主要针对 HTTP 请求中，request 和 response 的进一步封装，包括用户的输入（request）和输出（response），context 分别提供了 Input 对象进行解析和 Output 对象进行输出。

- **Context 对象**：对 Input 和 Output 对象的封装，封装了以下几个方法：

  - Redirect：
  - Abort
  - WriteString
  - GetCookie
  - SetCookie
  - Context 对象是 Filter 函数的参数对象，可以通过 filter 来修改相应的数据，或者提前结束整个的执行过程。

- **Input 对象**：针对 request 的封装

  - 主要参数

    ```go
    pnames        []string
    pvalues       []string
    data          map[interface{}]interface{} // store some values in this context when calling context in filter or controller.
    RequestBody   []byte
    ```

  - 主要方法

    ```go
    Protocol()	// 获取用户请求的协议，例如 HTTP/1.0
    URI()		// 用户请求的 RequestURI，例如 /hi?id=1001
    URL()		// 请求的 URL 地址，例如 /hi
    Site()		// 请求的站点地址,scheme+doamin 的组合，例如 http://beego.me
    Scheme()	// 请求的 scheme，例如 “http” 或者 “https”
    Domain()	// 请求的域名，例如 beego.me
    Host()		// 请求的域名，和 domain 一样，没有的话返回 localhost
    Method()	// 请求的方法，标准的 HTTP 请求方法，例如 GET、POST 等
    Is(method string)	// 判断是否是某一个方法，例如 Is("GET") 返回 true
    AcceptsJSON()		// 判断 request 是否接收 json response
    IP()				// 返回请求用户的 IP，如果用户通过代理，一层一层剥离获取真实的 IP
    Referer()			// 返回请求的 refer 信息
    Proxy()				// 返回用户代理请求的所有 IP
    Refer()				// 返回请求的 refer 信息
    SubDomains()		// 返回请求域名的根域名，例如请求是 blog.beego.me，那么调用该函数返回 beego.me
    Port() 				// 返回请求的端口，例如返回 8080
    UserAgent()			// 返回请求的 UserAgent，例如 Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.57 Safari/537.36
    ParamsLen()			// 
    Param(key string)	// 在路由设置的时候可以设置参数，这个是用来获取那些参数的，例如 Param(":id"),返回12
    Params()			
    SetParam(key, val string)
    ResetParams()
    Query(key string)	// 该函数返回 Get 请求和 Post 请求中的所有数据，和 PHP 中 $_REQUEST 类似
    Header(key string) 	// 返回相应的 header 信息，例如 Header("Accept-Language")，就返回请求头中对应的信息 zh-CN,zh;q=0.8,en;q=0.6
    Cookie(key string)	// 返回请求中的 cookie 数据，例如 Cookie("username")，就可以获取请求头中携带的 cookie 信息中 username 对应的值
    Session(key interface{})	// session 是用户可以初始化的信息，默认采用了 beego 的 session 模块中的 Session 对象，用来获取存储在服务器端中的数据。
    CopyBody(MaxMemory int64)	// 
    Data()
    GetData(key interface{})	// 用来获取 Input 中 Data 中的数据
    ParseFormOrMulitForm(maxMemory int64)
    Bind(dest interface{}, key string)
    ```

- **Output 对象**

  - 主要参数：

    ```go
    Context    *Context
    Status     int
    EnableGzip bool
    ```

  - 主要方法：

    - 设置输出信息：
      - Header
      - Body
      - Cookie
      - ContentType
      - SetStatus
      - Session
      - 
    - 格式转换：
      - Json：把 Data 格式化为 Json，然后调用 Body 输出数据
      - Jsonp
      - Xml
      - Download：把 file 路径传递进来，然后输出文件给用户
    - 根据 status 判断：
      - IsCachable：是否为缓存类的状态
      - IsEmpty：是否为输出内容为空的状态
      - IsOK：是否为 200 的状态、
      - IsSuccessful：是否为正常的状态
      - IsRedirect：是否为跳转类的状态
      - IsForbidden：是否为禁用类的状态
      - IsNotFound：是否为找不到资源类的状态
      - IsClientError：是否为请求客户端错误的状态
      - IsServerError：是否为服务器端错误的状态

### toolbox 模块

[toolbox 模块](https://beego.me/docs/module/toolbox.md)

- **主要功能：健康检查、性能调优、访问统计、计划任务。**
- **HealthCheck**（健康检查）：
  - 用于当你应用于产品环境中进程，检查当前的状态是否正常
- **profile**（性能调优）：profile 提供了方便的入口方便用户来调试程序，他主要是通过入口函数 `ProcessInput` 来进行处理各类请求，主要包括以下几种调试：
  - **lookup goroutine**：打印出来当前全部的 goroutine 执行的情况
  - **lookup heap**：用来打印当前 heap 的信息
  - **lookup threadcreate**：查看创建线程的信息
  - **lookup block**：查看 block 的信息
  - **start cpuprof**：开始记录 cpuprof 信息，生产一个文件 cpu-pid.pprof，开始记录当前进程的 CPU 处理信息
  - **stop cpuprof**：关闭 CPU 记录信息
  - **get memprof**：开启记录 memprof，生产一个文件 mem-pid.memprof
  - **gc summary**：查看 GC 信息
- **statistics**：api 性能统计信息。
- **task**：定时任务
  - **Spec**：spec 格式是参照 crontab 做的

### config 模块——配置文件解析

[config 模块](https://beego.me/docs/module/config.md)

- 支持解析的文件格式有 ini、json、xml、yam
- 获取环境变量



### i18n 模块——国际化

[i18n 模块](https://beego.me/docs/module/i18n.md)

## API 自动化文档

[API 自动化文档](https://beego.me/docs/advantage/docs.md)





## 使用技巧：

- 使用 queryRow 和 queryRows 的时候，**必须创建对应的 struct**；使用 values 查询单条或者列表数据，都可以直接映射为 map key 为数据库字段名称小写。