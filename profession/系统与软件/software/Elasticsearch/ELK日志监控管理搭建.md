# ELK 日志监控管理搭建
***

```sh
docker pull docker.elastic.co/logstash/logstash:7.5.2
```

Filebeat

```sh
docker pull docker.elastic.co/beats/filebeat:7.5.2
docker run \
docker.elastic.co/beats/filebeat:7.5.2 \
setup -E setup.kibana.host=192.168.110.109:5601 \
-E output.elasticsearch.hosts=["192.168.110.109:9200"] \
-E cloud.auth=elastic:BitWorker.2020
```


## 参考链接

- [Logstash Reference 7.5](https://www.elastic.co/guide/en/logstash/7.5/index.html)
- [ Getting Started With Filebeat](https://www.elastic.co/guide/en/beats/filebeat/7.5/filebeat-getting-started.html)
- [ Running Filebeat on Docker ](https://www.elastic.co/guide/en/beats/filebeat/7.5/running-on-docker.html)
- [Step 2: Configure Filebeat](https://www.elastic.co/guide/en/beats/filebeat/7.5/filebeat-configuration.html)