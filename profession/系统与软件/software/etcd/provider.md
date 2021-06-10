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

package org.dromars.soul.register.client.etcd3;

import io.etcd.jetcd.ByteSequence;
import io.etcd.jetcd.Client;
import io.etcd.jetcd.KeyValue;
import io.etcd.jetcd.kv.GetResponse;
import io.etcd.jetcd.kv.PutResponse;
import io.etcd.jetcd.options.GetOption;
import io.etcd.jetcd.options.PutOption;

import java.nio.charset.Charset;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

import static java.util.stream.Collectors.toList;

/**
 * @author lw1243925457
 */
public class Etcd3ClientRegisterRepository {

    private final Client client;

    public static final Charset UTF_8 = Charset.forName("UTF-8");
    private static final long TTL = 5L;
    String PATH_SEPARATOR = "/";
    private long timeout = 3000;

    public Etcd3ClientRegisterRepository(String url) {
        client = Client.builder().endpoints(url).build();
    }

    public void createPersistent(String path) {
        try {
            client.getKVClient()
                    .put(ByteSequence.from(path, UTF_8), ByteSequence.from(String.valueOf(path.hashCode()), UTF_8))
                    .get(timeout, TimeUnit.MILLISECONDS);
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            e.printStackTrace();
        }
    }

    public void createEphemeral(String path) {
        try {
            long leaseId = client.getLeaseClient().grant(TTL).get().getID();
            client.getKVClient()
                    .put(ByteSequence.from(path, UTF_8)
                            , ByteSequence.from(String.valueOf(leaseId), UTF_8)
                            , PutOption.newBuilder().withLeaseId(leaseId).build())
                    .get(timeout, TimeUnit.MILLISECONDS);
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            e.printStackTrace();
        }
    }

    public List<String> getChildren(String path) {
        try {
            int len = path.length();
            return client.getKVClient()
                    .get(ByteSequence.from(path, UTF_8),
                            GetOption.newBuilder().withPrefix(ByteSequence.from(path, UTF_8)).build())
                    .get(timeout, TimeUnit.MILLISECONDS)
                    .getKvs().stream().parallel()
                    .filter(pair -> {
                        String key = pair.getKey().toString(UTF_8);
                        int index = len, count = 0;
                        if (key.length() > len) {
                            for (; (index = key.indexOf(PATH_SEPARATOR, index)) != -1; ++index) {
                                if (count++ > 1) {
                                    break;
                                }
                            }
                        }
                        return count == 1;
                    })
                    .map(pair -> pair.getKey().toString(UTF_8))
                    .collect(toList());
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            e.printStackTrace();
        }
        return null;
    }

    public boolean put(String key, String value) {
        if (key == null || value == null) {
            return false;
        }
        CompletableFuture<PutResponse> putFuture =
                this.client.getKVClient().put(ByteSequence.from(key, UTF_8), ByteSequence.from(value, UTF_8));
        try {
            putFuture.get(timeout, TimeUnit.MILLISECONDS);
            return true;
        } catch (Exception e) {
            // ignore
        }
        return false;
    }

    public void putEphemeral(final String key, String value) {
        try {
            long leaseId = client.getLeaseClient().grant(TTL).get().getID();
            client.getKVClient()
                    .put(ByteSequence.from(key, UTF_8)
                            , ByteSequence.from(String.valueOf(value), UTF_8)
                            , PutOption.newBuilder().withLeaseId(leaseId).build())
                    .get(timeout, TimeUnit.MILLISECONDS);
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            e.printStackTrace();
        }
    }

    public String getKVValue(String key) {
        if (null == key) {
            return null;
        }

        CompletableFuture<GetResponse> responseFuture = this.client.getKVClient().get(ByteSequence.from(key, UTF_8));

        try {
            List<KeyValue> result = responseFuture.get(timeout, TimeUnit.MILLISECONDS).getKvs();
            if (!result.isEmpty()) {
                return result.get(0).getValue().toString(UTF_8);
            }
        } catch (Exception e) {
            // ignore
        }

        return null;
    }

    public static void main(String[] args) {
        Etcd3ClientRegisterRepository etcd = new Etcd3ClientRegisterRepository("http://localhost:2379");
        System.out.println("create");
        etcd.createPersistent("/test/1");
        etcd.createEphemeral("/test/2");
        System.out.println(etcd.getChildren("/test"));

        etcd.put("/test/3", "3");
        etcd.putEphemeral("/test/4", "4");
        System.out.println(etcd.getKVValue("/test/3"));
        System.out.println(etcd.getKVValue("/test/4"));
        System.out.println("end");
    }
}
```