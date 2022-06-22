---
title: Laravel 应用从 MySQL 迁移到 SQLite
tags: 
- Laravel
date: 2022-06-22 08:00:00
---

阿里云提醒 RDS 要续费了，几大百一年但实际上现在只有几个小型的应用在用 MySQL。疫情之下，本着开源结流的原则，打算把 MySQL 给替换成 SQLite 以结省银子。

之前对于 SQLite 也就是了解的程度但并没有实际使用，应此在正式切换之前还需要做一些准备工作。

SQLite 和 MySQL 类似，都是一种 关系数据库管理系统 （RDBMS，Relational Database Management System）。以数据表作为基础的数据存储系统。

## SQLite 的优缺点

> Small. Fast. Reliable.

SQLite 是一款 C 编写的关系数据库。正如其名，SQLite 并非是作为一个独立的进程运行，也不需要使用特这的通信息协议与应用程序连接，而是直接作为应用程序的一部分随程序发布，这样的特性使 SQLite 非常的轻量级与易于使用，在手机、电脑、嵌入式设备、应用内嵌数据库等方面有着非常广泛的应用。

![https://www.sqlitetutorial.net/what-is-sqlite/](https://tva1.sinaimg.cn/large/e6c9d24egy1h3h1e224qkj20bb034q2u.jpg)

### 优点

- 零配置，易使用
- 可跨平台。SQLite 基于特定格式的单个文件，可移动性和跨平台特性好
- 备份容易。直接使用 `cp` 复制数据库文件即可
- 开方测试方便。基于 SQLite 的自[包含特性](https://www.sqlite.org/selfcontained.html)，使其的以来非常少，在开发过程中可作为替代手段，待上线后有需要再改为其他 RDBMS

### 缺点

- 没有用户系统
- 不支持网络访问
- 不适用于大型程序
- 提升性能的手段有限

### 应用场景

- 嵌入式设备 
- 物联设备
- 作为 excel 的替代
- 小型应用
- 小量数据分析
- 数据缓存 
- 开发和测试阶段的临时方案
- 教学目的

## MySQL 的优缺点 

MySQL 是当前最为热门的关系数据库（RDBMS），目前世界上大多数应用都在使用它。

![https://www.sqlitetutorial.net/what-is-sqlite/](https://tva1.sinaimg.cn/large/e6c9d24egy1h3h1iy6ys1j20gs06m0sw.jpg)

### 优点

- 功能强大
- 用户管理功能
- 内置更多的安全功能
- 更精细的事务和锁机制
- 更好的并发性能
- 支持网络访问

### 缺点

- 数据跨平台性差
- 可靠性问题
- 发展停滞，尽管 MySQL 仍是开源软件，但自从被收购之后发展已放缓

### 应用场景

- 分布式协作
- 大流量网站或中型应用系统
- 事务支持的程序
- 需要大量数据写入
- 存储更大规模数据量

## Laravel 应用替换

`mysql-to-sqlite3` 是一款 Python 的程序，可以将 MySQL 的数据库转换为 SQLite3 格式的数据库。

### 安装并运行转换

    pip install mysql-to-sqlite3
    mysql2sqlite --help


    mysql2sqlite -f ./sqlite.db \
    -d mysql数据库名称 \
    -u mysql数据库用户名 \
    --mysql-password mysql数据库密码 \
    -h mysql数据库地址


上面的命令可以生成 `sqlite.db` 文件，直接使用即可。


### 调整 Laravel 配置


    DB_CONNECTION=sqlite
    DB_DATABASE=/absolute/path/to/database.sqlite


直接修改 `.env` 中的 MYSQL 配置，参考上面的的就行。注意 `database.sqlite` 文件需要有可写权限。
