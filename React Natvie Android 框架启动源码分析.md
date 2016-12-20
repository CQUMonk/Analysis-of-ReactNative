#React Natvie Android 框架启动源码分析

##Overview
`ReactNative`作为一种跨平台开发技术，在Native上下文Context以及运行时Runtime的基础上，启动JavaScript运行时完成布局，再由Native端完成渲染工作。其间涉及了大量的`Native <---> JS`的通信机制。而`ReactNative`框架正是基于该通信机制，完成JS与Native组件和接口之间的调用，从而保证了`ReactNative`业务逻辑高效有序的执行。
整个`ReactNative`框架的启动和初始化过程，其实就是整个JS与Native之间通信机制建立的过程，本文从`ReactAndroid V0.32`源码角度对这个过程进行了详细分析，为了便于阅读，我对源码进行了适当的裁剪。
在开始分析之前，首先对于RN框架中的几个重要的核心类进行介绍：
> - `ReactRootView`是RN界面的最顶层的View，主要负责进行监听分发事件，渲染元素等工作。它可以通过监听View的尺寸从而使UImanager可以对其子元素进行重新布局。


> - `ReactInstanceManager`（以下简称`RIM`）主要被用于管理`CatalystInstance`实例对象。通过`ReactPackage`，它向开发人员提供了配置和使用`CatalystInstance`实例对象的方法，并对该catalyst实例的生命周期进行追踪。`ReactInstanceManager`实例对象的生命周期与`ReactRootView`所在的activity保持一致。

##`ReactInstanceManager`的创建
在`ReactNative`业务开发中，所有与之相关的Activity均继承自`ReactActivity`。与其他Android activity一样，当跳转到
`ReactNative`业务界面后，将回调其`onCreate`方法，这里主要进行了如下工作：
> 1. 创建`ReactRootView`和`ReactInstanceManager`对象.
> 2. 调用startReactApplication方法启动`ReactNative`的运行时环境.
> 3. 将`ReactRootView`挂载到当前Activity上.

在启动`ReactNative`运行时环境之前，首先要创建`ReactRootView`和`ReactInstanceManager`对象。
`ReactRootView`对象的创建与普通View的创建并无差异。创建RIM对象时则需要调用`ReactNativeHost`的`createReactInstanceManager()`方法来创建。

根据具体的业务需求，开发者可以在`ReactNativeHost`中对自定义组件，开发模式，JSBundle路径等进行配置，并利用builder模式完成RIM的创建和初始化工作：
```java
protected ReactInstanceManager createReactInstanceManager() {
    ReactInstanceManager.Builder builder = ReactInstanceManager.builder()
      .setApplication(mApplication)
      .setJSMainModuleName(getJSMainModuleName())
      .setUseDeveloperSupport(getUseDeveloperSupport())
      .setInitialLifecycleState(LifecycleState.BEFORE_CREATE);

    for (ReactPackage reactPackage : getPackages()) {
      builder.addPackage(reactPackage);
    }

    String jsBundleFile = getJSBundleFile();
    if (jsBundleFile != null) {
      builder.setJSBundleFile(jsBundleFile);
    } else {
      builder.setBundleAssetName(Assertions.assertNotNull(getBundleAssetName()));
    }

    return builder.build();
}
```
方法中首先对创建`ReactInstanceManager`的builder进行配置，设置`JSmodule`入口名称：`JSMainModuleName`（debug模式有效），是否使用开发者模式：`DeveloperSupport`，初始化生命周期，并加入封装了自定义组件的package，以及自定义`JSBundle`文件的路径（如果开发者未指定JSBundle的路径，默认情况下RN则会从Android的asset目录下加载`index.android.js`文件）。  
配置完毕后对`ReactInstanceManager`进行创建，ReactAndroid0.32中RIM的实现类是`XReactInstanceManagerImpl`, 除了必要的配置参数外，在其构造函数中还初始化了`SoLoader`和`DevSupportManager`：
```java
initializeSoLoaderIfNecessary(applicationContext);

mDevSupportManager = DevSupportManagerFactory.create(
        applicationContext,
        mDevInterface,
        mJSMainModuleName,
        useDeveloperSupport,
        redBoxHandler);
```


