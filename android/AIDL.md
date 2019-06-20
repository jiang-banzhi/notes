## AIDL

  AIDL（Android Interface Definition Language, Android接口定义语言）
   是 Android 提供的一种进程间通信 (IPC) 机制 
 
### AIDL支持的数据类型
   
   + Java 的基本数据类型
   + List 和 Map (元素必须是AIDL支持的数据类型,Server端具体的类里则必须是ArrayList 或者 HashMap)
   + 其他 AIDL 生成的接口
   + 实现 Parcelable 的实体