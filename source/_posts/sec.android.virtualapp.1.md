---
title: sec.android.virtualapp.1
date: 2019-11-28 15:15:00
categories: learning
tags: [learning,virtualapp] 
---

研究生选择了网安方向，趁着大四没事先学一些研究生阶段的知识。
导师让我看的是用容器思想做权限隔离，下面是我的学习笔记，容器这方面虽然以前听说过，但基本上是从零开始。看了老师推荐的《Android插件化开发指南》，看了一圈还是觉得从VirtualApp看起比较好。
### 前言
本文将层层分析VirtualApp的原理，由于作者说没接触过4年Android框架就不要看，那我只好慢慢看了。这篇文章也不是日后整理，而是和我研究轨迹相似。
VirtualApp对安卓底层没有任何所谓Hook，主要是用反射做了一堆工作，总的来说就是往应用和系统中间强行插了一层。在启动App后，由于uid一样，没有什么linux底层权限限制，所以可以对自己开的App内存任意读写来Hook。安卓真的是个笑话，哈哈。

首先Clone [VirtualApp](https://github.com/asLody/VirtualApp)。拿Android Studio打开，不是商业版无所谓，问过价格30万一年。NDK版本需要配置成r17c，没有的就下载一下。
![](https://i.loli.net/2019/11/28/NF5hGJTpkb7PLRs.png)
VirtualApp分lib和app，app有入口好理解，就从app开始看起。建议拿着Android Studio跟着看，不然跳来跳去都不知道在说什么。
启动Application：`io.virtualapp/io.virtualapp.VApp`
启动Activity：`io.virtualapp/io.virtualapp.splash.SplashActivity`
```java
public class VApp extends MultiDexApplication {
    ...
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        mPreferences = base.getSharedPreferences("va", Context.MODE_MULTI_PROCESS);
        VASettings.ENABLE_IO_REDIRECT = true;
        VASettings.ENABLE_INNER_SHORTCUT = false;
        try {
            VirtualCore.get().startup(base);
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
    ...
}
```
其中`VASettings`是给后面startup启动时做配置用的。随后开始配置
```java
public void startup(Context context) throws Throwable {
        ...
        detectProcessType();
        InvocationStubManager invocationStubManager = InvocationStubManager.getInstance();
        invocationStubManager.init();
        invocationStubManager.injectAll();
        ContextFixer.fixContext(context);
        ...
}
```
 对VApp，Server，Main以及被创建的子进程进行分类处理，拦截对应的代理。其中各个拦截类种类繁多，以后再看。
```java
private void injectInternal() throws Throwable {
  if (VirtualCore.get().isMainProcess()) {
    return;
  }
  if (VirtualCore.get().isServerProcess()) {
    addInjector(new ActivityManagerStub());
    addInjector(new PackageManagerStub());
    return;
  }
  if (VirtualCore.get().isVAppProcess()) {
    addInjector(new LibCoreStub());
    addInjector(new ActivityManagerStub());
    addInjector(new PackageManagerStub());
    addInjector(HCallbackStub.getDefault()); 
    ... 
  }
} 
```
这里是把各项代理先加载到一个列表里，然后再`injectAll`里注册动态代理。
```java
    /**
     * Fuck AppOps
     *
     * @param context Context
     */
    public static void fixContext(Context context) {
    	...
    }
```
这个函数不知道是在干嘛，看注释应该是针对[`App Ops`](https://github.com/8enet/AppOpsX)做了一些工作，暂且不管。随后进入`onCreate`。

```java
    @Override
    public void onCreate() {
        gApp = this;
        super.onCreate();
        VirtualCore virtualCore = VirtualCore.get();
        virtualCore.initialize(new VirtualCore.VirtualInitializer() {

            @Override
            public void onMainProcess() {
                Once.initialise(VApp.this);
                new FlurryAgent.Builder()
                        .withLogEnabled(true)
                        .withListener(() -> {
                            // nothing
                        })
                        .build(VApp.this, "48RJJP7ZCZZBB6KMMWW5");
            }

            @Override
            public void onVirtualProcess() {
                //listener components
                virtualCore.setComponentDelegate(new MyComponentDelegate());
                //fake phone imei,macAddress,BluetoothAddress
                virtualCore.setPhoneInfoDelegate(new MyPhoneInfoDelegate());
                //fake task description's icon and title
                virtualCore.setTaskDescriptionDelegate(new MyTaskDescriptionDelegate());
            }

            @Override
            public void onServerProcess() {
                virtualCore.setAppRequestListener(new MyAppRequestListener(VApp.this));
                virtualCore.addVisibleOutsidePackage("com.tencent.mobileqq");
                virtualCore.addVisibleOutsidePackage("com.tencent.mobileqqi");
                virtualCore.addVisibleOutsidePackage("com.tencent.minihd.qq");
                virtualCore.addVisibleOutsidePackage("com.tencent.qqlite");
                virtualCore.addVisibleOutsidePackage("com.facebook.katana");
                virtualCore.addVisibleOutsidePackage("com.whatsapp");
                virtualCore.addVisibleOutsidePackage("com.tencent.mm");
                virtualCore.addVisibleOutsidePackage("com.immomo.momo");
            }
        });
    }
```
这里主要是初始化`VirtualCore`，对不同的进程做不同的处理。
`MainProcess`里初始化了[`Once`](https://github.com/jonfinerty/Once)，Flurry无关紧要。`VirtualProcess`是由他启动的进程，自己创建的delegate目测是阻止调用基类的方法，`ServerProcess`是服务进程。稍微看了一下`addVisibleOutsidePackage`，`IAppManager`是`VAppManagerService`的接口。
随后去`SplashActivity.onCreate`。

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) { 
        getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN,
                WindowManager.LayoutParams.FLAG_FULLSCREEN);
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_splash);
        VUiKit.defer().when(() -> { 
            long time = System.currentTimeMillis();
            doActionInThread();
            time = System.currentTimeMillis() - time;
            long delta = 3000L - time;
            if (delta > 0) {
                VUiKit.sleep(delta);
            }
        }).done((res) -> {
            HomeActivity.goHome(this);
            finish();
        });
    }

    private void doActionInThread() {
        if (!VirtualCore.get().isEngineLaunched()) {
            VirtualCore.get().waitForEngine();
        }
    }
    
    public static void goHome(Context context) {
        Intent intent = new Intent(context, HomeActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_REORDER_TO_FRONT);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(intent);
    }
```
去除没用的就是上面的，等待Engine启动，跳转到HomeActivity。这里应该是通过`ContentProvider`调用lib的`com.lody.virtual.server.BinderProvider`来启动。

```java
public final class BinderProvider extends ContentProvider {  
	...
    @Override
    public boolean onCreate() {
        Context context = getContext();
        DaemonService.startup(context);
        if (!VirtualCore.get().isStartup()) {
            return true;
        }
        VPackageManagerService.systemReady();
        IPCBus.register(IPackageManager.class, VPackageManagerService.get());
        VActivityManagerService.systemReady(context);
        IPCBus.register(IActivityManager.class, VActivityManagerService.get());
        IPCBus.register(IUserManager.class, VUserManagerService.get());
        VAppManagerService.systemReady();
        IPCBus.register(IAppManager.class, VAppManagerService.get());
        BroadcastSystem.attach(VActivityManagerService.get(), VAppManagerService.get());
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            IPCBus.register(IJobService.class, VJobSchedulerService.get());
        }
        VNotificationManagerService.systemReady(context);
        IPCBus.register(INotificationManager.class, VNotificationManagerService.get());
        VAppManagerService.get().scanApps();
        VAccountManagerService.systemReady();
        IPCBus.register(IAccountManager.class, VAccountManagerService.get());
        IPCBus.register(IVirtualStorageService.class, VirtualStorageService.get());
        IPCBus.register(IDeviceInfoManager.class, VDeviceManagerService.get());
        IPCBus.register(IVirtualLocationManager.class, VirtualLocationService.get());
        return true;
    } 
    ...
}
```
注册了一堆东西，随后进入`HomeActivity.onCreate`

```java
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        overridePendingTransition(0, 0);
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_home);
        mUiHandler = new Handler(Looper.getMainLooper());
        bindViews();
        initLaunchpad();
        initMenu();
        new HomePresenterImpl(this).start();
    }
```
`initMenu()`是右上角的菜单，`initLaunchpad()`是选择App启动。
### 参考资料
##### 项目
[YAHFA](https://github.com/rk700/YAHFA)
[VirtualXposed](https://github.com/android-hacker/VirtualXposed)
[VirtualApp](https://github.com/asLody/VirtualApp)
[epic](https://github.com/tiann/epic)
[App Ops](https://github.com/8enet/AppOpsX)
[Once](https://github.com/jonfinerty/Once)
##### 博客
[http://weishu.me/](http://weishu.me) 
[https://blog.csdn.net/leif_](https://blog.csdn.net/leif_)
[http://rk700.github.io/2017/03/15/virtualapp-basic/](http://rk700.github.io/2017/03/15/virtualapp-basic/)

