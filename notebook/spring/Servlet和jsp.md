## 1 Servlet接口有哪些方法及Servlet生命周期

Servlet接口定义了5个方法，前三个方法与Servlet生命周期有关：

- void init()
- void service()
- void destory()
- java.lang.String getServletInfo()
- ServletConfig getServletConfig()

**生命周期**：web容器加载Servlet并将其实例化后，Servlet生命周期开始，容器运行其init() 方法进行Servlet初始化；请求到达时调用Servlet的service()方法，service()方法会根据需要调用doPost()或doGet()方法；当服务器关闭或项目被卸载时服务器会将Servlet实例销毁，此时会调用Servlet的destroy()方法。**init()方法和destroy()方法只会执行一次，service()方法在客户端每次请求Servlet都会执行**。可以把初始化资源的代码放到init方法中，销毁资源的代码放到destroy方法中，这样就不需要每次处理客户端的请求都要初始化与销毁资源。

## 2 转发（forward）和重定向（redirect）的区别

**转发是服务器行为，重定向是客户端行为**

**转发**通过RequestDispatcher对象的forward（HTTPServletRequest request, HttpServletResponse response）方法实现的。RequestDispatcher可以通过HttpServletRequest 的 getRequestDispatcher()方法获得。

```java
request.getRequestDispatcher("login_success.jsp").forward(request, response);
```

**重定向**是利用服务器返回的状态码来实现的。客户端浏览器请求服务器的时候，服务器会返回一个状态码。服务器通过 HttpServletResponse 的setStatus(int status) 方法来设置状态码。如果服务器返回301或者302，则浏览器回到新的网址重新请求该资源。

```properties
# 从地址栏显示来说
forward是服务器请求资源，服务器直接访问目标地址的URL，把哪个URL的响应内容读取过来，然后把这些内容再发给浏览器。浏览器根本不知道服务器发送的内容从哪里来的，所以它的地址栏还是原来的地址。
redirect是服务器根据逻辑，发送一个状态码，告诉浏览器重新去请求哪个地址，所以地址栏显示的是新的URL。

# 从数据共享来说
forward：转发页面和转发到的页面可以共享request中的数据。redirect：不能共享数据

# 从运用地方来说
forward：一般用于用户登录的时候，根据角色转发到相应的模块
redirect：一般用于用户注销登录时返回主页面和跳转到其他的网站等

# 从效率来说
forward：高
redirect：低
```



## 3 Servlet和JSP的关系

Servlet是一个特殊的Java程序，**它运行于服务器的 JVM中**，能够依靠服务器的支持向浏览器提供显示内容。JSP本质上是Servlet的一种简易形式，JSP会被服务器处理成一个类似Servlet的Java程序，可以简化页面内容的生成。

Servlet和JSP最主要的不同在于：Servlet的应用逻辑是在java文件中，并且完全从表现层中的HTML中分离出来。而JSP 的情况是Java和HTML可以组合成一个扩展名为.jsp的文件。JSP侧重于视图，Servlet更侧重于控制逻辑，在MVC架构模式中，JSP适合充当视图（view）而Servlet适合冲到控制器（controller）



## 4 JSP工作原理

JSP是先部署后编译。**JSP会在客户端第一次请求JSP文件时被编译为HttpJspPage类**（接口Servlet的一个子类），该类会被服务器临时存放在服务器工作目录里面。

由于JSP只会在客户端第一次请求的时候编译，因此第一次请求JSP时会感觉比较慢，之后就会感觉快很多。如果把服务器保存的class文件删除，服务器也会重新编译JSP。

开发Web程序时经常需要修改JSP。**Tomcat能够自动检测到JSP程序的改动**。如果检测到JSP源代码发生了改动，Tomcat会在下次客户端请求JSP时重新编译JSP，而不需要重启Tomcat。这种自动检测功能是默认开启的，检测改动会耗费少量的时间，在部署web应用的时候可以在web.xml中将它关掉。

 ## 5 JSP九大内置对象

### out输出流对象

隐藏对象out 是javax.servlet.jsp.JspWriter类的实例，服务器向客户端输出的字符内容可以通过out对象输出。

==out对象常用的方法==

```java
void clear();	// 清除缓冲区的内容
void clearBuffer(); 	// 清除缓冲区的当前内容
void flush();	// 将缓冲内容flush到客户端浏览器
int getBufferSize();	// 返回缓冲大小，单位KB
int getRemaining();		//返回缓冲剩余大小，单位KB
boolean isAutoFlush();		// 返回缓冲区满时，是自动清空还是跑出异常
void close();	// 关闭输出流
```



