# Java Elasticsearch 使用
***
## 概览
基于官方教程，提供Elasticsearch的基本操作，并编写单元测试：

- 客户端连接
- 客户端关闭
- 建立索引
- 删除索引
- 插入文档
  - 单个文档的同步和异步插入
  - 多个文档的同步和异步插入
- 查询文档

本地Docker的Elasticsearch搭建可参考：https://juejin.cn/post/6965321555475693582/

## 代码细节
### Elasticsearch封装类

```java
import lombok.SneakyThrows;
import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.support.master.AcknowledgedResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.*;
import org.elasticsearch.client.indices.CreateIndexRequest;
import org.elasticsearch.client.indices.CreateIndexResponse;
import org.elasticsearch.client.indices.GetIndexRequest;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.MatchQueryBuilder;
import org.elasticsearch.rest.RestStatus;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.sort.SortBuilder;

import java.io.IOException;
import java.util.List;

/**
 * official document: https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.12/java-rest-high-document-bulk.html
 *
 * @author liuwei
 *
 * 自动重连说明：经过测试，在ES重启完成后，相关的操作便可恢复正常，无序额外操作
 */
public class ElasticClient {

    final private RestHighLevelClient client;

    /**
     * 构造Elasticsearch客户端
     * @param userName 用户名
     * @param password 密码
     * @param hosts 集群主机地址
     */
    public ElasticClient(final String userName, final String password, final HttpHost[] hosts) {
        UsernamePasswordCredentials credentials = new UsernamePasswordCredentials(userName, password);
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, credentials);

        client = new RestHighLevelClient(
                RestClient.builder(hosts)
                        .setHttpClientConfigCallback(builder -> builder.setDefaultCredentialsProvider(credentialsProvider)));
    }

    /**
     * 关闭客户端连接
     * @throws IOException IOException
     */
    public void close() throws IOException {
        client.close();
    }

    /**
     * 检测索引是否已建立
     * @param indexNames 索引名
     * @return bool
     */
    @SneakyThrows
    public boolean checkIndexExist(final String... indexNames) {
        GetIndexRequest request = new GetIndexRequest(indexNames);
        return client.indices().exists(request, RequestOptions.DEFAULT);
    }

    /**
     * 新建索引
     * @param indexName 索引名称
     * @param setting 索引配置内容（Json格式）
     * @return bool
     */
    @SneakyThrows
    public boolean createIndex(String indexName, String setting) {
        CreateIndexRequest request = new CreateIndexRequest(indexName);
        request.source(setting, XContentType.JSON);
        CreateIndexResponse createIndexResponse = client.indices().create(request, RequestOptions.DEFAULT);
        return createIndexResponse.isAcknowledged();
    }

    /**
     * 删除索引
     * @param indexName 删除索引
     * @return bool
     */
    @SneakyThrows
    public boolean deleteIndex(String indexName) {
        DeleteIndexRequest request = new DeleteIndexRequest(indexName);
        AcknowledgedResponse deleteIndexResponse = client.indices().delete(request, RequestOptions.DEFAULT);
        return deleteIndexResponse.isAcknowledged();
    }

    /**
     * 新增文档：同步阻塞
     * @param indexName 索引名
     * @param doc 文档
     * @param id 文档id
     * @return bool
     */
    @SneakyThrows
    public boolean addOneDocSync(String indexName, String doc, String id) {
        UpdateRequest request = new UpdateRequest(indexName, id).doc(doc, XContentType.JSON).upsert(doc, XContentType.JSON);
        UpdateResponse updateResponse = client.update(request, RequestOptions.DEFAULT);
        return updateResponse.status().equals(RestStatus.CREATED);
    }

    /**
     * 新增文档：异步非阻塞
     * @param indexName 索引名
     * @param doc 文档
     * @param id 文档id
     */
    @SneakyThrows
    public void addOneDocAsync(String indexName, String doc, String id) {
        ActionListener<UpdateResponse> listener = new ActionListener<UpdateResponse>() {
            @Override
            public void onResponse(UpdateResponse updateResponse) {

            }

            @Override
            public void onFailure(Exception e) {

            }
        };

        UpdateRequest request = new UpdateRequest(indexName, id).doc(doc, XContentType.JSON).upsert(doc, XContentType.JSON);
        client.updateAsync(request, RequestOptions.DEFAULT, listener);
    }

    /**
     * 批量新增文档：同步阻塞
     * @param indexName 索引名
     * @param docs 文档列表
     * @param ids ID列表
     * @return bool
     */
    @SneakyThrows
    public boolean addManyDocSync(String indexName, List<String> docs, List<String> ids) {
        if (docs.size() != ids.size()) {
            return false;
        }

        BulkRequest request = createBulkRequest(indexName, docs, ids);
        BulkResponse bulkResponse = client.bulk(request, RequestOptions.DEFAULT);
        return bulkResponse.status().equals(RestStatus.OK);

    }

    /**
     * 批量新增文档：异步非阻塞
     * @param indexName 索引名
     * @param docs 文档列表
     * @param ids ID列表
     * @return bool
     */
    @SneakyThrows
    public boolean addManyDocAsync(String indexName, List<String> docs, List<String> ids) {
        if (docs.size() != ids.size()) {
            return false;
        }

        ActionListener<BulkResponse> listener = new ActionListener<BulkResponse>() {
            @Override
            public void onResponse(BulkResponse bulkResponse) {

            }

            @Override
            public void onFailure(Exception e) {

            }
        };

        BulkRequest request = createBulkRequest(indexName, docs, ids);
        client.bulkAsync(request, RequestOptions.DEFAULT, listener);
        return true;
    }

    private BulkRequest createBulkRequest(String indexName, List<String> docs, List<String> ids) {
        BulkRequest request = new BulkRequest();
        for (int i = 0; i < docs.size(); i++) {
            String doc = docs.get(i);
            String id = ids.get(i);
            UpdateRequest r = new UpdateRequest(indexName, id).doc(doc, XContentType.JSON).upsert(doc, XContentType.JSON);
            request.add(r);
        }
        return request;
    }

    /**
     * 文档查询
     * @param pageIndex 分页：跳过的页数，从0开始
     * @param pageSize 分页：每页的数据量
     * @param queryCondition 查询条件
     * @param sortBuilders 排序条件
     * @param indexNames 索引名
     * @return {@link SearchHit} list
     */
    @SneakyThrows
    public SearchHit[] queryDoc(int pageIndex, int pageSize, MatchQueryBuilder queryCondition,
                                List<SortBuilder<?>> sortBuilders, String... indexNames) {
        SearchRequest searchRequest = new SearchRequest(indexNames);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(queryCondition);
        searchSourceBuilder.sort(sortBuilders);
        searchSourceBuilder.from(pageIndex * pageSize);
        searchSourceBuilder.size(pageSize);
        searchRequest.source(searchSourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        return searchResponse.getHits().getHits();
    }
}
```