----------
`ReactActivity`将创建好的RIM作为参数传入`startReactApplication`方法，同时被传入的还有入口模块名称和初始化的组件属性:
```java
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
    UiThreadUtil.assertOnUiThread();

    mReactInstanceManager = reactInstanceManager;
    mJSModuleName = moduleName;//要启动的JSModule名称
    mLaunchOptions = launchOptions;//启动时传入JS端的可选参数
	//判断当前Activity是否创建过ReactContext
    if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
      mReactInstanceManager.createReactContextInBackground();
    }
	//首次挂载该View时才触发measure过程，只有首次measure后才会将该rootview attach 到RIM上
    if (mWasMeasured) {
      attachToReactInstanceManager();
    }
  }
```


----------


##RN运行时上下文环境`ReactContext`的创建
从上面代码中可以看到，当Activity首次启动后，通过RIM的`createReactContextInBackground`方法开始创建ReactNative的上下文环境`ReactContext`（`createReactContextInBackground`方法只会在RN应用启动时候被调用，而主动重新加载JS的时候则是调用方法`recreateReactContextInBackground`）.  在RIM的实现类`XReactInstanceManagerImpl`中，最终会调用到`recreateReactContextInBackgroundInner`方法：
```java
private void recreateReactContextInBackgroundInner() {
    UiThreadUtil.assertOnUiThread();

    if (mUseDeveloperSupport && mJSMainModuleName != null) {//Debug模式且设置了入口module的文件名(index.android)
      final DeveloperSettings devSettings = mDevSupportManager.getDevSettings();

      // If remote JS debugging is enabled, load from dev server.
      if (mDevSupportManager.hasUpToDateJSBundleInCache() &&
          !devSettings.isRemoteJSDebugEnabled()) {//cache中有最新的JSbundle,且未开启JS远程Debug
        // If there is a up-to-date bundle downloaded from server,
        // with remote JS debugging disabled, always use that.
        onJSBundleLoadedFromServer();
      } else if (mBundleLoader == null) {
        mDevSupportManager.handleReloadJS();
      } else {
        mDevSupportManager.isPackagerRunning(
            new DevServerHelper.PackagerStatusCallback() {
              @Override
              public void onPackagerStatusFetched(final boolean packagerIsRunning) {
                UiThreadUtil.runOnUiThread(
                    new Runnable() {
                      @Override
                      public void run() {
                        if (packagerIsRunning) {
                          mDevSupportManager.handleReloadJS();
                        } else {
                          // If dev server is down, disable the remote JS debugging.
                          devSettings.setRemoteJSDebugEnabled(false);
                          recreateReactContextInBackgroundFromBundleLoader();
                        }
                      }
                    });
              }
            });
      }
      return;
    }
    //非Debug模式
    recreateReactContextInBackgroundFromBundleLoader();
}
```
通过对ReactNative所运行的模式进行判断，如果处于开发者模式下，则从Package Server加载JSBundle文件，否则从本地路径下获取：
```java
//Debug模式下从Package Server加载
private void onJSBundleLoadedFromServer() {
    recreateReactContextInBackground(
        new JSCJavaScriptExecutor.Factory(mJSCConfig.getConfigMap()),
        JSBundleLoader.createCachedBundleFromNetworkLoader(
            mDevSupportManager.getSourceUrl(),
            mDevSupportManager.getDownloadedJSBundleFile()));
}
```
```java
//本地路径加载
private void recreateReactContextInBackgroundFromBundleLoader() {
    recreateReactContextInBackground(
        new JSCJavaScriptExecutor.Factory(mJSCConfig.getConfigMap()),
        mBundleLoader);
}
```
从上面代码中可以看到，通过传入`recreateReactContextInBackground`方法的参数`JSBundleLoader`的不同从而实现从不同位置获取JS文件。`recreateReactContextInBackground`方法接受两个参数：
>- `JSCJavaScriptExecutor`: 位于C++层的JavaScript解析器。当类加载器加载`JSCJavaScriptExecutor.class`时会加载两个so文件，在它的构造函数里会调用JNI方法来对JSCJavaScriptExecutor进行初始化。
>- `JSBundleLoader ` :被用于存储要加载的JS bundle的信息，帮助`catalystInstance`可以通过`ReactBridge`来加载bundle文件。  

