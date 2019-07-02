### 跳转未在Manifest注册的Activity

- Hook上半场

    对AMN(ActivityManagerNative)进行hook
    
    ```
    public static void hookAMN() {
            //获取AMN的gDefault单例的gDefault
            Object gDefault = RefInvoke.getStaticFieldObject("android.app.ActivityManagerNative", "gDefault");
            //gDefault是一个android.util.Singleton对象,取出这个对象的mInstatnce字段
            Object mInstance = RefInvoke.getFieldValue("android.util.Singleton", gDefault, "mInstance");
            try {
                //创建代理对象，替换这个字段
                Class<?> classBInterface = Class.forName("android.app.IActivityManager");
                Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), 
                new Class[]{classBInterface}, new MockClass1(mInstance));
                RefInvoke.setFieldValue("android.util.Singleton", gDefault, "mInstance", proxy);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
    
    ```
    MockClass1的实现:
    
    ```
    
    public class MockClass1 implements InvocationHandler {
        private static final String TAG = "MockClass1";
        public static final String EXTRA_TARGET_INTENT = "extra_target_intent";
        Object target;

        public MockClass1(Object mInstance) {
            this.target = mInstance;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if ("startActivity".equalsIgnoreCase(method.getName())) {
                Intent raw;
                int index = 0;
                for (int i = 0; i < args.length; i++) {
                    if (args[i] instanceof Intent) {
                        index = i;
                        break;
                    }
                }
                raw = (Intent) args[index];
                Intent newIntent = new Intent();
                String stubPackage = raw.getComponent().getPackageName();
                //把启动activity临时替换为StunActivity
                ComponentName componentName = new ComponentName(stubPackage, StubActivity.class.getName());
                newIntent.setComponent(componentName);
                //保存原始启动activity
                newIntent.putExtra(EXTRA_TARGET_INTENT, raw);
                //替换Intent 欺骗Ams
                args[index] = newIntent;
                return method.invoke(target, args);
            }
            return method.invoke(target, args);
        }

    }
    ```
- 下半场

    方式一: 对H类的mCallback字段进行hook(跳转的Activity不能继承 AppCompatActivity?)
    
    ```
    public static void hookHandler() {
            Object sCurrentActivityThread = RefInvoke.getStaticFieldObject("android.app.ActivityThread", "sCurrentActivityThread");
            Handler mH = (Handler) RefInvoke.getFieldValue(sCurrentActivityThread.getClass(), sCurrentActivityThread, "mH");
            RefInvoke.setFieldValue(Handler.class, mH, "mCallback", new MockClass2(mH));
    }
   
    ```
    MockClass2的实现：
    ```
    public class MockClass2 implements Handler.Callback {
        Handler mbase;
        private static final String TAG = "MockClass2";
    
        public MockClass2(Handler mbase) {
            this.mbase = mbase;
        }
        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case 100:
                    handleLaunchAcitivty(msg);
                    break;
            }
            mbase.handleMessage(msg);
            return true;
        }
    
        private void handleLaunchAcitivty(Message msg) {
            Object o = msg.obj;
            Log.e(TAG, "handleLaunchAcitivty: ==>" + o);
            //替换回复真身
            Intent intent = (Intent) RefInvoke.getFieldValue(o.getClass(), o, "intent");
            Intent targetIntent = intent.getParcelableExtra(MockClass1.EXTRA_TARGET_INTENT);
            if (targetIntent != null) {
                intent.setComponent(targetIntent.getComponent());
            }
        }
    }
    ```
    示例(Activity 或者 Application 中)：
    
    ```
        @Override
        protected void attachBaseContext(Context newBase) {
            super.attachBaseContext(newBase);
            hookAMN();
            hookHandler();
        }

    ```
    
    方式二：对ActivityThread的mInstrumentation进行hook
    
    ```
    public static void hookActivityThreadInstrumentation() {
         //获取当前ActivityThread对象
         Object currentActivityThread = RefInvoke.invokeStaticMethod("android.app.ActivityThread", "currentActivityThread");
         //获取原始mInstrumentation字段
         Instrumentation mInstrumentation = (Instrumentation) RefInvoke.getFieldValue(currentActivityThread.getClass(), currentActivityThread, "mInstrumentation");
         //创建代理
         EInstrumentation2 eInstrumentation2 = new EInstrumentation2(mInstrumentation);
         //替换代理
         RefInvoke.setFieldValue(currentActivityThread.getClass(), currentActivityThread, "mInstrumentation", eInstrumentation2);
    }
        
    ```
    EInstrumentation2的实现：
    
    ```
    public class EInstrumentation2 extends Instrumentation {
        private static final String TAG = "EInstrumentation2";
        Instrumentation mBase;

        public EInstrumentation2(Instrumentation mBase) {
            this.mBase = mBase;
        }
        public Activity newActivity(ClassLoader cl, String className,
                                Intent intent) throws IllegalAccessException,
                                 ClassNotFoundException, InstantiationException {
            Intent rawIntent = intent.getParcelableExtra(MockClass1.EXTRA_TARGET_INTENT);
            if (rawIntent == null) {
            return mBase.newActivity(cl, className, intent);
            }
            String newClassName = rawIntent.getComponent().getClassName();
            return mBase.newActivity(cl, newClassName, rawIntent);
        }

    }

    ```
    示例(Activity 或者 Application 中)：
    
    ```
         @Override
            protected void attachBaseContext(Context newBase) {
                super.attachBaseContext(newBase);
                hookAMN();
                hookActivityThreadInstrumentation();
            }
    ```
    
    StubActivity:
    ```
    public class StubActivity extends AppCompatActivity {
    }

    ```
    
    