### 测试类
Junit5测试

```java
import lombok.SneakyThrows;
import org.apache.commons.io.FileUtils;
import org.apache.http.HttpHost;
import org.elasticsearch.index.query.MatchQueryBuilder;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.sort.FieldSortBuilder;
import org.elasticsearch.search.sort.SortOrder;
import org.junit.*;

import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.List;

@Ignore
public class ElasticClientTest {

    final private String index = "test_index";

    private ElasticClient elasticClient;

    @SneakyThrows
    @Test
    public void test() {
        testConnect();
        testAddIndex();
        testAddOneDoc();
        testAddManyDoc();

        Thread.sleep(5000);
        testQuery();

        testReconnect();
        testDeleteIndex();
        testClose();
    }

    private void testConnect() {
        String host = "localhost";
        int port = 9200;
        String scheme = "http";
        final HttpHost[] hosts = new HttpHost[] {
                new HttpHost(host, port, scheme)
        };
        String userName = "elastic";
        String password = "password";
        elasticClient = new ElasticClient(userName, password, hosts);

        if (elasticClient.checkIndexExist(index)) {
            assert elasticClient.deleteIndex(index);
        }
    }

    private void testAddIndex() throws IOException {
        String indexSettingFilePath = "TaskAndRobotLogIndex.json";
        File file=new File(indexSettingFilePath);
        String indexSetting = FileUtils.readFileToString(file,"UTF-8");
        assert elasticClient.createIndex(index, indexSetting);
        assert elasticClient.checkIndexExist(index);
    }

    private void testAddOneDoc() {
        LogModel oneAsyncLog = createLogModel("taskIdOneAsync");
        elasticClient.addOneDocAsync(index, oneAsyncLog.toJsonString(), "taskIdOneAsync");

        LogModel oneSyncLog = createLogModel("taskIdOneSync");
        assert elasticClient.addOneDocSync(index, oneSyncLog.toJsonString(), "taskIdOneSync");
    }

    private void testAddManyDoc() {
        List<String> logModelsAsync = new ArrayList<>(2);
        logModelsAsync.add(createLogModel("taskIdManyAsync1").toJsonString());
        logModelsAsync.add(createLogModel("taskIdManyAsync2").toJsonString());

        List<String> idsAsync = new ArrayList<>(2);
        idsAsync.add("taskIdManyAsync1");
        idsAsync.add("taskIdManyAsync2");

        assert elasticClient.addManyDocAsync(index, logModelsAsync, idsAsync);

        List<String> logModelsSync = new ArrayList<>(2);
        logModelsSync.add(createLogModel("taskIdManySync1").toJsonString());
        logModelsSync.add(createLogModel("taskIdManySync2").toJsonString());

        List<String> idsSync = new ArrayList<>(2);
        idsSync.add("taskIdManySync1");
        idsSync.add("taskIdManySync2");

        assert elasticClient.addManyDocSync(index, logModelsSync, idsSync);
    }

    private LogModel createLogModel(String id) {
        return LogModel.builder()
                .message("test")
                .id(id)
                .createTime(new Date())
                .build();
    }

    private void testQuery() {
        MatchQueryBuilder queryCondition = new MatchQueryBuilder("id", "id");
        FieldSortBuilder createTimeSort = new FieldSortBuilder("createTime").order(SortOrder.ASC);
        SearchHit[] results = elasticClient.queryDoc(0, 100, queryCondition,
                Collections.singletonList(createTimeSort), index);
        Assert.assertEquals(0, results.length);

        queryCondition = new MatchQueryBuilder("message", "test");
        createTimeSort = new FieldSortBuilder("createTime").order(SortOrder.ASC);
        results = elasticClient.queryDoc(0, 100, queryCondition,
                Collections.singletonList(createTimeSort), index);
        Assert.assertEquals(6, results.length);
        for (SearchHit result : results) {
            System.out.println(result.toString());
        }

        queryCondition = new MatchQueryBuilder("id", "taskIdManySync1");
        createTimeSort = new FieldSortBuilder("createTime").order(SortOrder.ASC);
        results = elasticClient.queryDoc(0, 100, queryCondition,
                Collections.singletonList(createTimeSort), index);
        Assert.assertEquals(1, results.length);
        for (SearchHit result : results) {
            System.out.println(result.toString());
        }
    }

    private void testReconnect() throws InterruptedException {
        MatchQueryBuilder queryCondition = new MatchQueryBuilder("message", "test");
        FieldSortBuilder createTimeSort = new FieldSortBuilder("createTime").order(SortOrder.ASC);
        SearchHit[] results = elasticClient.queryDoc(0, 100, queryCondition,
                Collections.singletonList(createTimeSort), index);
        Assert.assertEquals(6, results.length);
        for (SearchHit result : results) {
            System.out.println(result.toString());
        }

        System.out.println("Waiting ES restart");
        Thread.sleep(1000 * 30);

        queryCondition = new MatchQueryBuilder("message", "test");
        createTimeSort = new FieldSortBuilder("createTime").order(SortOrder.ASC);
        results = elasticClient.queryDoc(0, 100, queryCondition,
                Collections.singletonList(createTimeSort), index);
        Assert.assertEquals(6, results.length);
        for (SearchHit result : results) {
            System.out.println(result.toString());
        }
    }

    private void testDeleteIndex() {
        assert elasticClient.deleteIndex(index);
        assert !elasticClient.checkIndexExist(index);
    }

    private void testClose() throws IOException {
        elasticClient.close();
    }
}
```

