## Binder

  binder的目的是解决跨进程通信
  + binder分为client和server两个进程，发送消息的为client,接收消息的为server；
  + binder中的client和server不同直接通信,需要通过servermanager间接通信;
    client若要调用server的方法，需要获得server对象，但是servermanager不会返
    回server对象，只会返回server的代理对象Proxy,client调用proxy的方法，
    servermanager会帮它调用server的方法，并把结果返回给client;