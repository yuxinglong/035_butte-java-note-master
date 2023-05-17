# 一、PostgreSQL简介

**1、数据库简介**

PostgreSQL是一个功能强大的开源数据库系统，具有可靠性、稳定性、数据一致性等特点，且可以运行在所有主流操作系统上，包括Linux、Unix、Windows等。PostgreSQL是完全的事务安全性数据库，完整地支持外键、联合、视图、触发器和存储过程，支持了大多数的SQL:2008标准的数据类型，包括整型、数值型、布尔型、字节型、字符型、日期型、时间间隔型和时间型，它也支持存储二进制的大对像，包括图片、声音和视频。对很多高级开发语言有原生的编程接口API，如C/C++、Java、等，也包含各种文档。

**2、高度开源**

PostgreSQL的源代码可以自由获取，它的授权是在非常自由的开源授权下，这种授权允许用户在各种开源或是闭源项目中使用、修改和发布PostgreSQL的源代码。用户对源代码的可以按用户意愿进行任何修改、改进。因此，PostgreSQL不仅是一个强大的企业级数据库系统，也是一个用户可以开发私用、网络和商业软件产品的数据库开发平台。

# 二、Centos7下安装

**1、安装RPM**

RPM软件包管理器，一种用于互联网下载包的打包及安装工具，它包含在部分Linux分发版中。

```
yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

**2、安装客户端**

```
yum install postgresql11
```

**3、安装服务器端**

```
yum install postgresql11-server
```

**4、安装依赖包**

```
yum install postgresql11-libs
yum install postgresql11-contrib
yum install postgresql11-devel
```

**5、初始化和启动**

```
/usr/pgsql-11/bin/postgresql-11-setup initdb
systemctl enable postgresql-11
systemctl start postgresql-11
```

**6、重置密码**

```
passwd postgres
```

**7、登录服务**

```
su - postgres
psql
```

**8、安装Vim命令**

```
yum -y install vim*
```

**9、配置远程访问**

```
# 修改01
vim /var/lib/pgsql/11/data/postgresql.conf
listen_addresses = 'localhost' 
修改为
listen_addresses = '*'  

# 修改02
vim /var/lib/pgsql/11/data/pg_hba.conf
添加内容
host  all  all  0.0.0.0/0 trust ## 修改后需要重启
```

**10、开放端口**

```
firewall-cmd --query-port=5432/tcp

firewall-cmd --add-port=5432/tcp

firewall-cmd --add-port=5432/tcp --zone=public --permanent
```

**11、重新启动**

```
systemctl restart postgresql-11
```

# 三、创建数据库

**1、创建用户**

```sql
CREATE USER root01 WITH PASSWORD '123456';
CREATE ROLE;
```

**2、创建数据库**

```sql
CREATE DATABASE db_01 OWNER root01;
CREATE DATABASE;
```

**3、权限授予**

```sql
GRANT ALL PRIVILEGES ON DATABASE db_01 TO root01;
GRANT
```

**4、退出命令**

```
\q：退出SQL编辑
exit：退出脚本
```

# 四、基本操作

**1、创建表结构**

```sql
-- 用户表
CREATE TABLE pq_user (
	ID INT NOT NULL,
	user_name VARCHAR (32) NOT NULL,
	user_age int4 NOT NULL,
	create_time TIMESTAMP (6) DEFAULT CURRENT_TIMESTAMP,
	CONSTRAINT "pg_user_pkey" PRIMARY KEY ("id")
);

-- 订单表
CREATE TABLE pq_order (
	id int not null,
	user_id int not null,
	order_no varchar (32) not null,
	goods varchar (20) not null,
	price money not null,
	count_num int default 1, 
	create_time timestamp (6) default current_timestamp,
	constraint "pq_order_pkey" primary key ("id")
);
```

**2、写入数据**

```sql
INSERT INTO pq_user ("id", "user_name", "user_age", "create_time") 
VALUES ('1', 'user01', '18', '2020-04-09 19:44:57.16154');
INSERT INTO pq_order ("id", "user_id", "order_no", "goods", "price", "count_num", "create_time") 
VALUES ('1', '1', 'NO20200329652362', '书籍', '$12.20', '3', '2020-04-09 20:01:09.660208');
```

**3、常规查询**

```sql
-- 基础查询
select * from pq_user t1 where t1.id='2' and t1.user_name='user01';
select * from pq_user t1 where t1.id !='2' order by create_time desc;
-- 连接查询
select * from pq_user t1 join pq_order t2 on t1.id=t2.user_id;
select * from pq_user t1 left join pq_order t2 on t1.id=t2.user_id;
```

**4、更新和删除**

```
-- 更新数据
UPDATE pq_user SET "create_time"='2020-04-09 19:49:57' WHERE ("id"='2');
-- 删除记录
DELETE FROM pq_user WHERE "id" = 2;
```