在`recreateReactContextInBackground`方法中，RIM创建并开始执行异步任务`ReactContextInitAsyncTask`，该异步任务主要负责在后台完成对RN上下文环境`ReactContext`进行创建：
```java
private void recreateReactContextInBackground(
      JavaScriptExecutor.Factory jsExecutorFactory,
      JSBundleLoader jsBundleLoader) {
    UiThreadUtil.assertOnUiThread();

    ReactContextInitParams initParams =
        new ReactContextInitParams(jsExecutorFactory, jsBundleLoader);
    if (mReactContextInitAsyncTask == null) {
      // No background task to create react context is currently running, create and execute one.
      mReactContextInitAsyncTask = new ReactContextInitAsyncTask();
      mReactContextInitAsyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, initParams);
    } else {
      // Background task is currently running, queue up most recent init params to recreate context
      // once task completes.
      mPendingReactContextInitParams = initParams;
    }
}
```
从上面代码中可以看到，RIM首先为创建`ReactContext`的初始化参数`ReactContextInitParams`，然后根据`ReactContextInitAsyncTask`对象是否存在来判断是否有创建上下文的后台任务在运行。如果当前后台有创建任务正在执行，则将参数入队等待，在当前异步任务执行完毕后再执行。
`ReactContextInitAsyncTask`负责在AsyncTask中创建react context，每次只能执行一个创建任务。在任务执行前，首先会判断当前是否存在React context，如果有则进行销毁重建，保证同一时刻只有一个React context存在：
```java
//UI线程
protected void onPreExecute() {
      if (mCurrentReactContext != null) {
        tearDownReactContext(mCurrentReactContext);
        mCurrentReactContext = null;
      }
}
```
异步任务执行过程是在后台线程中进行的，首先会创建出JSExecutor,然后调用`createReactContext`方法来完成`ReactContext`的创建工作：
```java
//后台线程
protected Result<ReactApplicationContext> doInBackground(ReactContextInitParams... params) {
      Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);

      Assertions.assertCondition(params != null && params.length > 0 && params[0] != null);
      try {
        JavaScriptExecutor jsExecutor = params[0].getJsExecutorFactory().create();
        return Result.of(createReactContext(jsExecutor, params[0].getJsBundleLoader()));
      } catch (Exception e) {
        // Pass exception to onPostExecute() so it can be handled on the main thread
        return Result.of(e);
      }
}
```
执行完成后重新回到UI线程，将创建好的`ReactApplicationContext`交给UI线程进行后续的处理。UI线程中的处理工作我们暂且不表，首先看一下异步任务`ReactContextInitAsyncTask`到底在后台做了哪些工作。
在上面的`doInBackground`方法中，首先会创建出`JSExecutor`,然后调用`createReactContext`方法进行`ReactContext`的创建工作。在`createReactContext`方法中主要完成了5项工作：

>1. 处理modules package
>1. 建立Native模块的注册表
>1. 创建catalystInstance实例对象
>1. 创建并初始化`ReactApplicationContext`
>1. 执行JsBundle
下面针对这5方面工作进行深入分析。

### I. 处理`ReactPackage`

