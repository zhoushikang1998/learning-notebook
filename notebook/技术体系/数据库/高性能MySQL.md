## 转换表的引擎

### 1、ALTER TABLE

```mysql
mysql> ALTER TABLE mytable ENGINE = InnoDB;
```

问题：需要执行很长的时间；

### 2、导出与导入

使用 mysqldump 工具将数据导出到文件，然后修改文件中 CREATE TABLE 语句的存储引擎选项，注意同时修改表名，因为同一个数据库中不能存在相同的表明，即使使用的是不同的引擎。

**mysqldump 默认会自动在 CREATE TABLE 语句前加上 DROP TABLE 语句，不注意这点可能会导致数据丢失**

### 3、创建与查询

先创建一个新的存储引擎的表，然后利用 INSERT...SELECT 语法来导数据：

```mysql
CREATE TABLE innodb_table LIKE myisam_table;
ALTER TALBE innodb_talbe ENGINE=InnoDB;
INSERT INTO innodb_table SELECT * FROM myisam_table;
```

如果数据量很大，可以考虑做分批处理，针对每一段数据执行事务提交操作，以避免大事务产生过多的undo。

```mysql
START TRANSACTION;
INSERT INTO innodb_table SELECT * FROM myisam_table
WHERE id BETWEEN x AND y;
COMMIT;
```

