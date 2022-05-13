# Elasticsearch使用记录
***

## 错误处理记录
### ES报错 [FORBIDDEN/12/index read-only / allow delete (api)] - read only elasticsearch indices

```shell
curl --location --request PUT 'xxx.xxx.xxx.xxx:9200/index_name/_settings' \
--header 'Authorization: Basic xxxxxxxxxxx' \
--header 'Content-Type: application/json' \
--data-raw '{
    "index": {
        "blocks": {
            "read_only_allow_delete": false
        }
    }
}'
```

## 参考链接
- [ES删除全部数据的方法（Delete By Query）](https://blog.csdn.net/weixin_39198406/article/details/83016471)
- [如何从索引-ElasticSearch获取所有文档？](https://cloud.tencent.com/developer/ask/134519)
- [elasticsearch将javaAPI的搜索串打印成可以直接搜索的DSL](https://blog.csdn.net/Barbarousgrowth_yp/article/details/81535745)
- [ES查看索引库结构和数据](https://blog.csdn.net/qq_18671415/article/details/109691170)
- [Elasticsearch索引mapping的写入、查看与修改](https://blog.csdn.net/napoay/article/details/52012249)
- [ES报错 [FORBIDDEN/12/index read-only / allow delete (api)] - read only elasticsearch indices](https://blog.csdn.net/dyr_1203/article/details/85619238)