根据`React Native`的组件化思想，所有的Native组件和JS组件都要分别继承`NativeModule`和`JavaScriptModule`类，并且都需要在`CatalystInstance`中进行注册。Native组件和JS组件被封装进相应的`ReactPackage`中：
>- `NativeModule`: Native Module负责向JS端提供可调用的接口，这些接口由Native端实现。`ReactPackage`提供了`createNativeModules`方法来负责创建该`ReactPackage`中所封装的模块和组件的实例对象。
>- `JavaScriptModule`：JS Module负责向Native端提供可调用的接口，该接口具体实现由JS端负责。在Native端的`JavaScriptModule`接口仅仅负责标识该模块是JS module。`ReactPackage`提供了`createJSModules`方法来返回该`ReactPackage`总包含的JS Module的Class类。

`CoreModulesPackage`定义并组装了核心框架模块（例如UImanager），通常被用于那些需要与框架其他组件集成的模块。
RIM在这里分别对核心package`CoreModulesPackage`和用户自定义的`ReactPackage`进行处理：
```java
//创建Native Module 与JS Module注册表的builder
NativeModuleRegistry.Builder nativeRegistryBuilder = new NativeModuleRegistry.Builder();
JavaScriptModuleRegistry.Builder jsModulesBuilder = new JavaScriptModuleRegistry.Builder();
.......
//处理RN框架的核心module
CoreModulesPackage coreModulesPackage =
          new CoreModulesPackage(this, mBackBtnHandler, mUIImplementationProvider);
processPackage(coreModulesPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);

.......
for (ReactPackage reactPackage : mPackages) {//处理自定义module
	processPackage(reactPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
}
```


在`processPackage`方法中有两个for循环，要分别对`CoreModulesPackage`和用户自定义的Native和JS模块进行处理：
```java
private void processPackage(
      ReactPackage reactPackage,
      ReactApplicationContext reactContext,
      NativeModuleRegistry.Builder nativeRegistryBuilder,
      JavaScriptModuleRegistry.Builder jsModulesBuilder) {
    for (NativeModule nativeModule : reactPackage.createNativeModules(reactContext)) {
      nativeRegistryBuilder.add(nativeModule);
    }
    for (Class<? extends JavaScriptModule> jsModuleClass : reactPackage.createJSModules()) {
      jsModulesBuilder.add(jsModuleClass);
    }
}
```
其处理方式实际上就是创建出Native module的实例对象列表和JS module的Class列表，然后将其传递给各自注册表的builder备用。在`NativeRegistryBuilder`的add方法中：
```java
public Builder add(NativeModule module) {
      NativeModule existing = mModules.get(module.getName());
      if (existing != null && !module.canOverrideExistingModule()) {
        throw new IllegalStateException("... ");
      }
      mModules.put(module.getName(), module);
      return this;
}
```
`NativeRegistryBuilder`使用`HashMap<ModuleName, NativeModule>`来存放module信息，而`JavaScriptModuleRegistryBuilder`则使用了`List<JavaScriptModuleRegistration>`，其中`JavaScriptModuleRegistration`只是一个封装了JSmodule Class的对象。


### II. 模块的注册工作
利用builder，RIM分别建立Native模块与JS模块的注册表`NativeModulesRegistry`、`JavaScriptModuleRegistry`。首先来看`NativeModulesRegistry`的创建：  

```Java
public NativeModuleRegistry build() {
      Map<Class<NativeModule>, NativeModule> moduleInstances = new HashMap<>();
      for (NativeModule module : mModules.values()) {
        moduleInstances.put((Class<NativeModule>)module.getClass(), module);
      }
      return new NativeModuleRegistry(moduleInstances);
}
```

