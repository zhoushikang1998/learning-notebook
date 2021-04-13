## 一、查看网页源码

```bash
curl www.sina.com

# 保存网页，相当于 wget
curl -o [文件名] www.sina.com
```



## 二、自动跳转

```bash
# 自动跳转到 www.sina.com.cn
crul -L www.sina.com
```



## 三、显示头信息

```bash
# `-i` 参数可以显示 http response 的头信息，连同网页代码一起。
curl -i www.sina.com
# `-I` 参数则是只显示 http response 的头信息。
```



## 四、显示通信过程

```bash
# `-v`参数可以显示一次http通信的整个过程，包括端口连接和http request头信息。
curl -v www.sina.com
# 更详细的通信过程
curl --trace output.txt www.sina.com
curl --trace-ascii output.txt www.sina.com
```



## 五、发送表单信息

```bash
# 发送表单信息有GET和POST两种方法。GET方法相对简单，只要把数据附在网址后面就行。
curl example.com/form?data=xxx

# POST方法必须把数据和网址分开，curl就要用到--data参数。
curl -X POST --data "data=xxx" example.com/form

# 如果你的数据没有经过表单编码，还可以让curl为你编码，参数是`--data-urlencode`。
curl -X POST --data-urlencode "data=April 1" example.com/form
```



## 六、HTTP 动词

- curl默认的HTTP动词是GET，使用`-X`参数可以支持其他动词。

```bash
curl -X POST www.example.com

curl -X DELETE www.example.com
```



## 七、文件上传



## 八、Referer 字段

- 有时你需要在http request头信息中，提供一个referer字段，表示你是从哪里跳转过来的。

```bash
curl --referer http://www.example.com www.baidu.com
```



## 九、User Agent 字段

- 表示客户端的设备信息。服务器有时会根据这个字段，针对不同设备，返回不同格式的网页，比如手机版和桌面版。

```bash
curl --user-agent "[User Agent]" [URL]
```



## 十、cookie

- 使用 `--cookie` 参数，可以让curl发送cookie。

```bash
curl --cookie "name=xxx" www.example.com
```

- 具体的cookie的值，可以从http response头信息的`Set-Cookie`字段中得到。



- `-c cookie-file`可以保存服务器返回的cookie到文件，`-b cookie-file`可以使用这个文件作为cookie信息，进行后续的请求。

```bash
curl -c cookies http://www.example.com

curl -b cookies http://www.example.com
```



## 十一、增加头信息

- 在http request之中，自行增加一个头信息。`--header`参数就可以起到这个作用。

```bash
curl --header "Content-Type:application/json" http://example.com
```



## 十二、HTTP 认证

- 有些网域需要HTTP认证，这时curl需要用到`--user`参数。

```bash
curl --user name:password example.com
```



## 十三、参数汇总

### -A

- `-A`参数指定客户端的用户代理标头，即`User-Agent`。curl 的默认用户代理字符串是`curl/[version]`。

```bash
# 将User-Agent改成 Chrome 浏览器。
 curl -A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Safari/537.36' https://google.com
 
# 移除User-Agent标头
curl -A '' https://google.com

# 通过-H 参数直接指定标头，更改 User-Agent
curl -H 'User-Agent:php/1.0' https://google.com
```



### -b

- `-b`参数用来向服务器发送 Cookie。

```bash
# 生成一个标头 Cookie: foo=bar，向服务器发送一个名为foo、值为bar的 Cookie。
crul -b 'foo=bar' https://google.com

# 发送两个 Cookie
curl -b 'foo1=bar1;foo2=bar2' https://google.com

# 读取本地文件cookies.txt，里面是服务器设置的 Cookie（参见-c参数），将其发送到服务器。
curl -b cookies.txt https://google.com
```



### -c

- `-c`参数将服务器设置的 Cookie 写入一个文件。

```bash
# 将服务器的 HTTP 回应所设置 Cookie 写入文本文件cookies.txt。
curl -c cookies.txt https://www.google.com
```



### -d

- `-d`参数用于发送 POST 请求的数据体。

```bash
curl -d 'login=emma&password=123456' -X POST https://google.com/login
# 或者
curl -d 'login=emma' -d 'password=123456' -X POST https://google.com/login
```

- 使用`-d`参数以后，HTTP 请求会自动加上标头`Content-Type : application/x-www-form-urlencoded`。并且会自动将请求转为 POST 方法，因此**可以省略`-X POST`。**
- `-d`参数可以读取本地文本文件的数据，向服务器发送。

```bash
# 读取data.txt文件的内容，作为数据体向服务器发送。
curl -d '@data.txt' https://google.com/login
```



### --data-urlencode

- `--data-urlencode`参数等同于`-d`，发送 POST 请求的数据体，**区别在于会自动将发送的数据进行 URL 编码。**

```bash
# 发送的数据hello world之间有一个空格，需要进行 URL 编码。
curl --data-urlencode 'comment=hello world' https://google.com/login
```



### -e

- `-e`参数用来设置 HTTP 的标头`Referer`，表示请求的来源。