### request请求对象

隐藏对象request是javax.servlet.ServletRequest类的实例，代表客户端的请求。request包含客户端的信息以及请求的信息（来自GET或POST请求），如请求哪个文件，附带的地址参数等。每次客户端的请求都会产生一个request实例。

==request对象常用的方法==

```properties
setAttribute(String name,Object value)：设置名字为name的request 的参数值
getAttribute(String name)：返回由name指定的属性值
getAttributeNames()：返回request 对象所有属性的名字集合，结果是一个枚举的实例
removeAttribute(String name)：删除请求中的一个属性

getCookies()：返回客户端的所有 Cookie 对象，结果是一个Cookie 数组
getSession([Boolean create]) ：返回和请求相关 Session

getCharacterEncoding() ：返回请求中的字符编码方式

getContentLength() ：返回请求的 Body的长度

getHeader(String name) ：获得HTTP协议定义的文件头信息
getHeaders(String name) ：返回指定名字的request Header 的所有值，结果是一个枚举的实例
getHeaderNames() ：返回所有request Header 的名字，结果是一个枚举的实例

getInputStream() ：返回请求的输入流，用于获得请求中的数据

getMethod() ：获得客户端向服务器端传送数据的方法（get或post）

getParameter(String name) ：获得客户端传送给服务器端的有 name指定的参数值
getParameterNames() ：获得客户端传送给服务器端的所有参数的名字，结果是一个枚举的实例
getParameterValues(String name)：获得有name指定的参数的所有值

getProtocol()：获取客户端向服务器端传送数据所依据的协议名称

getQueryString() ：获得查询字符串

getRequestURI() ：获取发出请求字符串的客户端地址
getRemoteAddr()：获取客户端的 IP 地址
getRemoteHost() ：获取客户端的名字
getServletPath()：获取客户端所请求的脚本文件的路径

getServerName() ：获取服务器的名字
getServerPort()：获取服务器的端口号
```



### response响应对象

response是javax.servlet.ServletResponse类的实例，代表对客户端的响应。服务器端的任何输出都通过response对象发送到客户端浏览器中。每次服务器端都会响应一个response实例。

==response对象的常用方法：==

```properties
String getCharacterEncoding() 　　 返回响应用的是何种字符编码 
void setCharacterEncoding(“gb2312”) 　　设置响应头的字符集

ServletOutputStream getOutputStream() 　　返回响应的一个二进制输出流
PrintWriter getWriter() 　　返回可以向客户端输出字符的一个对象 

 void setContentLength(int len) 　　设置响应头长度 
 void setContentType(String type) 　　设置响应的MIME类型 
 
 sendRedirect(java.lang.String location) 　　重新定向客户端的请求 
```



### config配置对象

config对象是Javax.servlet.ServletConfig类的实例，ServletConfig封装了配置在web.xml中初始化JSP的参数。JSP通过config获取这些参数。每个JSP文件中共有一个config对象。

==config对象的常用方法==

```properties
String getInitParameter(String name)　　返回配置在web.xml中初始化参数 
Enumeration getInitParameterNames() 　　返回所有的初始化参数名称 

ServletContext getServletContext()　　返回ServletContext对象 

String getServletName　　返回Servlet对象
```

### session会话对象

session对象是javax.servlet.http.HttpSession类的实例。Servlet中通过request.getSession()来获取session对象，而JSP中可以直接使用。如果JSP配置了<%@page session="false" %>，则隐藏对象session不可用。每个用户对应一个session对象。

==session对象的常用方法==

```properties
long getCreationTime() 　　返回Session创建时间 
public String getId() 　　返回Session创建时JSP引擎为它设的唯一ID号 
long getLastAccessedTime() 　　返回此Session里客户端最近一次请求时间 
String[] getValueNames() 　　返回一个包含此Session中所有可用属性的数组 

int getMaxInactiveInterval()　　 返回两次请求间隔多长时间此Session被取消(ms) 

void invalidate() 　取消Session,使Session不可用 
 
boolean isNew() 　　返回服务器创建的一个Session,客户端是否已经加入

void removeValue(String name) 　　删除Session中指定的属性
void setAttribute(String key,Object obj) 　　设置Session的属性
Object getAttribute(String name)　　 返回session中属性名为name的对象
```

###  application应用程序对象