可以看到，nativeRegistryBuilder将之前存于`HashMap<ModuleName，NativeModule>`中的Native模块取出，重新封装进`HashMap<ClassName,NativeModule>`中后传递给`NativeModuleRegistry`的构造方法：
```java
private NativeModuleRegistry(Map<Class<NativeModule>, NativeModule> moduleInstances) {
    mModuleInstances = moduleInstances;
    mBatchCompleteListenerModules = new ArrayList<OnBatchCompleteListener>(mModuleInstances.size());
    for (NativeModule module : mModuleInstances.values()) {
      if (module instanceof OnBatchCompleteListener) {
        mBatchCompleteListenerModules.add((OnBatchCompleteListener) module);
      }
    }
}
```
构造方法中，取出那些JS调用Native层方法结束后需要接收通知的module封装进一个List中。
同样，`JavaScriptModuleRegistry.Builder`也根据JS模块的注册信息创建出JS注册表`JavaScriptModuleRegistry`：
```java
public JavaScriptModuleRegistry(List<JavaScriptModuleRegistration> config) {
    mModuleInstances = new WeakHashMap<>();
    mModuleRegistrations = new HashMap<>();
    for (JavaScriptModuleRegistration registration : config) {
      mModuleRegistrations.put(registration.getModuleInterface(), registration);
    }
}
```
`JavaScriptModuleRegistry`首先创建了一个`WeakHashMap`和一个`HashMap<Class<? extends JavaScriptModule>, JavaScriptModuleRegistration>`。然后对后者进行了初始化，实际上就是把JSModule的Class对象和封装该类对象的`JavaScriptModuleRegistration`分别作为<K,V>放入`HashMap`中。

### III. 创建并初始化`catalystInstance`实例对象

`CatalystInstance`是一个封装了异步JSC的顶级API，它提供了JS和Java之间交互的环境。`CatalystInstance`的实现类为`CatalystInstanceImpl`。
RIM对`CatalystInstanceImpl`对象进行了封装，从而帮助开发者通过RIM对它进行操作。`CatalystInstance`初始化时构建了`ReactBridge`，从而通过所持有的`ReactBridge`（JNI）来实现Native和JS的通信工作。  
在创建`catalystInstance`的实例对象时，同样是利用builder模式。首先构建builder：
```java
CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
        .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
        .setJSExecutor(jsExecutor)
        .setRegistry(nativeModuleRegistry)
        .setJSModuleRegistry(jsModulesBuilder.build())
        .setJSBundleLoader(jsBundleLoader)
        .setNativeModuleCallExceptionHandler(exceptionHandler);
```
在builder中，我们可以为CatalystInstance设置以下组件：
-  `ReactQueueConfigurationSpec`：该类包含了创建的消息队列的配置信息ReactQueueConfiguration的参数；
- `NativeModuleRegistry`与`JSmoduleRegistry`:Native模块与JS模块的注册表；
- `JSBundleLoader`：该类存储了JSbundle的信息，保证CatalystInstance通过ReactBridge加载正确的bundle；
- `NativeModuleCallExceptionHandler`负责处理JS调用native接口所引发的异常；

在`CatalystInstanceImpl`的构造方法中，首先根据之前传入的`ReactQueueConfigurationSpec`参数创建了一个`ReactQueueConfiguration`对象：
```java
mReactQueueConfiguration = ReactQueueConfigurationImpl.create(
        ReactQueueConfigurationSpec,
        new NativeExceptionHandler());
```
这个类中封装了RN框架中通信的三个消息队列线程：
```JAVA
MessageQueueThread getUIQueueThread();
MessageQueueThread getNativeModulesQueueThread();
MessageQueueThread getJSQueueThread();
```
`MessageQueueThread`中封装了一个线程，该线程持有一个looper，可以接受Runnable/Callable对象来执行。  
除了创建消息队列线程，还将调用JNI方法初始化Hybrid数据，以及完成对`ReactBridge`的初始化工作。  
之前对于ReactBridge的初始化，RN在Java层进行了实现，而在0.32之后的版本中则选择使用JNI调用Cpp的代码来实现。

### IV. 创建并初始化`ReactApplicationContext`

`ReactApplicationContext`是一个Context Wapper，里面封装了Android的Application Context以及`CatalystInstance`对象，以及RN框架处理消息的三个线程：
```java
final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);
```
创建好的`catalystInstance`对象将通过所持有的JS线程执行ReactContext的初始化工作：  
```java
catalystInstance.getReactQueueConfiguration().getJSQueueThread().callOnQueue(
        new Callable<Void>() {
          @Override
          public Void call() throws Exception {
            reactContext.initializeWithInstance(catalystInstance);

            
            try {
              catalystInstance.runJSBundle();
            }
            return null;
          }
        }).get();
```
`ReactContext`的初始化工作实际上就是把创建好的`catalystInstance`对象以及该对象所持有的三个消息队列线程传递给`ReactContext`。


