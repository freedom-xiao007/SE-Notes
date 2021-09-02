# OKHttp3 使用记录
***
### 使用Cookie登录后发起请求
```java
package com.ninetech.cloud.bw.rpa.service.task.dispatch.http;

import com.google.gson.Gson;
import com.ninetech.cloud.bw.rpa.service.task.dispatch.feign.XxlJobInfo;
import com.ninetech.cloud.bw.rpa.service.task.dispatch.feign.XxlJobResponse;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import okhttp3.*;
import org.springframework.stereotype.Service;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
@Service
public class XxlJobOkHttp3Client {

    private final String XXL_JOB_ADDRESS = "http://127.0.0.1:8080/xxl-job-admin";
    private final Gson gson = new Gson();

    private final OkHttpClient httpClient = new OkHttpClient.Builder().cookieJar(new CookieJar() {

        private final HashMap<String, List<Cookie>> cookieStore = new HashMap<>();

        @Override
        public void saveFromResponse(HttpUrl httpUrl, List<Cookie> list) {
            cookieStore.put(httpUrl.host(), list);
        }

        @Override
        public List<Cookie> loadForRequest(HttpUrl httpUrl) {
            List<Cookie> cookies = cookieStore.get(httpUrl.host());
            return cookies != null ? cookies : new ArrayList<>();
        }
    }).build();

    public XxlJobOkHttp3Client() {
        login();
    }

    private void login() {
        FormBody.Builder formBody = new FormBody.Builder();
        formBody.add("userName", "admin");
        formBody.add("password", "123456");
        sendRequest(formBody, "/login");
    }

    @SneakyThrows
    private XxlJobResponse sendRequest(final FormBody.Builder formBody, final String url) {
        final Request request = new Request.Builder()
                .url(XXL_JOB_ADDRESS + url)
                .post(formBody.build())
                .build();
        final Call call = httpClient.newCall(request);
        final Response response = call.execute();
        final ResponseBody body = response.body();
        final String bodyStr = body.string();
        log.info(bodyStr);
        assert bodyStr.contains("200");
        return gson.fromJson(bodyStr, XxlJobResponse.class);
    }

    @SneakyThrows
    public long addTask(final XxlJobInfo jobInfo) {
        FormBody.Builder formBody = new FormBody.Builder();
        for (Field field : jobInfo.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            if (field.get(jobInfo) != null) {
                formBody.add(field.getName(), String.valueOf(field.get(jobInfo)));
            }
        }
        final XxlJobResponse res = sendRequest(formBody, "/jobinfo/add");
        return Long.parseLong(res.getContent().toString());
    }

    public boolean startTask(final long id) {
        FormBody.Builder formBody = new FormBody.Builder();
        formBody.add("id", String.valueOf(id));
        sendRequest(formBody, "/jobinfo/start");
        return true;
    }

    public static void main(String[] args) {
        XxlJobOkHttp3Client client = new XxlJobOkHttp3Client();
        client.login();
    }
}
```

## 参考链接
- [使用Gson解析Okhttp3返回的结果报错：Exception: closed](https://blog.csdn.net/u014521739/article/details/93486077)
- [使用HttpClient和OkHttp实现模拟登录方正教务系统](https://www.cxyzjd.com/article/starexplode/81058988)
- [A Quick Guide to Post Requests with OkHttp](https://www.baeldung.com/okhttp-post)