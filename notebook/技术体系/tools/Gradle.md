### Gradle的改进

Gradle 使用一种基于 Groovy 的特定领域语言（DSL）来声明项目设置，抛弃了基于 XML 的各种繁琐配置。面向 Java应用为主，当前支持的语言限于 Java、Groovy 和 Scala。

### 1  Gradle安装配置（Windows版）

1. 官网下载gradle-xxx-bin.zip文件
2. 解压缩文件
3. 配置环境变量

### 2 Gradle 和 idea 集成



### 3 Groovy 语言简单介绍

```groovy
// 介绍 Groovy
println("hello groovy");
// groovy 可以省略语句最末尾的分号
println("hello groovy")
// 可以省略 括号
println "hello groovy"

// 如何定义变量
// def 是弱类型的，Groovy会自动根据情况来给变量赋予对应的类型
def i = 18
println i

def s = "xiaoming"
println s

// list
def list = ['a', 'b']
list << 'c'
println list.get(2)

// map
def map = ['key1':'value1', 'key2':'value2']
map.key3 = 'value3'
println map.get("key3")


// groovy中的闭包
// 什么是闭包？闭包其实就是一段代码块，在 gradle 中，主要是把闭包当参数来使用
// 定义一个闭包
def b1 = {
    println "hello b1"
}
// 定义一个方法，方法里面需要闭包类型的参数
def method1(Closure closure) {
    closure()
}
// 调用方法 method1
method1(b1)

// 定义一个带有参数的闭包
def b2 = {
    v ->
        println "hello ${v}"
}
// 定义一个方法，方法里面需要闭包类型的参数
def method2(Closure closure) {
    closure("xiaoma")
}
// 调用方法 method1
method2(b2)
```



### 4 Gradle 仓库的配置

### 5 Gradle 入门案例

### 6 Gradle 创建 Java Web 工程并在 Tomcat 下运行

### 7 Gradle 构建多模块项目

