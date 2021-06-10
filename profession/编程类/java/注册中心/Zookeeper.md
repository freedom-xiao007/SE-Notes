# Java Zookeeper
***

```xml
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

```java
public class ZookeeperRegisterUtils implements RegisterUtils, AutoCloseable {

    private final String ROOT = "/soul/register/";

    private final Gson gson = new Gson();

    private ZkClient zkClient;

    public ZookeeperRegisterUtils(String zookeeperUrl, int zookeeperSessionTimeout, int zookeeperConnectionTimeout) {
        this.zkClient = new ZkClient(zookeeperUrl, zookeeperSessionTimeout, zookeeperConnectionTimeout);
    }

    @Override
    public void doRegister(String json, String url, RpcTypeEnum rpcTypeEnum) {
        System.out.println("zookeeper register");
        Map jsonObject = gson.fromJson(json, Map.class);
        String name = jsonObject.get("contextPath").toString() + jsonObject.get("path").toString();
        updateZkNode(ROOT + name.replace("/", "."), json);
    }

    private void updateZkNode(String path, String json) {
        if (zkClient.exists(path)) {
            zkClient.delete(path);
        }
        zkClient.createEphemeral(path, json);
    }

    @Override
    public void close() throws Exception {
        System.out.println("Zookeeper register close");
        zkClient.deleteRecursive(ROOT);
    }
}
```

```java
public class ZookeeperRegister {

    private final ZkClient zkClient;

    private final String ROOT = "/soul/register";

    public ZookeeperRegister(final ZkClient zkClient) {
        System.out.println("register zk start");
        this.zkClient = zkClient;
        watch();
    }

    private void watch() {
        zkClient.subscribeChildChanges(ROOT, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String dataPath: currentChildren) {
                    System.out.println("add watch::" + ROOT + "/" + dataPath);
                    watcher(ROOT + "/" + dataPath);
                }
            }
        });
    }

    private void watcher(String dataPath) {
        zkClient.subscribeDataChanges(dataPath, new IZkDataListener() {

            @Override
            public void handleDataChange(final String dataPath, final Object data) {
                System.out.println(dataPath + "::" + data);
            }

            @Override
            public void handleDataDeleted(final String dataPath) {
                System.out.println(dataPath);
            }
        });
    }
}
```