---
title: 备份 MySQL 的数据到 Dropbox
date: '2015-03-22 01:16:20'
tags:
- 新技能GET
---

### 导出 MySQL 的数据

#### 使用命令行 (mysqldump) 导出数据

执行以下命令
```shell
$ mysqldump --opt -u [uname] -p[pass] [dbname] > [backupfile.sql]
```

- [uname] 数据用户名
- [pass] 数据密码
- [dbname] 数据库名称
- [backupfile.sql] 备份文件
- [—opt] 其他选项
     - —add-drop-table 在导出SQL中的 CREATE TABLE 语句前增加 DROP TABLE 语句
     - —no-data 只导出表结构
     - —add-locks 在导出的 SQL 中增加 LOCK TABLES 和 UNLOCK TABLES 语句
     - --single-transaction 在导出数据时不执行锁表操作, 避免导出用户没有锁表权限

#### 压缩导出的数据文件

```shell
$ mysqldump --opt -u [uname] -p[pass] [dbname] | gzip -9 > [backupfile.sql.gz]
```

### 恢复导出的数据到 MySQL

在确认目标数据库已经创建的情况下, 执行以下命令
```shell
$ mysql -u [uname] -p[pass] [db_to_restore] < [backupfile.sql]
```

### 将备份的数据库数据上传到 Dropbox

Dropbox 提供丰富的 API 接口, 而官方的客户端过于庞大, 推荐一个轻量级的命令行上传工具 Dropbox-Uploader
https://github.com/andreafabrizi/Dropbox-Uploader

运行 Dropbox-Uploader 脚本, 按照提示创建 Dropbox 应用. 填入 App Key 和 App Secret, 并且在浏览器上授权应用的访问权限, 就可以使用其命令上传和下载文件了.

```shell
$ dropbox_uploader upload [sourcefile] [remotefile]
```

### 优化流程

有了以上的工具, 我们希望更方便的使用或者自动化运行, 可以

- 编写命令行小工具整合备份和上传的操作
- 添加到定时任务中定时执行