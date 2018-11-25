为了获取到thirft client公网地址进行的二次开发
修改了thrift了源码，用于采集系统服务端获取客户端公网地址

从服务端的套接字里面获取到了客户端的源IP即公网地址



###网络编程模式
arg.selectorThreads(Integer.parseInt(mProp.get("LogServerSelectorThread").toString()));
这步骤是启动了多个线程，每个线程里面有个bocking queue队列，队列元素是socketchannel，线程启动后就不断消费这个队列
并不是select使用了多线程，而是便利selectkey时，没当有一个连接socketchannel进来就加入队列，


arg.workerThreads(Integer.parseInt(mProp.get("LogServerWorkThread").toString()));  //worker线程数
这一步其实是处理selector的连接数，内部使用了ExcuteService这个线程池，
每天有个连接进来，就是用从这个线程池里面任意取一个线程来执行下面这个方法，目的是分别client到一个指定SelectorThread线程里面
invoker.submit(new Runnable() { public void run() { doAddAccept(targetThread, client); } });

 

###问题1: 为什么这里使用了两个多线程，为什么不用一个就够了呢？
我的分析是：
1、如果用一个线程池，那么不断有连接进来，就会不断生成一个线程处理，
每个线程都要处理完这些数据才会释放，那么并发高度时候，就会有大量的处理线程积压，
这个线程池就会不断膨胀，最终崩溃
2、如果每个线程内部采用一个队列，那就要分别启动这些线程，并加入socketchannel到这个队列里面

thirft调度策略有：FAIR_ACCEPT和FAST_ACCEPT，默认是后者，
就是说FAST_ACCEPT模式其实就是答案里说的，只是用了一个线程池模式，没有用ExcuteServcie

 

###问题2: SelectorThread内部为什么用blocking queue，而不用concurrent queue呢?
因为这个queue不需要共享，blocking queue效率更快
