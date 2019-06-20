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

#### 第 2 阶段： AMS 处理 Launcher 传来的信息

#### 第 3 阶段: Launcher 休眠并通知 AMS

#### 第 4 阶段: AMS 启动新的进程

#### 第 5 阶段: 新的进程启动 以 ActivityThread 的 main 函数 为入口

#### 第 6 阶段: AMS 告诉新 App 启动哪个 Activity

#### 第 7 阶段: 启动 App 首页 Activity