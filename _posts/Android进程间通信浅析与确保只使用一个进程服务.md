---
title: Android进程间通信浅析与确保只使用一个进程服务
date: 2019-11-07 14:10:58
tags: 晴(立冬前一天)
---

# Android进程间通信

**1. 进程间通信，首先就是想到aidl，但是aidl文件并不是任意一个进程，它只是定义我们的app和共享进程之间的桥梁设计，里面有我们的app可以使用的方法，例如获取共享进程的版本信息、当前连接着共享进程的app个数、启动共享进程中的某个功能等等。**

**2. 这里的共享进程其实就是一个Android四大组件之一的Service，它就是通过aidl中定义的桥梁和我们的可以是很多app间进行通信**

**3. 可以在github搜索 dronekit-android，它是一套非常完美的进程间通信的示例，它的作用是建立Android设备和无人机(例如PX4)之间的连接以及数据通信。下面的一些示例很多都出自这里。**

## 一. aidl文件

好的aidl文件的示例：

    interface IDroidPlannerServices {
		int getServiceVersionCode();
		void releaseDroneApi(IDroneApi droneApi);
		int getApiVersionCode();
		IDroneApi registerDroneApi(IApiListener listener, String appId);
		Bundle[] getConnectedApps(String requesterId);
	}


定义好aidl文件后，编译即可自动生成同名的java文件，在build/generated/source/aidl目录下

**这个java文件的每一部分在使用时都至关重要**

### 1. Stub.asInterface(android.os.IBinder obj)
	
这个方法返回此java类的对象，那么它的参数从哪里传，在通过bindService方法连接共享进程的时候有一个ServiceConnection的监听，监听的方法内有一个IBinder对象，该方法就是在绑定共享进程的时候在此监听中使用：

	private IDroidPlannerServices o3drServices;

	private final AtomicBoolean isServiceConnecting = new AtomicBoolean(false);

    private final ServiceConnection o3drServicesConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            isServiceConnecting.set(false);

            o3drServices = IDroidPlannerServices.Stub.asInterface(service);
            try {
                o3drServices.asBinder().linkToDeath(binderDeathRecipient, 0);
                notifyTowerConnected();
            } catch (RemoteException e) {
                notifyTowerDisconnected();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            isServiceConnecting.set(false);
            notifyTowerDisconnected();
        }
    };

### 2. asBinder()

返回IBinder对象，使用可以看到也在上面使用，对其添加了一个app和共享进程之间的连接断开的监听

### 3. 内部静态抽象类Stub

实现aidl中定义的桥梁方法的关键类，除了一中的使用，要看它的具体实现子类DPServices，这子类中便是共享进程具体去实现aidl中定义的所有方法。 `（其实具体实现还是在共享进程<四大组件之一Service>中，DPServices只是具体的通信代理者，不像aidl只是抽象接口）`


## 二. 共享进程Service

**没什么好说的，就是干活的**

## 三. 怎么确保很多App使用的都是同一个共享进程Service，而不是又重新启动了一个新的Service，也就是正确的绑定这个服务才是最重要的

### 1. 先来看 `bindService(Intent service, @NonNull ServiceConnection conn, @BindServiceFlags int flags);` 的三个参数,第二个参数可以在 `（一）` 中的 `第1小点` 中知道如何创建及作用，第三个参数一般传 `Context.BIND_AUTO_CREATE` 常量，所以要想做好绑定都是在第一个参数Intent的处理上

### 2. 创建Intent传递一个action，标识共享进程的，在AndroidManifest.xml文件中注册是定义；通过PackageManger和这个Intent是可以查到这个共享进程的Service启动个数的，如果已经创建，可以改变Intent的设置直接使用这个已启动的Service

	Intent getAvailableServicesInstance(@NonNull final Context context) {
        final PackageManager pm = context.getPackageManager();

        //Check if an instance of the services library is up and running.
        final Intent serviceIntent = new Intent(SERVICES_CLAZZ_NAME);
        final List<ResolveInfo> serviceInfos = pm.queryIntentServices(serviceIntent, PackageManager.GET_META_DATA);
        if(serviceInfos != null && !serviceInfos.isEmpty()){
            Log.e("ServiceIntent", "serviceInfos size = " + serviceInfos.size());
            for(ResolveInfo serviceInfo : serviceInfos) {
                final Bundle metaData = serviceInfo.serviceInfo.metaData;
                if (metaData == null)
                    continue;

                final int coreLibVersion = metaData.getInt(METADATA_KEY, INVALID_LIB_VERSION);
                if (coreLibVersion != INVALID_LIB_VERSION && coreLibVersion >= VersionUtils.getCoreLibVersion(context)) {
                    serviceIntent.setClassName(serviceInfo.serviceInfo.packageName, serviceInfo.serviceInfo.name);
                    Log.e("ServiceIntent", METADATA_KEY + " = " + coreLibVersion);
                    Log.e("ServiceIntent", "serviceInfo.serviceInfo.packageName = " + serviceInfo.serviceInfo.packageName);
                    Log.e("ServiceIntent", "serviceInfo.serviceInfo.name = " + serviceInfo.serviceInfo.name);
                    return serviceIntent;
                }
            }
        }

        //Didn't find any that's up and running. Enable the local one
        DroidPlannerService.enableDroidPlannerService(context, true);
        serviceIntent.setClass(context, DroidPlannerService.class);
        Log.e("ServiceIntent", "serviceIntent action = " + serviceIntent.getAction());
        Log.e("ServiceIntent", "serviceIntent class = " + serviceIntent.getComponent().getPackageName()
                + "." + serviceIntent.getComponent().getClassName());
        return serviceIntent;
    }

### 3. 虽然是改变了Intent的设置来使用已存在的Service，但是当我们继续启动，继续查询List<ResolveInfo>的个数时，还是会发现个数在不断的增加，这里处理的关键就是上面的 `DroidPlannerService.enableDroidPlannerService(context, true);` 这句话。

	public static void enableDroidPlannerService(Context context, boolean enable){
        final ComponentName serviceComp = new ComponentName(context, DroidPlannerService.class);
        final int newState = enable ? PackageManager.COMPONENT_ENABLED_STATE_ENABLED
                : PackageManager.COMPONENT_ENABLED_STATE_DISABLED;

        context.getPackageManager().setComponentEnabledSetting(serviceComp, newState, PackageManager.DONT_KILL_APP);
    }

### 4. 上面的方法通俗的讲就是是否允许这个共享进程启动，第二个参数true即允许，false即不允许，首次启动一定设为true，如果只在这里设置，当第一个app启动成功后，不断开，后面的app都可以启动成功，但是断开后，别的app都无法绑定成功，所以在共享进程Service的onDestory生命周期中调用 `enableDroidPlannerService(getApplicationContext(), false);`