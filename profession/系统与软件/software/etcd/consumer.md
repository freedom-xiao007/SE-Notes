```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.dromara.soul.register.server.etcd3;

import com.coreos.jetcd.Client;
import com.coreos.jetcd.KV;
import com.coreos.jetcd.Lease;
import com.coreos.jetcd.Watch;
import com.coreos.jetcd.data.ByteSequence;
import com.coreos.jetcd.data.KeyValue;
import com.coreos.jetcd.kv.DeleteResponse;
import com.coreos.jetcd.kv.GetResponse;
import com.coreos.jetcd.kv.PutResponse;
import com.coreos.jetcd.lease.LeaseGrantResponse;
import com.coreos.jetcd.options.GetOption;
import com.coreos.jetcd.options.PutOption;
import com.coreos.jetcd.options.WatchOption;
import com.coreos.jetcd.watch.WatchResponse;
import com.google.common.util.concurrent.ThreadFactoryBuilder;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.function.BiConsumer;
import java.util.function.Consumer;
import java.util.function.Supplier;
import java.util.stream.Collectors;

/**
 * @author lw1243925457
 */
public class EtcdClient {

    private static Log log = LogFactory.getLog(EtcdClient.class);

    private static final int EPHEMERAL_LEASE = 60;
    private static final int DEFAULT_CORE_POOL_SIZE = 2;
    private static final int DEFAULT_QUEUE_SIZE = 1000;

    private Client client;
    private long leaseId;

    public EtcdClient(Client client) throws ExecutionException, InterruptedException {
        this.client = client;
        initLease();
    }

    private void initLease() throws ExecutionException, InterruptedException {
        Lease lease = client.getLeaseClient();
        LeaseGrantResponse response = lease.grant(EPHEMERAL_LEASE).get();
        leaseId = response.getID();
        lease.keepAlive(leaseId);
    }

    public Node get(String key) throws ExecutionException, InterruptedException {
        log.debug(String.format("get node info of %s", key));
        KV kv = client.getKVClient();
        ByteSequence storeKey =
                Optional.ofNullable(key)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        GetResponse response = kv.get(storeKey).get();
        log.debug(response.getHeader());
        return response.getKvs().stream()
                .map(EtcdClient::kv2NodeInfo)
                .findFirst().orElse(null);

    }

    public List<Node> list(String prefix) throws ExecutionException, InterruptedException {
        log.debug(String.format("list node info of %s", prefix));
        KV kv = client.getKVClient();
        ByteSequence storePrefix =
                Optional.ofNullable(prefix)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        GetOption option = GetOption.newBuilder().withPrefix(storePrefix).build();
        GetResponse response = kv.get(storePrefix, option).get();
        return response.getKvs().stream()
                .filter(o -> !o.getKey().equals(storePrefix))
                .map(EtcdClient::kv2NodeInfo)
                .collect(Collectors.toList());
    }

    public List<String> listKeys(String prefix) throws ExecutionException, InterruptedException {
        KV kv = client.getKVClient();
        ByteSequence storePrefix =
                Optional.ofNullable(prefix)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        GetOption option = GetOption.newBuilder()
                .withKeysOnly(true)
                .withPrefix(storePrefix).build();
        GetResponse response = kv.get(storePrefix, option).get();
        return response.getKvs().stream()
                .map(o -> o.getKey().toStringUtf8())
                .filter(k -> !k.equals(prefix))
                .collect(Collectors.toList());
    }

    public long put(String key, String value) throws ExecutionException, InterruptedException {
        return put(key, value, false);
    }

    public long put(String key, String value, boolean ephemeral) throws ExecutionException, InterruptedException {
        log.debug(String.format("put %s to %s", value, key));
        KV kv = client.getKVClient();
        ByteSequence storeKey =
                Optional.ofNullable(key)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        ByteSequence storeValue =
                Optional.ofNullable(value)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        PutOption.Builder builder = PutOption.newBuilder();
        if (ephemeral) {
            builder.withLeaseId(leaseId);
        }

        PutResponse response = kv.put(storeKey,
                storeValue, builder.build()).get();
        log.info(String.format("put key-value: key: %s, reversion: %s, has-prev: %s, ephemeral: %s",
                key, response.getHeader().getRevision(), response.hasPrevKv(), ephemeral));
        return response.getHeader().getRevision();
    }

    public long delete(String key) throws ExecutionException, InterruptedException {
        log.info(String.format("delete node: %s", key));
        if (listKeys(key).size() > 0) {
//            throw new NodeNotEmptyException("Node not empty: " + key);
            System.out.println("Node not empty: " + key);
        }

        KV kv = client.getKVClient();
        ByteSequence storeKey =
                Optional.ofNullable(key)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        DeleteResponse response = kv.delete(storeKey).get();
        return response.getDeleted();
    }

    public boolean exists(String key) throws ExecutionException, InterruptedException {
        log.debug(String.format("check exists: %s", key));
        KV kv = client.getKVClient();
        ByteSequence storeKey =
                Optional.ofNullable(key)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        GetOption option = GetOption.newBuilder().withCountOnly(true).build();
        GetResponse response = kv.get(storeKey, option).get();
        log.debug(String.format("exists: %s ? : %s", key, response.getCount()));
        return response.getCount() == 1;
    }

    public void watchChildren(String key, BiConsumer<Event, Node> consumer) throws InterruptedException {
        watchChildren(key, () -> false, consumer);
    }

    public void watchChildren(String key, Supplier<Boolean> exitSignSupplier, BiConsumer<Event, Node> consumer) throws InterruptedException {
        ByteSequence storeKey =
                Optional.ofNullable(key)
                        .map(ByteSequence::fromString)
                        .orElse(null);

        try (Watch watch = client.getWatchClient();
             Watch.Watcher watcher = watch.watch(storeKey,
                     WatchOption.newBuilder().withPrefix(storeKey).build())) {
            while (!exitSignSupplier.get()) {
                WatchResponse response = watcher.listen();
                response.getEvents().forEach(watchEvent -> {
                    // 跳过根节点的变化
                    if (watchEvent.getKeyValue().getKey().equals(storeKey)) {
                        return;
                    }
                    Event event;
                    switch (watchEvent.getEventType()) {
                        case PUT:
                            event = Event.UPDATE;
                            break;
                        case DELETE:
                            event = Event.DELETE;
                            break;
                        default:
                            event = Event.UNRECOGNIZED;
                    }
                    KeyValue keyValue = watchEvent.getKeyValue();
                    Node info = kv2NodeInfo(keyValue);
                    consumer.accept(event, info);
                });
            }
        }
    }

    public void createParentNode(String parentNode) throws ExecutionException, InterruptedException {
        log.debug(String.format("check exists: %s", parentNode));
        KV kv = client.getKVClient();
        ByteSequence storeKey =
                Optional.ofNullable(parentNode)
                        .map(ByteSequence::fromString)
                        .orElse(null);
        if (exists(parentNode)) {
            return;
        }
        PutResponse response = kv.put(storeKey, ByteSequence.fromString("")).get();
        log.info(String.format("create parent nodes: %s, cluster: %s, member: %s",
                parentNode,
                response.getHeader().getClusterId(),
                response.getHeader().getMemberId()));
    }

    static Node kv2NodeInfo(KeyValue kv) {
        String key = kv.getKey().toStringUtf8();
        String value = Optional.ofNullable(kv.getValue()).map(ByteSequence::toStringUtf8).orElse("");
        return new Node(key, value, kv.getCreateRevision(), kv.getModRevision(), kv.getVersion());
    }

    public void close() throws Exception {
        Optional.ofNullable(client).ifPresent(Client::close);
    }

    static class Stoppable implements Supplier<Boolean> {
        private static final Consumer<Throwable> DEFAULT_ON_ERROR = (c) -> {
        };

        private boolean exit;


        @Override
        public Boolean get() {
            return exit;
        }

        void stop() {
            this.exit = true;
        }

    }

    private static class Holder implements BiConsumer<Event, Node> {

        @Override
        public void accept(Event event, Node node) {
            String key = node.getKey();
            System.out.println(event.toString());
            switch (event) {
                case DELETE:

                    break;
                case UPDATE:

                    break;
                default:
                    log.info(String.format("unrecognized event: %s, key: %s", event, node.getKey()));

            }
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println("start");
        ThreadPoolExecutor defaultPoolExecutor = new ThreadPoolExecutor(
                DEFAULT_CORE_POOL_SIZE, DEFAULT_CORE_POOL_SIZE * 5,
                0L, TimeUnit.NANOSECONDS,
                new ArrayBlockingQueue<>(DEFAULT_QUEUE_SIZE),
                new ThreadFactoryBuilder()
                        .setNameFormat("SuniperRegistryServiceListUpdater-%d")
                        .setDaemon(true).build());

        EtcdClient etcdClient = new EtcdClient("http://localhost:2379", 5, 3000);

        List<String> list = etcdClient.getChildren("/soul/register");
        System.out.println(list.toString());

        String soul = "/soul/register/metadata/dubbo";
        String test = "/test";
        etcdClient.watchChildren(soul, new Stoppable(), new EtcdListenHandler() {

            @Override
            public void updateHandler(String path, String value) {
                System.out.println("update: " + path + "::" + value);
            }

            @Override
            public void deleteHandler(String path, String value) {
                System.out.println("delete: " + path + "::");
            }
        });

//        etcdClient.watchChildren("/test", new Stoppable(), new Holder());

//        defaultPoolExecutor.execute(() -> {
//            try {
//                etcdClient.watchChildren("/test", new Stoppable(), new Holder());
//            } catch (Exception e) {
//                log.warn(String.format("Watch exception of %s", "/s"), e);
//            }
//        });
    }
}
```

