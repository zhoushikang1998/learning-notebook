## 一、基本操作

```mysql
# Windows 服务
-- 启动MySQL
	net start mysql
-- 创建Windows服务
	sc create mysql binPath= mysqld_bin_path(注意：等号与值之间有空格)
# 连接与断开服务器
mysql -h 地址 -P 端口 -u 用户名 -p 密码
SHOW PROCESSLIST 	-- 显示哪些线程正在运行
SHOW VARIABLES 		-- 显示系统变量信息
```



## 二、数据库操作