### V. 运行JS Bundle
异步任务的最后一项工作是在JS线程中通过调用`catalystInstance`对象的runJSBundle方法，对JS bundle进行加载和解析:
```java
public void runJSBundle() {
    mAcceptCalls = true;
    mJSBundleHasLoaded = true;
    mJSBundleLoader.loadScript(CatalystInstanceImpl.this);
}
```
而`JSBundleLoader`最终会调用到`CatalystInstanceImpl`的JNI方法来加载JSBundle文件，这个过程是异步的。
`ReactContext`创建和初始化完毕后，后台执行的异步任务终于结束了。
`ReactContextInitAsyncTask`的`onPostExecute`方法将在主线程中回调，负责进行后续的处理工作。为了便于理解，我们假定异步任务在后台执行期间，`ReactActivity`中setContentView方法还没有执行完毕，也就是说RootView的尺寸大小还未被测量。
通常情况下，异步任务的回调方法通常会在主线程空闲后才执行，这里为了衔接方便，分析了`onPostExecute`方法立即被回调的情况。
异步任务的`onPostExecute`方法进行收尾工作。首先由RIM调用`setupReactContext`方法启动刚刚创建好的`ReactContext`：
```java
private void setupReactContext(ReactApplicationContext reactContext) {

    UiThreadUtil.assertOnUiThread();

    mCurrentReactContext = Assertions.assertNotNull(reactContext);
    CatalystInstance catalystInstance =
        Assertions.assertNotNull(reactContext.getCatalystInstance());

    catalystInstance.initialize();
    mDevSupportManager.onNewReactContextCreated(reactContext);
    mMemoryPressureRouter.addMemoryPressureListener(catalystInstance);
    moveReactContextToCurrentLifecycleState();

    for (ReactRootView rootView : mAttachedRootViews) {
      attachMeasuredRootViewToInstance(rootView, catalystInstance);
    }

.......


  }
```
在`setupReactContext`方法中，RIM首先将调用`catalystInstance`的`initialize()`方法对Native模块进行初始化：
```Java
 public void initialize() {
    UiThreadUtil.assertOnUiThread();
    mInitialized = true;
    mJavaRegistry.notifyCatalystInstanceInitialized();
  }
```
在UI线程中，`notifyCatalystInstanceInitialized`方法调用各自的initialize方法，从而完成Native模块的初始化工作。
```JAVA
  /* package */ void notifyCatalystInstanceInitialized() {
    UiThreadUtil.assertOnUiThread();
    try {
      for (NativeModule nativeModule : mModuleInstances.values()) {
        nativeModule.initialize();
      }
    }
  }
```
RIM对象维护了一个列表来存储与之关联的`ReactRootView`。由于我们假定当前RootView的尺寸仍未被测量，因此还未被添加进该列表中，关联方法`attachMeasuredRootViewToInstance`也不会被执行。
`setupReactContext`执行完毕后，当前`ReactContextInitAsyncTask`异步任务实例也被销毁掉，如果队列中没有新的异步任务请求，那么异步任务的收尾工作就此完成。