数据模型

```java
import com.alibaba.fastjson.JSONObject;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;

import java.util.Date;

/**
 * @author lw1243925457
 */
@Builder
@Data
@AllArgsConstructor
public class LogModel {
    private String message;
    private String id;
    private Date createTime;

    public String toJsonString() {
        return JSONObject.toJSONString(this);
    }
}
```

索引配置文件：

```json
{
  "mappings":{
    "properties":{
      "message":{
        "type":"text",
        "analyzer":"ik_max_word",
        "search_analyzer":"ik_smart"
      },
      "id":{
        "type":"keyword"
      },
      "createtime":{
        "type":"date",
        "format": "yyyy-MM-dd HH:mm:ss.SSS"
      }
    },
    "_source":{
      "enabled":true
    }
  },
  "settings":{
    "number_of_shards":2,
    "number_of_replicas":2
  }
}
```

## 参考链接
### 教程
- [官方参考文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/7.12/java-rest-high-create-index.html#java-rest-high-create-index-request-mappings)
- [maven](https://search.maven.org/search?q=g:org.elasticsearch.client)
- [maven detail](https://search.maven.org/artifact/org.elasticsearch.client/elasticsearch-rest-client/7.12.1/jar)
- [kibana查询及可视化操作](https://www.jianshu.com/p/918d88fc6b09)
- [ElasticSearch Java查询某字段既不为null也不为空的条目](https://blog.csdn.net/qq_30146831/article/details/106649800)

### 问题
- [解决elasticsearch client的maven中httpclient版本冲突的问题](https://blog.csdn.net/wupan6688/article/details/88667593)
- [Spring Boot with Elastic Search causing java.lang.NoSuchFieldError: IGNORE_DEPRECATIONS](https://stackoverflow.com/questions/62338588/spring-boot-with-elastic-search-causing-java-lang-nosuchfielderror-ignore-depre)