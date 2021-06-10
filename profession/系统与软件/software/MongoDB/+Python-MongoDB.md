# Python MongoDB
***
## Code
### 基本使用
```python
#!/usr/bin/env python
# @Time    : 2018/11/19 11:46
# @Author  : LiuWei
# @Site    : 
# @File    : MongoDBUtil.py
# @Software: PyCharm
# -*- coding: utf-8 -*-
import pymongo


class MongoDBUtil:
    __client = None
    __db = None
    __col = None

    @staticmethod
    def connect():
        if MongoDBUtil.__client is None:
            MongoDBUtil.__client = pymongo.MongoClient("mongodb://172.18.0.24:27017/")
            MongoDBUtil.__db = MongoDBUtil.__client["attack_packets"]
            MongoDBUtil.__col = MongoDBUtil.__db["attack"]
            print("Connected!")

    @staticmethod
    def insert_one(data):
        if MongoDBUtil.__client is None:
            MongoDBUtil.connect()
        ret = MongoDBUtil.__col.insert_one(data)
        print(ret)

    @staticmethod
    def find_one():
        if MongoDBUtil.__client is None:
            MongoDBUtil.connect()
        ret = MongoDBUtil.__col.find_one()
        # print(ret)

    @staticmethod
    def close():
        MongoDBUtil.__client.close()
        print("Close MongoDB")


if __name__ == "__main__":
    # client = pymongo.MongoClient("mongodb://172.18.0.24:27017/")
    MongoDBUtil.connect()
    MongoDBUtil.insert_one({"data": "test"})
    MongoDBUtil.find_one()
    MongoDBUtil.close()
```

### ObjectId处理
```python
from bson.objectid import ObjectId
a = ObjectId('55717c9eb2c983c127000000')
a.generation_time.timetuple() 
```

### 统计
```python
results = db.datasets.find({"test_set":"abc"}).sort("abc",pymongo.DESCENDING).skip((page-1)*num).limit(num)
results_count = results.count()
```

### 去重
```python
tags = db.mycoll.find({"category": "movie"}).distinct("tags")
```

## 参考链接
- [Python Mongodb 查询文档](http://www.runoob.com/python3/python-mongodb-query-document.html)
- [Python Mongodb](http://www.runoob.com/python3/python-mongodb.html)
- [python -【mongo】 处理ObjectID](https://blog.csdn.net/u011744758/article/details/50085013)