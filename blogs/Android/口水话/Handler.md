---
Handler 消息机制口水话
---

#### 大纲

1. 基本使用
2. 源码分析
3. 常见问题
4. 实际应用
5. 其他

handler是Android Framework 架构中的一个 基础组件，用于 子线程与主线程间的通讯，实现了一种 非堵塞的消息传递机制。

Handler 的基本使用我就不多说了，不过需要注意的是内存泄露，最好使用静态内部类 + 弱引用，并且在 Activity 退出时移除消息。

Handler 消息机制比较简单：

首先通过 Handler 的 sendMessage 或 post 两种方式去发送消息，post 内部也是 sendMessage 形式，只不过把传入的 Runnable 参数包装成 Message 的 callback。然后把 Message 传给 MessageQueue 的 enqueueMessage 入队，其实就是维护一个 Message 链表，它会根据 Message 的 when 时间排序，延迟消息不过是延时时间再加上当前时间。

然后就是核心逻辑了，在 Looper 的 loop 方法，它是一个死循环，里面做了三件事，调用 MessageQueue 的 next 方法取消息，然后通过 Message 的 target 也就是 handler 去 dispatchMessage 分发消息，最后回收消息。MessageQueue 的 next 也是一个死循环，首先会调用 nativePollOnce 函数，如果没有消息或者处理时间未到，就会阻塞在 Native 层的 pollOnce 函数，睡眠时间就是从 Java 层传过来的。不过其核心实现是在 pollInner 中，如果监听的文件描述符没有发生 IO 读写事件，那么当前线程就会在 epoll_wait 中进入休眠。如果当前线程有新的消息需要处理，就会被唤醒，然后沿着之前的调用路径返回到 Java 层，最后就是对新消息进行处理。

平时代码中为了提高运行效率，有两点可以参考，第一是获取 Message 最好通过 Message 的 obtain 获取，不要直接 new，因为 Message.obtain 会从缓存里面去取。第二是可以使用 IdleHandler 在消息队列空闲时提前做一些操作。（这个我在项目中也有用到，在点击消息中心图标时，会跳到 h5 页面，并且在 url 后面加密拼接 uid 和 phone number，这一加密操作是同步的，所以就可以通过 IdleHandler 提前做这一操作，并且返回 false，表示只做一次。）

下面在简单说下设置同步屏障，即 MessageQueue.postSyncBarrier()。同步屏障的原理其实很简单，本质上是通过创建一个 target 成员为 null 的 Message 并插入到消息队列中，遇到同步屏障，MessageQueue 会循环遍历找出一条异步消息，然后处理。在同步屏障没移除前，只会处理异步消息，处理完所有的异步消息后，就会处于堵塞。如果想恢复处理同步消息，需要调用 removeSyncBarrier() 移除同步屏障。最经典的实现就 ViewRootImpl 调用 scheduleTraversals 方法进行视图更新时使用，它通过postSyncBarrier添加屏障， Choreographer 通过postCallback方法 把 mTraversalRunnable 设置成异步消息优先执行，在真正执行完scheduleTraversals操作，即mTraversalScheduled之后才会移除这个同步屏障。
