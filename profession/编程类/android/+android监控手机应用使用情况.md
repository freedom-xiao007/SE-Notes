# Android 监控手机应用使用情况
***

这是我参与2022首次更文挑战的第19天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
本篇文章中通过Android获取手机顶部的Activity的方式，来达到监控手机应用使用情况的目的

## 技术背景
在自己的日常学习中，想要自动收集手机软件各个应用的使用时间，比如得到、极客时间、Keep、微信读书等

但在查找下，没有找到他们有对应的API开发接口，没有办法去获取

小米手机里面有手机应用使用情况，屏幕使用时间里面，数据看着挺好，挺合适的，但也不知道怎么获取

最后只能自己写一个原生的Android应用，通过获取手机顶层Activity的方式，每隔10秒上报服务器，并设置好Activity与应用名称的映射，来达到自己的手机应用使用统计目的

## 代码细节
完整代码GitHub上有：[https://github.com/lw1243925457/self_growth_android](https://github.com/lw1243925457/self_growth_android)

仅做代码参考，目前数据监控上传是有了，但界面这些还很粗糙，没有完善

使用这个功能需要注意下面的几点：

- 1.需要将监听设置为后台服务，避免切换成其他应用后频繁被kill掉
- 2.需要调用相关的权限设置，让用户开启相关的应用情况获取权限
- 3.剩下的就是应用监听代码的编写了

具体的代码如下：

### 应用监听-获取手机顶层Activity
我们需要新建一个后台服务，继承Service即可，然后开启一个定时器，每十秒获取一次顶层Activity，数据上传部分可忽略

代码如下：

```java
public class MonitorActivityService extends Service {

    private String beforeActivity;
    private final ActivityRequest activityRequest = new ActivityRequest();

    /*
     * @param intent
     * @param flags
     * @param startId
     * @return
     */
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        //处理任务
        return START_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @SuppressLint("CommitPrefEdits")
    @Override
    public void onCreate() {
        super.onCreate();
        Log.d("foreground", "onCreate");
        //如果API在26以上即版本为O则调用startForefround()方法启动服务
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            setForegroundService();
        }
    }

    /**
     * 通过通知启动服务
     * 
     * 10秒获取一次
     */
    @androidx.annotation.RequiresApi(api = Build.VERSION_CODES.O)
    public void  setForegroundService()
    {
        //设定的通知渠道名称
        String channelName = "test";
        //设置通知的重要程度
        int importance = NotificationManager.IMPORTANCE_LOW;
        //构建通知渠道
        NotificationChannel channel = new NotificationChannel("232", channelName, importance);
        channel.setDescription("test");
        //在创建的通知渠道上发送通知
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this, "232");
        builder.setSmallIcon(R.drawable.ic_launcher_foreground) //设置通知图标
                .setContentTitle("正在监控手机活动并上报")//设置通知标题
                .setContentText("正在监控手机活动并上报")//设置通知内容
                .setAutoCancel(true) //用户触摸时，自动关闭
                .setOngoing(true);//设置处于运行状态
        //向系统注册通知渠道，注册后不能改变重要性以及其他通知行为
        NotificationManager notificationManager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
        notificationManager.createNotificationChannel(channel);
        //将服务置于启动状态 NOTIFICATION_ID指的是创建的通知的ID
        startForeground(232,builder.build());

        Handler handler=new Handler();
        Runnable runnable=new Runnable(){
            @Override
            public void run() {
                Log.d("Monitor Detect", "定时检测顶层应用");
                getTopActivity();
                handler.postDelayed(this, 10000);
            }
        };
        handler.postDelayed(runnable, 10000);//每两秒执行一次runnable.
    }

    /**
     * 获取手机顶层Activity
     */
    public void getTopActivity()
    {
        long endTime = System.currentTimeMillis();
        long beginTime = endTime - 10000;
        UsageStatsManager sUsageStatsManager = (UsageStatsManager) this.getSystemService(Context.USAGE_STATS_SERVICE);
        String result = "";
        UsageEvents.Event event = new UsageEvents.Event();
        UsageEvents usageEvents = sUsageStatsManager.queryEvents(beginTime, endTime);
        while (usageEvents.hasNextEvent()) {
            usageEvents.getNextEvent(event);
            if (event.getEventType() == UsageEvents.Event.MOVE_TO_FOREGROUND) {
                result = event.getPackageName()+"/"+event.getClassName();
            }
        }
        if (!android.text.TextUtils.isEmpty(result)) {
            Log.d("Service", result);
            beforeActivity = result;
        } else {
            Log.d("Before Service", beforeActivity == null ? "null" : beforeActivity);
        }

        if (beforeActivity == null) {
            Toast.makeText(MonitorActivityService.this.getApplicationContext(),"活动为空",Toast.LENGTH_SHORT).show();
            return;
        }

        activityRequest.uploadRecord(beforeActivity, success -> {

        }, failed -> {
            Toast.makeText(MonitorActivityService.this.getApplicationContext(),"上传失败",Toast.LENGTH_SHORT).show();
            Log.w("Activity", "上传失败:" + failed);
        });
    }
}
```

### 相关权限设置
需要在配置中开启相关的权限

在AndroidManifest.xml文件中加上：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.selfgrowth">

    // 应用使用情况权限
    <uses-permission android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions"/>
    ......

    <queries>
    .......
    </queries>

    <application
    .......
        <!-- Declare foreground service -->
        <service
            android:name=".server.foreground.MonitorActivityService"
            android:enabled="true"
            android:exported="true" />
        <!-- Declare activity -->
    </application>

</manifest>
```

### 应用启动
在MainActivity中加上启动逻辑

```java
public class MainActivity extends AppCompatActivity {

    private AppBarConfiguration mAppBarConfiguration;
    private ActivityMainBinding binding;

    @RequiresApi(api = Build.VERSION_CODES.O)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
	......

        // 获取手机使用情况权限
        if (!isStatAccessPermissionSet()) {
            Intent intent = new Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS);
            this.startActivity(intent);
        }

        // Android 8.0使用startForegroundService在前台启动新服务
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
        {
            this.startForegroundService(new Intent(MainActivity.this, MonitorActivityService.class));
        }
        else{
            this.startService(new Intent(MainActivity.this, MonitorActivityService.class));
        }

        if (!isNotificationEnabled()) {
            goToNotificationSetting();
        }
    }

    /**
     * 判断允许通知，是否已经授权
     * 返回值为true时，通知栏打开，false未打开。
     */
    @RequiresApi(api = Build.VERSION_CODES.KITKAT)
    private boolean isNotificationEnabled() {

        String CHECK_OP_NO_THROW = "checkOpNoThrow";
        String OP_POST_NOTIFICATION = "OP_POST_NOTIFICATION";

        AppOpsManager mAppOps = (AppOpsManager) this.getSystemService(Context.APP_OPS_SERVICE);
        ApplicationInfo appInfo = this.getApplicationInfo();
        String pkg = this.getApplicationContext().getPackageName();
        int uid = appInfo.uid;

        Class appOpsClass = null;
        /* Context.APP_OPS_MANAGER */
        try {
            appOpsClass = Class.forName(AppOpsManager.class.getName());
            Method checkOpNoThrowMethod = appOpsClass.getMethod(CHECK_OP_NO_THROW, Integer.TYPE, Integer.TYPE,
                    String.class);
            Field opPostNotificationValue = appOpsClass.getDeclaredField(OP_POST_NOTIFICATION);

            int value = (Integer) opPostNotificationValue.get(Integer.class);
            return ((Integer) checkOpNoThrowMethod.invoke(mAppOps, value, uid, pkg) == AppOpsManager.MODE_ALLOWED);

        } catch (ClassNotFoundException | NoSuchMethodException | NoSuchFieldException | InvocationTargetException | IllegalAccessException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 跳转到app的设置界面--开启通知
     */
    private void goToNotificationSetting() {
        Intent intent = new Intent();
        if (Build.VERSION.SDK_INT >= 26) {
            // android 8.0引导
            intent.setAction("android.settings.APP_NOTIFICATION_SETTINGS");
            intent.putExtra("android.provider.extra.APP_PACKAGE", this.getPackageName());
        } else if (Build.VERSION.SDK_INT >= 21) {
            // android 5.0-7.0
            intent.setAction("android.settings.APP_NOTIFICATION_SETTINGS");
            intent.putExtra("app_package", this.getPackageName());
            intent.putExtra("app_uid", this.getApplicationInfo().uid);
        } else {
            // 其他
            intent.setAction("android.settings.APPLICATION_DETAILS_SETTINGS");
            intent.setData(Uri.fromParts("package", this.getPackageName(), null));
        }
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        this.startActivity(intent);
    }

    /**
     * Determine whether the application permission with permission to view usage has been obtained
     */
    public boolean isStatAccessPermissionSet() {
        try {
            PackageManager packageManager = this.getPackageManager();
            ApplicationInfo info = packageManager.getApplicationInfo(this.getPackageName(), 0);
            AppOpsManager appOpsManager = (AppOpsManager) this.getSystemService(APP_OPS_SERVICE);
            return appOpsManager.checkOpNoThrow(AppOpsManager.OPSTR_GET_USAGE_STATS, info.uid, info.packageName) == AppOpsManager.MODE_ALLOWED;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }
}
```

使用Android Studio启动后就可以看到相关的输出日志了