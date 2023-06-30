---
Dalvik JIT ART
---

#### 目录

1. 思维导图
2. Android 虚拟机发展史
3. Dalvik 和 ART 的区别
4. JIT 和 AOT 的区别
5. 参考

#### 思维导图

![](https://i.loli.net/2018/12/30/5c2847c900abb.png)


#### Dalvik 和 ART 的区别

ART 相比于 Dalvik 的优点：

1. 预编译 AOT
2. 垃圾回收方面的优化
   - 只有一次（ 而非两次 ）GC 暂停
   - 在 GC 保持暂停状态期间并行处理
   - 压缩 GC 以减少后台内存使用和碎片
     
         首先介绍下Dalvik的GC过程，主要有四个过程：  
         1. 当GC被触发时候，其会去查找所有活动的对象，这个时候整个程序与虚拟机内部的所有线程就会
         挂起，这样目的是在较少的堆栈里找到所引用的对象。  
         2. GC对符合条件的对象进行标记。  
         3. GC对标记的对象进行回收。  
         4. 恢复所有线程的执行现场继续运行。
            
         Dalvik这么做的好处是，当pause了之后，GC势必是相当快速的，但是如果出现GC频繁并且内存吃紧
         势必会导致UI卡顿、掉帧、操作不流畅等等。
         后来ART改善了这种GC方式，主要的改善点在将其非并发过程改成了部分并发，还有就是对内存的重新
         分配管理。  
         当ART GC发生时：  
         1. GC将会锁住Java堆，扫描并进行标记。
         2. 标记完毕释放掉Java堆的锁，并且挂起所有线程。
         3. GC对标记的对象进行回收。
         4. 恢复所有线程的执行继续运行。
         5. 重复2-4直到结束。  
         可以看出整个过程做到了部分并发使得时间缩短，GC效率提高两倍。

3. 开发和调试方面的优化,提高内存使用，减少碎片化
   
         在ART中，它将Java分了一块空间命名为 Large-Object-Space，这个内存空间的引入用来
         专文存放大对象，同时ART又引入了 moving collector 的技术，即将不连续的物理内存快进行对齐。对
         齐之后内存碎片化就得到了很好的解决。Large-Object-Space的引入是因为moving collector对大块内
         存的位移时间成本太高。据官方统计，ART的内存利用率提高了10倍左右，大大提高了内存的利用率。

#### AOT 和 JIT 的区别

##### JIT：

![](https://i.loli.net/2018/12/30/5c28227e9df60.png)

JIT 的优点：

1. 对热点代码进行编译优化

JIT 的缺点：

1. 每次启动应用都需要重新编译
2. 运行时比较耗电

##### AOT

![](https://i.loli.net/2018/12/30/5c283927b3b96.png)

优点：

1. 应用启动更快、运行速度更加流畅
2. 耗电问题得以优化

缺点：

1. 应用安装和系统升级之后的应用优化比较耗时
2. 优化后的文件会占用额外的存储空间



#### Android 虚拟机发展史

##### Android 诞生之初

Dalvik 担任虚拟机的角色，每次运行程序的时候，Dalvik 负责加载 dex/odex 文件并解析成机器码交由系统调用。

##### Android 2.2 JIT 登场

和其他大多数 JVM 一样，Dalvik 使用 JIT 进行即时编译，借助 Java HotSpot VM，JIT 编译器可以对热点代码进行编译优化，将 dex/odex 中的 Dalvik Code ( Smali 指令集 ) 翻译成相当精简的 Native Code 去执行，JIT 的引入使得 Dalvik 的性能提升了 3~6 倍。

##### Android 4.4 ART 和 AOT 登场

在 Android 4.4 带来了全新的虚拟机运行环境 ART 和全新的编译策略 AOT，此时 ART 和 Dalvik 是共存的。

##### Android 5.0 ART 全面取代 Dalvik

至此，Dalvik 退出历史舞台，AOT 也成为了唯一的编译方式。

##### Android 7.0 JIT 回归

形成了 AOT / JIT 混合编译模式。该混合模式综合了 AOT 和 JIT 的各种优点，使得应用在安全速度加快的同时，运行速度、存储空间和耗电量等指标都得到了优化。

#### 参考

[https://juejin.im/post/5c232907f265da61662482b4](https://juejin.im/post/5c232907f265da61662482b4)

[https://source.android.com/devices/tech/dalvik?hl=zh-cn](https://source.android.com/devices/tech/dalvik?hl=zh-cn)
