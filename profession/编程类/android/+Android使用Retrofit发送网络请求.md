# Android 使用 Retrofit 发送网络请求
***

这是我参与2022首次更文挑战的第20天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
在Android应用中，如果不是单机的话，应该都有请求后端接口API的情况，本篇文章就介绍下Retrofit在Android中如何进行使用的

## 相关代码
我们以一个简单的登录接口为例

完整代码GitHub上有：[https://github.com/lw1243925457/self_growth_android](https://github.com/lw1243925457/self_growth_android)

仅做代码参考，目前数据监控上传是有了，但界面这些还很粗糙，没有完善

### 相关的依赖引入
首先我们在工程中引入相关的依赖：

```gradle
    implementation 'com.squareup.okhttp3:okhttp:4.5.0'
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
```

### 相关的手机权限开启
需要在文件中开启网络权限：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.selfgrowth">

    <uses-permission android:name="android.permission.INTERNET"/>

    <application
    ......
    </application>

</manifest>
```

### 配置Retrofit Client
Client的相关配置：单例，配置基于OKHTTP，Gson序列化;OKHTTP中添加了请求拦截器

```java
@Data
public class RetrofitClient {

    private static final RetrofitClient instance = new RetrofitClient();

    private final Retrofit retrofit = new Retrofit.Builder()
            .baseUrl(HttpConfig.ADDRESS) //基础url,其他部分在GetRequestInterface里
            .client(httpClient())
            .addConverterFactory(GsonConverterFactory.create()) //Gson数据转换器
            .build();

    public static RetrofitClient getInstance() {
        return instance;
    }

    private OkHttpClient httpClient() {
        return new OkHttpClient.Builder()
                .addInterceptor(new AccessTokenInterceptor())
                .connectTimeout(20, TimeUnit.SECONDS)
                .build();
    }
}
```

### 配置通用的请求拦截器
比如在请求中，带上Authorization等

```java
public class AccessTokenInterceptor implements Interceptor {

    @NonNull
    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        if (UserCache.getInstance().getToken() == null) {
            return chain.proceed(chain.request());
        }

        Request original = chain.request();
        Request.Builder requestBuilder = original.newBuilder()
                .addHeader("Authorization", UserCache.getInstance().getToken());
        Request request = requestBuilder.build();
        return chain.proceed(request);
    }
}
```

### Retrofit接口定义
登录请求的接口定义：

```java
public interface UserApi {

    /**
     * 用户登录
     **/
    @POST("auth/user/login")
    Call<ApiResponse> login(@Body LoginUser user);
}
```

### Retrofit Request具体请求编写
我们首先定义一个抽象类，在其中持有我们的RetrofitClient全局类，在其中发起请求，由于Android UI的形式，请求是异步的

```java
public abstract class Request {

    final Retrofit retrofit;

    public Request() {
        this.retrofit = RetrofitClient.getInstance().getRetrofit();
    }

    /**
     * 发送网络请求(异步)
     * @param call call
     */
    void sendRequest(Call<ApiResponse> call, Consumer<? super Object> success, Consumer<? super Object> failed) {
        call.enqueue(new Callback<ApiResponse>() {
            @Override
            public void onResponse(Call<ApiResponse> call, Response<ApiResponse> response) {
                if (response.code() != 200) {
                    Log.w("Http Response", "请求响应错误");
                    failed.accept(response.raw().message());
                    return;
                }
                if (response.body() == null || response.body().getData() == null) {
                    success.accept(null);
                    return;
                }
                Object res = response.body().getData();
                if (String.valueOf(res).isEmpty()) {
                    success.accept(null);
                    return;
                }
                success.accept(res);
            }

            @Override
            public void onFailure(Call<ApiResponse> call, Throwable t) {
                System.out.println("GetOutWarehouseList->onFailure(MainActivity.java): "+t.toString() );
            }
        });
    }
}
```

如上所示，请求成功就执行success相关的逻辑，失败则执行failed相关的逻辑

登录请求的具体逻辑如下：构造Retrofit Interface，发起请求

```java
public class UserRequest extends Request {

    public void login(LoginUser user, Consumer<? super Object> success, Consumer<? super Object> failed) {
        UserApi request = retrofit.create(UserApi.class);
        Call<ApiResponse> call = request.login(user);
        sendRequest(call, success, failed);
    }
}
```

### Android UI中进行调动
使用示例如下，点击一个登录按钮后触发

```java
public class LoginFragment extends Fragment {

    private final UserRequest userRequest = new UserRequest();

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View rootView = inflater.inflate(R.layout.fragment_login, container, false);
        Button loginButton = rootView.findViewById(R.id.login_button);
        loginButton.setOnClickListener(view -> {
            EditText email = rootView.findViewById(R.id.login_email_edit);
            EditText password = rootView.findViewById(R.id.login_password_edit);

            final LoginUser user = LoginUser.builder()
                    .email(email.getText().toString())
                    .password(password.getText().toString())
                    .build();

            // 获取相关的用户名和密码后，调用登录接口
            userRequest.login(user, (token) -> {
                UserCache.getInstance().initUser(email.getText().toString(), token.toString());
                final SharedPreferences preferences = requireContext().getSharedPreferences("userInfo", Context.MODE_PRIVATE);
                final SharedPreferences.Editor edit = preferences.edit();
                edit.putString("username", email.getText().toString());
                edit.putString("password", password.getText().toString());
                edit.apply();
                Snackbar.make(view, "登录成功:" + token.toString(), Snackbar.LENGTH_LONG)
                        .setAction("Action", null).show();
            }, failedMessage -> {
                Snackbar.make(view, "登录失败:" + failedMessage, Snackbar.LENGTH_LONG)
                        .setAction("Action", null).show();
            });
        });
        return rootView;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        final SharedPreferences preferences = getActivity().getSharedPreferences("userInfo", Activity.MODE_PRIVATE);
        final String userName = preferences.getString("username", "");
        final String password = preferences.getString("password", "");
        final EditText emailEdit = getView().findViewById(R.id.login_email_edit);
        final EditText passwordEdit = getView().findViewById(R.id.login_password_edit);
        emailEdit.setText(userName);
        passwordEdit.setText(password);
    }
}
```

## 总结
本篇文章中介绍了如Android学习中如何使用Retrofit发起网络请求

但由于吃初学，虽然感觉能用，但有点繁琐，不知道在实际的Android开发中，网络请求的最近实践是怎么样的，如果有的话，大佬可以在评论区告知下，感谢