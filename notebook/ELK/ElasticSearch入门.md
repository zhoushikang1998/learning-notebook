# ElasticSearch

## 数据结构

### ElasticSearch 会对文字进行分词

- 分出来词的集合叫做 Term Dictionary

	- Term Dictionary 会排序（查找的时候使用二分查找）

- 词对应的文档 ID 叫做 Posting List

	- Frame Of Reference（FOR）编码技术对里面的数据进行压缩，节约磁盘空间。
	- 使用 Roaring Bitmaps 求出文档 ID 的交并集。

- 由于 Term Dictionary 的词量太大，不能全部装入内存中，抽象了一层词前缀叫做 Term Index

	- Term Index 以 FST 数据结构来保存词的前缀，节省内存且查找速度快。

## ElasticSearch 架构

### ElasticSearch 相关术语

- Index

	- Table

- Type（新版本已废除）
- Document

	- Row

- Field

	- Column

- Mapping

	- Schema

- DSL

	- SQL

### 架构

- 一个主节点，多个从节点（主从模式）
- 主分片可以写入、读取；副本分片只可以读取。

	- 数据写入的时候是写到主分片，副本分片会复制主分片的数据，读取的时候主分片和副本分片都可以读。

- 一个 Index，多个分片，可以提高系统的吞吐量。
- 副本分片主要为了实现系统的高可用。

## 查询

### 查询的方式简单分成两种

- 根据 ID 查询 doc

	- 检索内存的Translog文件
	- 检索硬盘的Translog文件
	- 检索硬盘的Segement文件

- 根据 query（搜索词）去查询匹配的 doc

	- 同时去查询内存和硬盘的Segement文件

### 查询可以分为三个阶段

- QUERY_AND_FETCH

	- 查询完就返回整个 Doc 内容

- QUERY_THEN_FETCH

	- 先查询出对应的 Doc id ，然后再根据 Doc id 去匹配对应的文档

- DFS_QUERY_THEN_FETCH

	- 先算分，再查询

### 用得最多的是 QUERY_THEN_FETCH

- 向各个主分片和副本分片分发请求
- 得到各个节点返回的 doc id，组成 doc id 集合
- 再次请求各个分片拿到对应的完整 Doc

## 更新和删除

- 有一个 merge 任务，会将多个 segement 文件合并成一个 segement 文件。
- 给对应的 doc 记录打上 .del 标识，如果是删除操作就打上 delete 状态，如果是更新操作就把原来的 doc 标志为 delete，然后重新新写入一条数据。
- 在合并的过程中，会把带有 delete 状态的 doc进行物理删除。

## 写入

- 集群上的每个节点都是coordinating node（协调节点），协调节点表明这个节点可以做路由。
  - 通过hash算法可以计算出是在哪个主分片上，然后路由到对应的节点

### 写入过程

- 1.写到内存缓冲区，该内存缓冲区每秒会刷新到文件系统缓冲区（此时会生成一个 segment 文件），有 segment 文件才能被索引。
- Elasticsearch 写入的数据需要 1s 才能查询到
	
- 2.写到 translog 内存缓冲区，每隔 5s 会刷新到磁盘中。主要用作于备份（如果中途节点挂了，可以通过 translog 恢复数据），最多有 5s 的数据丢失。
- 3.隔 30 分钟或者 translog 文件过大，会执行一次 commit 操作，真正意义完成一次持久化。