application对象是javax.servlet.ServletContext类的实例。application封装JSP所在Web应用程序的信息，例如web.xml中配置的全局的初始化信息。Servlet中application对象需要通过ServletConfig.getServletContext()来获取。整个web应用程序对应一个application对象。

==application对象常用方法==

```properties
Object getAttribute(String name)　　返回application中属性为name的对象
Enumeration getAttributeNames() 　　返回application中的所有属性名
void setAttribute(String name,Object value)　　设置application属性
void removeAttribute(String name) 　　移除application属性

String getInitParameter(String name)　　返回全局初始化函数
Enumeration getInitParameterNames(）　　返回所有的全局初始话参数

String getMimeType(String filename)　　返回文件的文档类型，例如getMimeType(“abc.html”)将返回“text.html”
String getRealPath(String relativePath）　　返回Web应用程序内相对网址对应的绝对路径
```



### page页面对象

page对象是javax.servlet.jsp.HttpJspPage类的实例。page对象代表当前JSP页面，是当前JSP编译后的Servlet类的对象。page相当于Java类中的关键字this



### pageContext 页面上下文对象

pageContext对象是javax.servlet.jsp.pageContext类的实例。pageContext对象代表当前JSP页面编译后的内容。**通过pageContext可以获取到JSP中的资源。**

==pageContext对象的常用方法==

```properties
JspWriter getOut() 　　返回out对象
HttpSession getSession() 　　 返回Session对象(session) 
Object getPage() 　　返回page对象 
ServletRequest getRequest() 　　 返回request对象
ServletResponse getResponse() 　　 返回response对象

void setAttribute(String name,Object attribute) 　　　设置属性及属性值 ，在page范围内有效
void setAttribute(String name,Object obj,int scope)　　 在指定范围内设置属性及属性值, 1=page,2=request,3=session,4=application
public Object getAttribute(String name) 　　取属性的值
Object getAttribute(String name,int scope) 　　在指定范围内取属性的值
public Object findAttribute(String name) 　　寻找一属性,返回起属性值或NULL
void removeAttribute(String name) 　　删除某属性
void removeAttribute(String name,int scope) 　　 在指定范围删除某属性
int getAttributeScope(String name)　　 返回某属性的作用范围
Enumeration getAttributeNamesInScope(int scope) 　　返回指定范围内可用的属性名枚举

void release() 　　释放pageContext所占用的资源

void forward(String relativeUrlPath) 　　 使当前页面重导到另一页面

void include(String relativeUrlPath) 　　 在当前位置包含另一文件
```

### exception异常对象

exception对象是java.lang.Exception类的对象。exception封装了JSP中抛出的异常信息。要使用exception隐藏对象，需要设置<%@page isErrorPage="true" %>。exception对象通常被用来处理错误页面。



## 6 JSP三大指令

### page指令

**实例**：<%@ page language=”java” import=”java.util.*” pageEncoding=”UTF-8”%>

**import**：等同与import语句 
<%@ page import=”java.util.*” %> 
<%@ page import=”java.util., java.net.” %> 
在一个JSP页面中可以给出多个page指令，而且import是可以重复出现的 
<%@ page import=”java.util.*” %> 

**pageEncoding**：指定当前页面的编码 
如果pageEncoding没有指定，那么默认为contentType的值； 
如果pageEncoding和contentType都没有指定，那么默认值为iso-8859-1

**contentType**：等同与调用response.setContentType(“text/html;charset=xxx”); 
如果没有指定contentType属性，那么默认为pageEncoding的值；                                                                                     如果pageEncoding和contentType都没有指定，那么默认值为iso-8859-1 

**errorPage**：如果当前页面出现异常，那么跳转到errorPage指定的jsp页面。 
例如：<%@ page errorPage=”b.jsp” %> 

**isErrorPage**：上面示例中指定b.jsp为错误页面，但在b.jsp中不能使用内置对象exception，只有有b.jsp中使用<%@page isErrorPage=”true”%>时，才能在b.jsp中使用错误页面。

**autoFlush**：当autoFlush为true时，表示out流缓冲区满时会自动刷新。默认为true 

**buffer**：指定out流的缓冲区大小，默认为8KB 

**isELIgnored**：当前JSP页面是否忽略EL表达式，默认为false，表示不忽略，即支持EL表达式



**page指令不常用的属性：**

**language**：当前JSP编译后的语言！默认为java，当前也只能选择java

**info**：当前JSP的说明信息 