----------
##启动JS Application
在异步任务创建上下文环境`ReactContext`的同时，`ReactActivity`在主线程中对`ReactRootView`进行测量，回调其`onMeasure`方法：
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    .....
    mWasMeasured = true;
    // Check if we were waiting for onMeasure to attach the root view
    if (mReactInstanceManager != null && !mIsAttachedToInstance) {
      // Enqueue it to UIThread not to block onMeasure waiting for the catalyst instance creation
      UiThreadUtil.runOnUiThread(new Runnable() {
        @Override
        public void run() {
          attachToReactInstanceManager();
        }
      });
    }
}
```
测量完成后，判断RIM创建成功，`ReactRootView`最终调用了RIM的`attachMeasuredRootView`方法，将自己和刚刚创建好的RIM实例对象关联起来：
```java
public void attachMeasuredRootView(ReactRootView rootView) {
    UiThreadUtil.assertOnUiThread();
    mAttachedRootViews.add(rootView);

    if (mReactContextInitAsyncTask == null && mCurrentReactContext != null) {
      attachMeasuredRootViewToInstance(rootView, mCurrentReactContext.getCatalystInstance());
    }
}
```
RIM对象维护了一个列表来存储与之关联的`ReactRootView`。RIM把当前RootView添加至该列表后，如果在后台线程中进行的异步任务已经将上下文环境`ReactContext`创建完毕，则将其与对应的`CatalystInstance`相关联：

```java
private void attachMeasuredRootViewToInstance(
      ReactRootView rootView,
      CatalystInstance catalystInstance) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachMeasuredRootViewToInstance");
    UiThreadUtil.assertOnUiThread();

    rootView.removeAllViews();
    rootView.setId(View.NO_ID);

    UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class);
    int rootTag = uiManagerModule.addMeasuredRootView(rootView);
    rootView.setRootViewTag(rootTag);
   
    ......
}
```
RIM首先重置了该`RootView`下的所有内容（删除所有子View），其中的内容随后将被JS提供的界面所填充。
接下来，RIM获取到Native模块`UIManagerModule`，并调用其`addMeasuredRootView`方法，把`RootView`注册到该模块。
>`UIManagerModule`作为一个Native module主要负责JS创建和更新Native View。

在`addMeasuredRootView`方法中，`UIManagerModule`首先为RootView生成了一个Tag值，可以被JS端所用于管理其中子View：
```java
public int addMeasuredRootView(final SizeMonitoringFrameLayout rootView) {
    final int tag = mNextRootViewTag;
    mNextRootViewTag += ROOT_VIEW_TAG_INCREMENT;

    final int width;
    final int height;

    if (rootView.getLayoutParams() != null &&
        rootView.getLayoutParams().width > 0 &&
        rootView.getLayoutParams().height > 0) {
      width = rootView.getLayoutParams().width;
      height = rootView.getLayoutParams().height;
    } else {
      width = rootView.getWidth();
      height = rootView.getHeight();
    }

    final ThemedReactContext themedRootContext =
        new ThemedReactContext(getReactApplicationContext(), rootView.getContext());

    mUIImplementation.registerRootView(rootView, tag, width, height, themedRootContext);

    rootView.setOnSizeChangedListener(
      new SizeMonitoringFrameLayout.OnSizeChangedListener() {
        @Override
        public void onSizeChanged(final int width, final int height, int oldW, int oldH) {
          getReactApplicationContext().runOnNativeModulesQueueThread(
            new Runnable() {
              @Override
              public void run() {
                updateRootNodeSize(tag, width, height);
              }
            });
        }
      });

    return tag;
}
```
在获取到RootView的尺寸后，将其注册给`UIImplementation`.
>`UIImplementation`负责接收来自JS端的调用命令并将其转换为shadow node层级，随后被映射为真实的Native View层级关系

回到RIM的`attachMeasuredRootViewToInstance`方法中，RIM获取到JSMoudle`AppRegistry`，设置好初始化参数后，调用其方法`runApplication`启动JS application。
```java
@Nullable Bundle launchOptions = rootView.getLaunchOptions();
    WritableMap initialProps = Arguments.makeNativeMap(launchOptions);
    String jsAppModuleName = rootView.getJSModuleName();

    WritableNativeMap appParams = new WritableNativeMap();
    appParams.putDouble("rootTag", rootTag);
    appParams.putMap("initialProps", initialProps);
    catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
```
此时，RN的上下文环境`ReactContext`以及随后的收尾工作基本完成。


----------


