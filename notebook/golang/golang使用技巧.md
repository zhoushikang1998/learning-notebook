

## 使用技巧

- 会自动执行的函数
  - init 函数先于 main 函数自动执行，不能被其他函数调用；
  - init 函数没有输入参数、返回值；
  - 每个包可以有多个 init 函数；
  - **包的每个源文件也可以有多个 init 函数**，这点比较特殊；
  - 同一个包的 init 执行顺序，golang 没有明确定义，编程时要注意程序不要依赖这个执行顺序。
  - 不同包的 init 函数按照包导入的依赖关系决定执行顺序。

```go
fun init(){}  // 会自动执行
```

- map 复制前要先初始化：make
  
  - 删除 map 不存在的键值对时，不会报错，相当于没有任何作用；获取不存在的键值对时，返回值类型对应的零值。
- go mod 和 gopath 模式互不兼容（IDEA 设置中开启或关闭 go modules）
  
  - gopath：go env -w GO111MODULE=off
  
  - go  mod：go env -w GO111MODULE=on
  
    - ```properties
      replace data-operation => E:\zhoushikang\GoProjects\src\data-operation
      ```
- 未知错误：
  
  - 确保每个 err 都有处理。
- slice 截取操作符 [i:j:k]
  - [i:j] 形式，k 默认为数组长度
  - 截取获得的切片的长度和容量分别是：**j-i、k-i。**

- 错误处理
  
  - 



## api



### flag 包：处理命令行参数

- 方法

```go
/**
	flag.Xxx(nama, value, usage) p：定义 flag 参数
	flag.XxxVar(p, nama, value, usage)：将 flag 绑定到一个变量
	flag.Var()：绑定自定义类型(需要实现 Value 接口)

	flag.Parse()：解析命令行参数到定义的flag
		- 解析函数将会在碰到第一个非flag命令行参数时停止，非flag命令行参数是指不满足命令行语法的参数
	
	 flag.Args()、flag.NArg()、flag.Arg(i) 函数：获取未能解析(非 flag)的命令行参数。
*/
```

- 命令行传参的格式：

  ```shell
  -isbool    (一个 - 符号，布尔类型该写法等同于 -isbool=true)
  -age=x     (一个 - 符号，使用等号)
  -age x     (一个 - 符号，使用空格)
  --age=x    (两个 - 符号，使用等号)
  --age x    (两个 - 符号，使用空格)
  ```

  



### os 包：文件操作、进程管理、信号和用户账号

- 作用：提供了对操作系统功能的非平台相关访问接口。接口为 Unix 风格。提供的功能包括：<b>文件操作、进程管理、信号和用户账号等</b>。

- 环境变量：

  ```go
  /**
  	Environ: 获取所有环境变量, 返回变量列表
  	
  	Getenv: 获取指定环境变量
  	
  	Setenv: 设置环境变量
  	
  	Clearenv: 清除所有环境变量
  */
  ```

- 文件模式：

  ```go
  
  ```

- 文件信息：

  ```go
  /**
  	FileInfo: 文件信息结构体
  	
  	Stat：获取文件信息对象, 符号链接将跳转
  	
  	Lstat：获取文件信息对象, 符号链接不跳转
  	
  	IsExist：根据错误，判断 文件或目录是否存在
  	IsNotExist：是否不存在
  	
  	IsPermission：根据错误，判断是否为权限错误
  */
  
  ```

- 文件/目录操作：

  ```go
  /**
  	Getwd: 获取当前工作目录
  	
  	Chdir：修改当前工作目录
  	
  	Chmod：修改文件的 FileMode
  	
  	Chtimes：修改文件的 访问时间和修改时间
  	
  	Mkdir：创建目录
  	MkdirAll：递归创建目录
  	
  	Remove：移除文件或目录（单一文件）
  	RemoveAll：递归移除文件或目录
  	
  	Rename：文件重命名或移动
  	
  	Truncate(name, size)：修改文件大小。size:截取长度。超出源文件大小时，超出部分被无效字符填充。
  	
  	SameFile：比较两个文件信息对象， 是否指向同一文件
  	
  */
  
  
  ```

- 文件目录对象

  ```go
  /**
  	Create：创建文件, 如果文件存在，清空原文件
  	
  	Open: 打开文件，获取文件对象, 以读取模式打开
  	
  	OpenFile: 以指定模式打开文件
  	
  	Name: 获取文件路径
  */
  
  /**
  	Read：读取文件内容, 读入长度取决 容器切片长度
  	
  	ReadAt：从某位置读取文件内容
  	
  	Write：写入内容
  	
  	WriteString：写入字符
  	
  	WriteAt：从指定位置写入
  	
  	Seek(offset, whence)：设置下次读写位置(光标位置)
  	
  	Close: 关闭文件
  */
  ```

  

### os.Signal：监听信号

```go
/**
	signal.Notify: 用于监听信号
	
	singal.Stop: 用于停止监听
	
*/
```



