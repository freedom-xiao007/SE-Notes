# MongoDB 使用记录
***
## 安装配置
```sh
vim /etc/yum.repos.d/mongodb-org-4.0.repo

[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc

yum install -y mongodb-org

service mongod start

# 修改能被外界访问
vim /etc/mongodb.conf

# 启动和开机启动设置
systemctl start mongod
systemctl enable mongod
```

## 操作
### 删除collection
```sh
mongo
>use test
switched to db test
> show collections
mycol
mycollection
newcollection
>db.mycollection.drop()
>db.users.find().count()
>db.col.createIndex({"description":-1})
true
```

### 排序

### 索引
```sh
# 语法中 Key 值为你要创建的索引字段，1 为指定按升序创建索引，如果你想按降序来创建索引指定为 -1 即可。
db.collection.createIndex(keys, options)

# 查看集合索引
db.col.getIndexes()

# 查看集合索引大小
db.col.totalIndexSize()

# 删除集合所有索引
db.col.dropIndexes()

# 删除集合指定索引
db.col.dropIndex("索引名称")
```

### 去重统计
```sh
db.collection_name.distinct(field,query,options)
db.collection_name.distinct(field,query,options).length

field -----指定要返回的字段(string)
query-----条件查询(document)
```

### 不同数据库之间表复制
```sh
db.threat.find({'unit_id': '90000001'}).forEach(function(d){ db.getSiblingDB('test_center')['threat'].insert(d); });
db.comprehensiveIndexTest.find().forEach(function(d){ db.getSiblingDB('my_visual')['comprehensiveIndex'].insert(d); });
db.basedIndexTest.find().forEach(function(d){ db.getSiblingDB('my_visual')['basedIndex'].insert(d); });
db.fragileIndexTest.find().forEach(function(d){ db.getSiblingDB('my_visual')['fragileIndex'].insert(d); });
db.threadIndexTest.find().forEach(function(d){ db.getSiblingDB('my_visual')['threadIndex'].insert(d); });
```

### 数据库复制
```sh
db.copyDatabase("cx_tag","cx_tag","127.0.0.1:27017");
```

### sort()方法的size限制
```sh
db.adminCommand({setParameter:1, internalQueryExecMaxBlockingSortBytes:335544320})
```

### 模糊查询
```sh
db.getCollection('unit').find({"target":/192.168.169.0.24.*/})
```

### 导入导出

```shell
mongoimport -h 我的IP --port 端口 -d 数据库名 -c 表名 -u 帐号 -p 密码 --upsert /usr/share/nginx/html/xxx.json --authenticationDatabase admin

mongorestore -d db_name 文件夹目录
  
# 导入json
mongoimport -h 172.19.104.18 --port 20017 -d testdb -c testcol -u mytest -p 12345 --upsert /tmp/test.json --authenticationDatabase testdb
  
# 导出数据库
mongodump -h 172.19.104.18 --port 20017 -u mytest -p 12345 -d testdb -o /tmp --authenticationDatabase testdb
# 导出表
mongoexport -h 172.19.104.18 --port 20017 -u mytest -p 12345 -d testdb -c testcol -o /tmp/test.json --authenticationDatabase testdb
```

## 参考链接
- [https://robomongo.org/](https://robomongo.org/)
- [MongoDB删除集合](https://www.yiibai.com/mongodb/mongodb_drop_collection.html)
- [MongoDB 查询文档](http://www.runoob.com/mongodb/mongodb-query.html)
- [MongoDB更换数据库路径](https://www.jianshu.com/p/c43d7cf49052)
- [MongoDB中distinct的详细用法](https://blog.csdn.net/skh2015java/article/details/55667829)
- [MongoDB在不同主机间复制数据库和集合](https://blog.csdn.net/wulex/article/details/83479516)
- [mongodb——复制数据库和表](https://blog.csdn.net/hh12211221/article/details/78900098)
- [mongodb 数据库like语句查询](https://blog.csdn.net/zhouyan8603/article/details/6825834)
- [docker mongo](https://hub.docker.com/_/mongo)
- [Download Studio 3T for MongoDB](https://studio3t.com/download/)
- [ Robo 3T](https://robomongo.org/download)
- [MongoDB日常运维操作命令小结](https://www.cnblogs.com/kevingrace/p/8184087.html)