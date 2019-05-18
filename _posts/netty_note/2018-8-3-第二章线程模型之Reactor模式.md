# 第二章 线程模型之Reactor模式



上面这幅图描述了netty的线程模型，其中mainReacotor,subReactor,Thread Pool是三个线程池。mainReactor负责处理客户端的连接请求，并将accept的连接注册到subReactor的其中一个线程上；subReactor负责处理客户端通道上的数据读写；Thread Pool是具体的业务逻辑线程池，处理具体业务。



https://www.jianshu.com/p/1633b98fff12

