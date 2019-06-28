### App安装流程

#### 为什么app安装时不把它解压?
 每次从apk中读取资源，并不是先解压在寻找资源，而是解析apk中的resources.arse文件，在这个文件中存储着资源的所有信
 息，包括资源在apk中的位置、大小等等，按图索骥，从文件中快速找到相应的资源。这是一种高效的算法。不解压apk的好处是
 节省空间。
 
#### app的安装流程
 Android 系统使用 PMS 解析apk的 AndroidManiFeast 文件：
 
 1.四大组件的信息，
 2.分配用户id和用户组id，用户id是唯一的，应为Android是一个linux系统。用户组id指的是各种权限，每个权限都在一个用
 户组中，分配了哪些用户组id就能拥有了哪些权限.
 3.在Launcher 生成一个 icon,icon中保存着默认启动Activity的信息
 4.将上面信息记录在一个xml文件中，以备下次安装时再次使用
 - android 手机每次启动时，都会使用PMS,把android系统中的所有apk都安装一遍：
 1.读取上传安装时保存的app安装信息
 2.扫描/安装在特定目录下的Apk
   + data/app-private 受DRM保护的App
   + data/app 用户自己安装的app
   + system/framework 资源型app，用于打包
   + system/app 系统自带的App
   + vender/app 设备厂商提供的app
 3.为app分配用户组id
 4.把前面的安装信息写入本地文件，下次安装时使用
 