```bash
# Referer 标头设为 https://google.com?q=example。
curl -e 'https://google.com?q=example' https://www.example.com
```

- `-H`参数可以通过直接添加标头`Referer`，达到同样效果。

```bash
curl -H 'Referer: https://google.com?q=example' https://www.example.com
```



### -F

- `-F`参数用来向服务器上传二进制文件。

```bash
# 给 HTTP 请求加上标头 Content-Type: multipart/form-data，然后将文件 photo.png 作为 file 字段上传。
curl -F 'file=@photo.png' https://google.com/profile
```



- `-F`参数可以指定 MIME 类型。

```bash
# 指定 MIME 类型为image/png，否则 curl 会把 MIME 类型设为application/octet-stream。
curl -F 'file=@photo.png;type=image/png' https://google.com/profile
```



- `-F`参数也可以指定文件名。

```bash
# 原始文件名为photo.png，但是服务器接收到的文件名为me.png。
curl -F 'file=@photo.png;filename=me.png' https://google.com/profile
```



### -G

- `-G`参数用来构造 URL 的查询字符串。

```bash
# 发出一个 GET 请求，实际请求的 URL 为https://google.com/search?q=kitties&count=20。如果省略--G，会发出一个 POST 请求。
curl -G -d 'q=kitties' -d 'count=20' https://google.com/search

# 数据需要 URL 编码，可以结合--data--urlencode参数。
curl -G --data-urlencode 'comment=hello world' https://www.example.com
```



### -H

- `-H`参数添加 HTTP 请求的标头。

```bash
# 添加 HTTP 标头Accept-Language: en-US。
curl -H 'Accept-Language: en-US' https://google.com

# 添加两个 HTTP 标头。
curl -H 'Accept-Language: en-US' -H 'Secret-Message: xyzzy' https://google.com

# 添加 HTTP 请求的标头是Content-Type: application/json，然后用-d参数发送 JSON 数据。
curl -d '{"login": "emma", "pass": "123"}' -H 'Content-Type: application/json' https://google.com/login
```



### -i/-I

- `-i `参数打印出服务器回应的 HTTP 标头。

```bash
# 收到服务器回应后，先输出服务器回应的标头，然后空一行，再输出网页的源码。
curl -i www.baidu.com

# 
```

- `-I` 参数则是只显示 http response 的头信息。

```bash
# 输出服务器对 HEAD 请求的回应。
curl -I https://www.baidu.com

# --head参数等同于-I。
curl --head https://www.baidu.com
```



### -k

- `-k`参数指定跳过 SSL 检测。

```bash
# 不会检查服务器的 SSL 证书是否正确
curl -k https://www.example.com
```



### -L

- `-L`参数会让 HTTP 请求跟随服务器的重定向。curl 默认不跟随重定向。

```bash
curl -L -d 'tweet=hi' https://api.twitter.com/tweet
```



### --limit-rate

- `--limit-rate`用来限制 HTTP 请求和回应的带宽，模拟慢网速的环境。

```bash
# 将带宽限制在每秒 200K 字节。
curl --limit-rate 200k https://google.com
```



### -o/-O

- `-o` 参数将服务器的回应保存成文件，等同于 `wget` 命令。
- `-O` 参数将服务器回应保存成文件，并将 URL 的最后部分当作文件名。

```bash
# 将www.example.com保存成example.html
curl -o example.html https://www.example.com

# 将服务器回应保存成文件，文件名为 bar.html
curl -O https://www.example.com/foo/bar.html
```



### -s/-S

- `-s` 参数将不输出错误和进度信息。
- `-S` 参数指定只输出错误信息，通常与`-o`一起使用。

```bash
curl -s -o /dev/null https://google.com
curl -S -o /dev/null https://google.com
```



### -u

- `-u `参数用来设置服务器认证的用户名和密码。

```bash
# 设置用户名为bob，密码为12345，然后将其转为 HTTP 标头Authorization: Basic Ym9iOjEyMzQ1
curl -u 'bob:123456' https://google.com/login

# 只设置了用户名，执行后，curl 会提示用户输入密码。
curl -u 'bob' https://google.com/login
```

- curl 能够识别 URL 里面的用户名和密码。

```bash
# 同上
curl https://bob:12345@google.com/login
```



### -v

- `-v`参数输出通信的整个过程，用于调试。
- `--trace`参数也可以用于调试，还会输出原始的二进制数据。
- 也可用 `--trace-ascii`

```bash
curl -v https://www.example.com
curl --trace output.txt https://www.example.com
```



### -x

- `-x`参数指定 HTTP 请求的代理。

```bash
# 指定 HTTP 请求通过 myproxy.com:8080的 socks5 代理发出。
curl -x socks5://james:cats@myproxy.com:8080 https://www.example.com

# 没有指定代理协议，默认为 HTTP。
curl -x james:cats@myproxy.com:8080 https://www.example.com
```



### -X 

- `-X`参数指定 HTTP 请求的方法。

```bash
# 对 https://www.example.com 发出 POST 请求。
curl -X POST https://www.example.com
```