```
Failed to execute goal on project soul-client-sofa: Could not resolve dependencies for project org.dromara:soul-client-sofa:jar:2.3.0-SNAPSHOT: Failed to collect dependencies for org.dromara:soul-client-sofa:jar:2.3.0-SNAPSHOT: Could not resolve version conflict among [org.dromara:soul-client-core:jar:2.3.0-SNAPSHOT -> org.dromara:soul-register-client-etcd:jar:2.3.0-SNAPSHOT -> io.etcd:jetcd-core:jar:0.5.0 -> io.etcd:jetcd-common:jar:0.5.0 -> io.grpc:grpc-core:jar:1.27.1, org.dromara:soul-client-core:jar:2.3.0-SNAPSHOT -> org.dromara:soul-register-client-etcd:jar:2.3.0-SNAPSHOT -> io.etcd:jetcd-core:jar:0.5.0 -> io.etcd:jetcd-resolver:jar:0.5.0 -> io.grpc:grpc-core:jar:1.27.1, org.dromara:soul-client-core:jar:2.3.0-SNAPSHOT -> org.dromara:soul-register-client-etcd:jar:2.3.0-SNAPSHOT -> io.etcd:jetcd-core:jar:0.5.0 -> io.grpc:grpc-core:jar:1.27.1, org.dromara:soul-client-core:jar:2.3.0-SNAPSHOT -> org.dromara:soul-register-client-etcd:jar:2.3.0-SNAPSHOT -> io.etcd:jetcd-core:jar:0.5.0 -> io.grpc:grpc-netty:jar:1.27.1 -> io.grpc:grpc-core:jar:[1.27.1,1.27.1], org.dromara:soul-client-core:jar:2.3.0-SNAPSHOT -> org.dromara:soul-register-client-etcd:jar:2.3.0-SNAPSHOT -> io.etcd:jetcd-core:jar:0.5.0 -> io.grpc:grpc-grpclb:jar:1.27.1 -> io.grpc:grpc-core:jar:[1.27.1,1.27.1], com.alipay.sofa:sofa-rpc-all:jar:5.7.6 -> io.grpc:grpc-netty-shaded:jar:1.28.0 -> io.grpc:grpc-core:jar:[1.28.0,1.28.0]] -> [Help 1]
[ERROR] 
```