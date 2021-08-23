# MySQL使用记录
***
## 安装
### centos7
```bash
wget 'https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm'
rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
yum install mysql-community-server
systemctl start mysqld
systemctl enable mysqld

grep 'temporary password' /var/log/mysqld.log
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
grant all privileges on *.* to 'root'@'%' identified by "123456" with grant option;
flush privileges;
```

## 常用命令记录
```
create user share@'%' identified by 'password';
set password for 'share'@'%'=password('share');
grant select on bachang.index1 to share@'%' ;
grant select on bachang.index2 to share@'%' ;
revoke insert on bachang.attack_info from worker@'%';
grant select on bachang.vulnerability to worker@'%';

delete from 表名;

drop database databasename;

# MySQL查看所有连接的客户端ip
SELECT substring_index(host, ':',1) AS host_name,state,count(*) FROM information_schema.processlist GROUP BY state,host_name;
```

## 语句使用
### 建库、建表一般语句
```sql
# 用于用户数据库及表的初始化

CREATE DATABASE if not exists `mall`;

USE `mall`;

# 用户表
CREATE TABLE IF NOT EXISTS `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(16) NOT NULL,
  `password` varchar(16) NOT NULL,
  `phoneNumber` varchar(15) NOT NULL,
  `money` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;

# 商品表
CREATE TABLE IF NOT EXISTS `goods` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(16) NOT NULL,
    `description` varchar(1024) NOT NULL,
    `store_id` int(11) NOT NULL,
    `store_name` varchar(16) NOT NULL,
    `status` int(1) NOT NULL,
    PRIMARY KEY (`id`),
    foreign key (store_id) references stores(id)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=latin1;
# SQL FOREIGN KEY 约束:https://www.runoob.com/sql/sql-foreignkey.html

insert into mall.stores (name, description) VALUES ("name", "description");
insert into mall.goods(name, description, store_id, store_name, status) VALUES ("name", "description", 1, "name",
                                                                                    1);
"value"}');
```

### 初始化数据：循环插入、随机数、字符串拼接
```sql
DROP PROCEDURE IF EXISTS users_initData;
DELIMITER $
CREATE PROCEDURE users_initData()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i<=100 DO
        insert into mall.users (name, password, phoneNumber, money) VALUES (CONCAT("user", i), "password",
                                                                            CONCAT("phoneNumber", i), 0);
        SET i = i+1;
    END WHILE;
END $
CALL users_initData();

DROP PROCEDURE IF EXISTS goods_initData;
DELIMITER $
CREATE PROCEDURE goods_initData()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i<=10000 DO
        insert into mall.goods(name, description, store_id, store_name, status)
        VALUES (CONCAT("name", i), CONCAT("description", i), CEILING(rand()*100), "storeName", 1);
        SET i = i+1;
    END WHILE;
END $
CALL goods_initData();
```

### 统计相关
```sql
select timestampdiff(week,'2011-09-30','2015-05-04');
```

### 开启/关闭 安全模式
```sql
# 因为MySql运行在safe-updates模式下，该模式会导致非主键条件下无法执行update或者delete命令，执行命令如下命令关闭安全模式.
SET SQL_SAFE_UPDATES = 0;
```

## 参考链接
- [mysql-创建用户并授权，设置允许远程连接](https://www.cnblogs.com/gpdm/p/6492449.html)
- [创建MySQL用户 赋予某指定库表的权限](https://www.cnblogs.com/wuyifu/p/7580494.html)
- [Centos 7 安装 MySQL](https://www.jianshu.com/p/7cccdaa2d177)
- [centos7下mysql配置远程连接](https://blog.csdn.net/song634/article/details/80394965)
- [MySQL ALTER命令](https://www.runoob.com/mysql/mysql-alter.html)
- [mysql sum() 求和函数和TIMESTAMPDIFF时间差函数相结合的用法](https://blog.csdn.net/guo_qiangqiang/article/details/90480945)