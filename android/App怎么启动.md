## App 怎么启动

   1. Launcher通知AMS,要启动App,而且指定要启动App的那个界面(首页)
   2. AMS 通知Launcher收到请求,同时将要启动的首页记录下来
   3. Launcher当前页面进入 Pause 状态,然后通知AMS,进入休眠状态,可以对要App进行操作了
   4. AMS 检查App 是否已经启动了, 是 唤起App 即可 否 启动一个新的进程,AMS在新进程中创建一个ActivityThread对象,
   启动其中的main函数
   5. App启动后,通知AMS 启动完成
   6. AMS 查找出之前存的启动页面,通知App启动那个页面
   7. App启动首页,创建Context并与首页Activity关联,然后调用首页Activity的onCreate函数
    
   启动流程可分为两部分：1-3阶段，Launcher和ASM通信，4-7阶段，app和ASM通信.

#### 第 1 阶段： Launcher 通知 AMS
   1. lanucher调用startactivity,startactivity方法经过一系列重载，会调用startActivityForResult.
   
   ```
       @Override
       public void startActivity(Intent intent, @Nullable Bundle options) {
           if (options != null) {
               startActivityForResult(intent, -1, options);
           } else {
               // Note we want to go through this call for compatibility with
               // applications that may have overridden the method.
               startActivityForResult(intent, -1);
           }
       }
   ```
   2. startActivityForResult   
    startActivityForResult 的方法实现中，会调用 Instrumentation 的 execStartActivity 方法，
    
    ```
     public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                @Nullable Bundle options) {
            if (mParent == null) {
                options = transferSpringboardActivityOptions(options);
                Instrumentation.ActivityResult ar =
                    mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
             //省略代码
            } else {
             //省略代码
            }
     } 
     
    ```   
    
   此处传递了2个重要参数：
   + 通过 ActivityThread 的 getApplicationThread() 方法获取到的一个 binder 对象，这个对象的类型
     为ApplicationThread，代表 Launcher 所在的app进程
   + mToken 也是一个 binder 对象,代表 Launcher 这个 Activity 也通过 Instrumentation 传递给 AMS,
     AMS查询后就知道是谁向AMS发起了请求
         
   这两个参数是为了以后AMS需要通知Launcher的时候,可以通过这2个参数找到Launcher
   3. Instrumentation 的 execStartActivity 方法
   
   ```
   public class Instrumentation{
        public execStartActivity(
            Context who , IBinder contextThread, IBinder token , Activity target ,
            Intent intent , int requestCode , Bundle options) {
            try {
                int result = ActivityManagerNative.getDefault()//api>=6 ActivityManager.getService()
                                    .startActivity (whoThread, who. getBasePackageName () , intent ,
                                    intent.resolveTypeifNeeded(who.getContentResolver()),
                                    token , target != null ? target . mEmbeddedID : null ,
                                    requestCode , 0, null , options);
                } catch (RemoteException e) {
                    throw new RuntimeException (” Failure from system”, e) ;
                }
            return null ;
        }
   }         
            
   ```
   借助 Instrumentation ,Activity 把参数传递个给ActivityManagerNative(Api>=26 是传递给ActivityManager);
  
   ServiceManager 是一个容器类   AMN通过getDefault()方法,从ServiceManager中取得一个名为activity的对象,
   然后将其包装成一个 ActivityManagerProxy 对象(AMP),   AMP 就是 AMS 的代理对象
   4. AMP 的 startActivity 方法
   
   AMP的startActivity方法与AIDL的Proxy方法都是写入数据到另一个进程（AMS），然后等待AMS返回结果

#### 第 2 阶段： AMS 处理 Launcher 传来的信息

   AMS检查app的 AndroidManifest 文件是否存在要启动的 Activity ,如果不存在就抛出 ActivityNotFoundException,
   无论是新启动 app 还是app内部跳转到另一个activity都会做这个检查，然后 ASM 通过 ApplicationThreadProxy 发送
   消息，launcher 则通过 ApplicationThread 来接收消息  
    
#### 第 3 阶段: Launcher 休眠并通知 AMS
   launcher 通过 ApplicationThread 接收到消息后，调用ActivityThread 的 sendMessage 方法，向 Launcher 主线程
   消息队列发送发送一个 PAUSE_ACTIVITY 消息，发送这个消息时通过名为 H 的 handler 的类来完成,然后在 H 的 handleMessage
   方法处理 PAUSE_ACTIVITY 消息
   
     public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case PAUSE_ACTIVITY: {
                 Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                 SomeArgs args = (SomeArgs) msg.obj;
                 handlePauseActivity((IBinder) args.arg1, false,(args.argi1 & USER_LEAVING) != 0, 
                                            args.argi2,(args.argi1 & DONT_REPORT) != 0, args.argi3);
                 maybeSnapshot();
                 Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                     
            } break;
        }
     }
   如上面代码，调用 ActivityThread 的 handlePauseActivity方法, 将ActivityThread 中的 mActivities 中保存
   的 Launcher 中所有打开的 Activity 找出并让其休眠，然后通过 AMP 通知 AMS 我已休眠
   

#### 第 4 阶段: AMS 启动新的进程

#### 第 5 阶段: 新的进程启动以ActivityThread的 main 函数为入口
   
   启动新的进程的时候，为这个对象创建 ActivityThread 对象(UI线程),然后立即进入 main 函数，
   1.创建一个主线程Looper,即为 MainLooper;
   
   2.创建 Application
   最后将 ActivityThread 发送给 AMS,AMS 存储这个新的App的登基信息, AMS 以后就通过这个 ActivityThread对象,
   向这个App 发送消息


#### 第 6 阶段: AMS 告诉新 App 启动哪个Activity

   AMS 把传入的 ActivityThread 对象转换为 ApplicaTionThread 对象，用于以后和这个 App 跨进程通讯
   
   AMS取出在之前存在 AMS 中要启动的Activity,通过 APT 告诉 App

#### 第 7 阶段: 启动 App 首页Activity

   App 通过 APT 接受 AMS 的消息,在H的 handleMessage() 中的 switch语句中处理，此次类型为 LAUNCH_ACTIVITY
   
      public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case PAUSE_ACTIVITY: {
                     Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                     final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                     
                     r.packageInfo = getPackageInfoNoCheck(
                         r.activityInfo.applicationInfo, r.compatInfo);
                         handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                     Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                         
                } break;
            }
      }
   getPackageInfoNoCheck 方法会提取 Apk 中的资源，然后设置 r 的 packageInfo属性，这个属性的类型叫 LoadApk ,
   在H的这个分支上,最终会调用 ActivityThread 的 handleLaunchActivity 方法，
   
   handleLaunchActivity 方法：
   
   - 通过 Instrumentation 的 newActivity() 方法,创建要启动的Activity实例
   - 为这个Activity 创建一个 上下文 Context 对象,并与Activity 进行关联
   - 通过 Instrumentation 的 callActivityOnCreate 方法,执行 Activity 的onCreate方法,从而启动 Activity