**isThreadSafe**：当前JSP是否执行只能单线程访问，默认为false，表示支持并发访问 

**session**：当前页面是否可以使用session，默认为false，表示支持session的使用。

**extends**：指定JSP编译的servlet的父类！



### include指令

**JSP可以通过include指令来包含其他文件。**被包含的文件可以是JSP文件、HTML文件或文本文件。包含的文件就好像是该JSP文件的一部分，会被同时编译执行。

Include指令的语法格式如下： 
<%@ include file=”文件相对 url 地址” %> 

### taglib指令

**taglib指令是用来在当前jsp页面中导入第三方的标签库** 

<%@ taglib uri="http://java.sun.com/jsp/jstl/core" % prefix=”c” > 

prefix：指定标签前缀，这个东西可以随意起名 
uri：指定第三方标签库的uri（唯一标识） 
**需要先把第三方标签库所需jar包放到类路径中。**



## 7 JSP 七大动作

**jsp:include**：在页面被请求的时候引入一个文件。

**jsp:useBean**：寻找或者实例化一个 JavaBean。

**jsp:setProperty**：设置 JavaBean 的属性

**jsp:getProperty**：输出某个 JavaBean 的属性

**jsp:forward**：把请求转到一个新的页面。 

**jsp:plugin**：根据浏览器类型为 Java 插件生成 OBJECT 或 EMBED 标记 

**jsp:text**：允许在JSP页面和文档中使用写入文本的模板



## 8 JSP的四种作用域

JSP中的四种作用域包括：page、request、session、application：

- **page** ：代表与一个页面相关的对象和属性
- **request** ：代表与Web客户端发出的一个请求相关的对象和属性。**一个请求可能跨越多个页面，涉及多个Web组件；**需要在页面显示的临时数据可以置于此作用域。
- **session** ：代表与某个用户与服务器建立的一次会话相关的对象和属性。跟某个用户相关的数据应该放在用户自己的session中。
- **application** ：代表与整个Web应用程序相关的对象和属性。它实质上是跨越整个Web应用程序，包括多个页面、请求和会话的一个全局作用域。

## 9 request.getAttribute()和 request.getParameter()有何区别

**获取方向：**

`getParameter()`是获取 POST/GET 传递的参数值；

`getAttribute()`是获取对象容器中的数据值；

**用途**：

getParameter()用于客户端重定向时，即点击了链接或提交按钮时传值用，即用于在用表单或url重定向传值时接收数据用。

getAttribute() 用于服务器端重定向时，即在 sevlet 中使用了 forward 函数,或 struts 中使用了 mapping.findForward。 getAttribute 只能收到程序用 setAttribute 传过来的值。

可以用setAttribute()，getAttribute()**发送接收对象**，而getParameter()只能传递字符串。`setAttribute()` 是应用服务器把这个对象放在该页面所对应的一块内存中去，当页面服务器重定向到另一个页面时，应用服务器会把这块内存拷贝到另一个页面所对应的的内存中。这样getAttribute() 就能取得所设下的值，也可以取得对象。

session也一样，只是对象在内存中的生命周期不一样而已。

**总结**

`getParameter()`返回的是String,用于读取提交的表单中的值;（获取之后会根据实际需要转换为自己需要的相应类型，比如整型，日期类型等等）

`getAttribute()`返回的是Object，需进行转换,可用`setAttribute()`设置成任意对象，使用很灵活，可随时用



## 10 include指令和include动作的区别

**include指令：** JSP可以通过include指令来**包含其他文件**。被包含的文件可以是JSP文件、HTML文件或文本文件。包含的文件就好像是该JSP文件的一部分，会被**同时编译执行**。 语法格式如下： <%@ include file="文件相对 url 地址" %>

i**nclude动作：** `<jsp:include>`动作元素用来**包含静态和动态的文件**。该动作**把指定文件插入正在生成的页面**。语法格式如下： <jsp:include page="相对 URL 地址" flush="true" />

## 11 cookie 和 session 的区别

Cookie 和 session都是用来跟踪浏览器用户身份的会话方式。

**cookie**：一般用来保存用户信息。数据保存在客户端(浏览器端)。如果使用 Cookie 的一些敏感信息不要写入 Cookie 中，最好能将 Cookie 信息加密然后使用到的时候再去服务器端解密。cookie只能存储String类型的对象。

**Session：**主要作用就是通过服务端记录用户的状态。 数据保存在服务器端。相对来说 Session 安全性更高。session能够存储任意的Java对象。

