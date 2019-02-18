[TOC]

## 一、影响数据库性能的因素有哪些

- 硬件资源
- 系统配置参数
- 数据库表结构
- 索引和SQL语句

## 二、如何发现有问题的SQL

### 2.1 如何发现有问题的SQL？

使用MySQL慢查询日志对有效率问题的SQL进行监控

```mysql
show variables like 'slow_query_log';
set global slow_query_log_file='';
```

### 2.2 慢查询日志的分析工具

**1、mysqldumpslow输出**

**2、pt-query-digest**

输出到文件：

pt-query-digest slow-log > slow_log.report

