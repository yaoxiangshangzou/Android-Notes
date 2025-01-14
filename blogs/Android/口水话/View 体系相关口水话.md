---
View 体系相关口水话
---

#### 目录

1. View 绘制流程
2. View 事件分发
3. View 刷新机制
4. 项目总结

#### View 绘制流程

##### addView流程  
  
View 的显示是以 Activity 为载体的，Activity 是在 ActivityThread 的 performLaunchActivity 中进行创建的，在创建完成之后就会调用其 attach 方法，attach 方法所做的就是 new 一个 PhoneWindow 并关联 WindowManager。接下来就是 onCreate 方法，这一步就是把我们的布局文件解析成 View 塞到 DecorView 里的 mContentView 中。然后在 handleResumeActivity 中，通过 WindowManagerImpl 的 addView 方法把 DecorView 添加进去，实际实现是 WindowManagerGlobal 的 addView 方法，它里面管理着所有的 DecorView 及其对应的 ViewRootImpl，ViewRootImpl 是 DecorView 的管理者，它负责 View 树的测量、布局、绘制，以及通过 Choreographer 来控制 View 的刷新。  
![WPS图片-抠图](https://github.com/yaoxiangshangzou/Android-Notes/assets/20512625/58454c85-78b2-4a31-906c-523a2a74c81d)

（接下来就是绘制 View 了）

##### 绘制View流程  

从 performTraversals 开始，Measure、Layout、Draw 三步走。

    Measure：测量视图宽高。 单一View:measure() -> onMeasure() -> getDefaultSize() 计算View的宽/高值 ->
    setMeasuredDimension存储测量后的View宽 / 高 ViewGroup: -> measure() -> 需要重写onMeasure( ViewGroup
    没有定义测量的具体过程，因为ViewGroup是一个抽象类，其测量过程的onMeasure方法需要各个子类去实现。
    如：LinearLayout、RelativeLayout、FrameLayout等等，这些控件的特性都是不一样的，测量规则自然也都不一
    样。)遍历测量ViewGroup中所有的View -> 根据父容器的MeasureSpec和子View的LayoutParams等信息计算子
    View的MeasureSpec -> 合并所有子View计算出ViewGroup的尺寸 -> setMeasuredDimension 存储测量后的宽 / 高
    从顶层父View向子View的递归调用view.layout方法的过程，即父View根据上一步measure子View所得到的布局大小
    和布局参数，将子View放在合适的位置上。
    Layout：先通过 measure 测量出 ViewGroup 宽高，ViewGroup 再通过 layout 方法根据自身宽高来确定自身
    位置。当 ViewGroup 的位置被确定后，就开始在 onLayout 方法中调用子元素的 layout 方法确定子元素的位置。子
    元素如果是 ViewGroup 的子类，又开始执行 onLayout，如此循环往复，直到所有子元素的位置都被确定，整个
    View 树的 layout 过程就执行完了。
    Draw：绘制视图。ViewRoot创建一个Canvas对象，然后调用OnDraw()。六个步骤：①、绘制视图的背景；
    ②、保存画布的图层（Layer）；③、绘制View的内容；④、绘制View子视图，如果没有就不用；⑤、还原图层
    （Layer）；⑥、绘制View的装饰(例如滚动条等等)。

完成这三步之后，会在 ActivityThread 的 handleResumeActivity 最后调用 Activity 的 makeVisible，这个方法就是将 DecorView 设置为可见状态。

##### MeasureSpec是什么

    MeasureSpec表示的是一个32位的整形值，它的高2位表示测量模式SpecMode，低30位表示某种测量模式下的
    规格大小SpecSize。MeasureSpec是View类的一个静态内部类，用来说明应该如何测量这个View。它由三种测量模
    式，如下：
    EXACTLY：精确测量模式，视图宽高指定为match_parent或具体数值时生效，表示父视图已经决定了子视图的
    精确大小，这种模式下View的测量值就是SpecSize的值。
    AT_MOST：最大值测量模式，当视图的宽高指定为wrap_content时生效，此时子视图的尺寸可以是不超过父视
    图允许的最大尺寸的任何尺寸。
    UNSPECIFIED：不指定测量模式, 父视图没有限制子视图的大小，子视图可以是想要的任何尺寸，通常用于系统
    内部，应用开发中很少用到。
    MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，为了方便操作，其提
    供了打包和解包的方法，打包方法为makeMeasureSpec，解包方法为getMode和getSize。

WMS 是所有 Window 窗口的管理者，它负责 Window 的添加和删除、Surface 的管理和事件分发等等，因此每一个 Activity 中的 PhoneWindow 对象如果需要显示等操作，就需要要与 WMS 交互才能进行。这一步是在 ViewRootImpl 的 setView 方法中，会调用 requestLayout，并且通过 WindowSession 的 addToDisplay 与 WMS 进行交互，WMS 会为每一个 Window 关联一个 WindowState。除此之外，ViewRootImpl 的 setView 还做了一件重要的事就是注册 InputEventReceiver，这和 View 事件分发有关。

在 WindowState 中，会创建 SurfaceSession，它会在 Native 层构造一个 SurfaceComposerClient 对象，它是应用程序与 SurfaceFlinger 沟通的桥梁。至此，ViewRootImpl 与 WMS、SurfaceFlinger 都已经建立起连接，但是此时 View 还没显示出来，我们知道，所有的 UI 最终都要通过 Surface 来显示，那么 Surface 是什么时候创建的呢？

这就要回到前面所说的 ViewRootImpl 的 requestLayout 方法了，这个方法首先会 checkThread 检查是否是主线程，然后调用 scheduleTraversals 方法，这个方法首先会设置同步屏障，然后通过 Choreographer 在下一帧到来时去执行 doTraversal 方法。在 doTraversal 中调用 performTraversal 中真正进行 View 的绘制流程，即调用 performMeasure、performLayout、performDraw。不过在它们之前，会先调用 relayoutWindow 通过 WindowSession 与 WMS 进行交互，即把 Java 层创建的 Surface 与 Native 层的 Surface 关联起来。



#### View 事件分发

事件分发最开始是传递给 DecorView 的，DecorView 的 dispatchTouchEvent 是传给 Window Callback 接口方法 dispatchTouchEvent，而 Activity 实现了 Window Callback 接口，在 Activity 的 dispatchTouchEvent 方法里，是调到 Window 的 dispatchTouchEvent，Window 的唯一实现类 PhoneWindow 又会把这个事件回传给 DecorView，DecorView 在它的 superDispatchTouchEvent 把事件转交给了 ViewGroup。

所以，事件分发的流程是：

```xml
DecorView -> Activity -> PhoneWindow -> DecorView -> ViewGroup -> View
```

这里面涉及了三个角色，Activity、ViewGroup 和 View。

    1、Activity：只有分发dispatchTouchEvent和消费onTouchEvent两个方法。Activity 的 dispatchTouchEvent 前面说过，
    它的 dispatchTouchEvent 一般都是返回 false 不消费往下传；
    2 View 只有分发和消费两个方法。view的 dispatchTouchEvent，如果注册了 OnTouchListener 就调用其 onTouch 方法，
    如果 onTouch 返回 false 还会接着调用 onTouchEvent 函数，onTouchEvent 作为一种兜底方案，它在内部会根据 MotionEvent 
    的不同类型做相应处理，比如是 ACTION_UP 就需要执行 performClick 函数。
    3 ViewGroup：拥有分发、拦截和消费三个方法。ViewGroup 因为涉及对子 View 的处理，其派发流程没有 View 那么简单直接，
    它重写了 dispatchTouchEvent 方法，如果 ViewGroup 允许拦截，就调用其 onInterceptTouchEvent 来判断是否要真正执行拦
    截了，如果拦截了就交由自己的 onTouchEvent 处理，如果不拦截，就从后遍历子 View 处理，它有两个函数可以过滤子 View，
    一个是判断这个子 View 是否接受 Pointer Events 事件，另一个是判断落点有没有落在子 View 范围内。如果都满足，则调用
    其 dispatchTouchEvent 处理。如果该子 View 是一个 ViewGroup 就继续调用其 dispatchTouchEvent，否则就是 View 的 
    dispatchTouchEvent 方法，如此循环往复，直到事件真正被处理。

伪代码表示为：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean consume = false;
    if (onInterceptTouchEvent(event)) {
        consume = onTouchEvent(event);
    } else {
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

最后可以画一下这个图：

![](https://i.loli.net/2020/07/22/qgVSpUYRJP7ycrO.jpg)

在最新的 Android 系统中，事件的处理者不再由 InputEventReceiver 独自承担，而是通过多种形式的 InputStage 来分别处理，它们都有一个回调接口 onProcess 函数，这些都声明在 ViewRootImpl 内部类里面，并且在 setView 里面进行注册，比如有 ViewPreImeInputStage 用于分发 KeyEvent，这里我们重点关注与 MotionEvent 事件分发相关的 ViewPostImeInputStage。在它的 onProcess 函数中，如果判断事件类型是 SOURCE_CLASS_POINTER，即触摸屏的 MotionEvent 事件，就会调用 mView 的 dispatchPointerEvent 方法处理。

到这里基本上事件分发就讲完了，但是还可以扩展一下，事件到底是从哪里来的？

这就要涉及 InputManagerService 相关的知识了。IMS 的创建过程和 WMS 类似，都是由 SystemServer 统一启动，在创建时 IMS 会把自己的实例传给 WMS，也就是表明 WMS 是 InputEvent 的派发者，这样的设计是自然而然的，因为 WMS 记录了当前系统中所有窗口的完整状态信息，所以也只有它才能判断应该把事件投递给哪一个具体的应用程序处理。

IMS 在 Native 层创建了两个线程，InputReaderThread 和 InputDispatcherThread，前者负责从驱动节点中读取 Event，后者专责与分发事件。InputReaderThread 中的实现核心是 InputReader 类，但在 InputReader 实际上并不直接去访问设备节点，而是通过 EventHub 来完成这一工作。EventHub 通过读取 /dev/input 下的相关文件来判断是否有新事件，并通知 InputReader。InputDispatcherThread 在创建之初，就把自己的实例传给了 InputReaderThread，这样便可以源源不断的获取事件进行分发了。

分发时是如何找到投递目标呢？也就是 findFocusedWindowTargetsLocked 方法的实现。也就是通过 InputMonitor 来找到 “最前端” 的窗口即可。这个 InputMonitor 是 WMS 提供的，而 IMS 实现了 WindowManagerCallbacks 接口，并把 InputMonitor 作为参数传递进去。在获知 InputTarget 之后，InputDispatcher 就需要和窗口建立连接，是通过 InputChannel，这也是一个跨进程通信，但是并不是采用 Binder，而是 Unix Domain Socket 实现。

在 Java 层，InputEventReceiver 对 InputChannel 进行包装，它是一个抽象类，它唯一的实现 WindowInputEventReceiver 就是在 ViewRootImpl 中，这样 ViewRootImpl 就可以获取到事件了。在 ViewRootImpl 中，一旦获知 InputEvent，就会进行入队操作，如果是紧急事件就直接调用 doProcessInputEvent 处理，如果不是紧急事件，就会把这个 InputEvent 推送到消息队列，然后按顺序处理，此时需要注意 Message 为异步消息。

至此，事件就流向了 ViewRootImpl 了。

#### View 刷新机制

当我们调用 View 的 invalidate 刷新视图时，它会调到 ViewRootImp 的 invalidateChildInParent，这个方法首先会 checkThread 检查是否是主线程，然后调用其 scheduleTraversals 方法。这个方法就是视图绘制的开始，但是它并不是立即去执行 View 的三大流程，而是先往消息队列里面添加一个同步屏障，然后在往 Choreographer 里面注册一个 TRAVERSAL 的回调。在下一次 Vsync 信号到来时，会去执行 doTraversals 方法。

Choreographer 主要是用来接收 Vsync 信号，并且在信号到来时去处理一些回调事件。事件类型有四种，分别是 Input、Animation、Traversal、Commit。在 Vsync 信号到来时，会依次处理这些事件，前三种比较好理解，第四种 Commit 是用来执行组件的 onTrimMemory 函数的。Choreographer 是通过 FrameDisplayEventReceiver 来监听底层发出的 Vsync 信号的，然后在它的回调函数 onVsync 中去处理，首先会计算掉帧，然后就是 doCallbacks 处理上面所说的回调事件。

Vsync 信号可以理解为底层硬件的一个消息脉冲，它每 16ms 发出一次，它有两种方式发出，一种是 HWComposer 硬件产生，一种是用软件模拟，即 VsyncThread。不管使用哪种方式，都统一由 DispSyncThread 进行分发。

#### 项目总结

##### 自定义 View 

待定。

##### 事件分发

熟悉事件分发是处理嵌套滑动的基础。

在项目中，我们遇到了一个 ViewPager2 嵌套 RecyclerView 滑动过于灵敏的问题，即稍微滑动一点 RecyclerView 就会导致 ViewPager2 切换，这在使用 ViewPager 是没啥问题，但是 ViewPager2 内部使用的也是 RecyclerView + SnapHelper 做横向滑动，导致了滑动灵敏问题。又因为 ViewPager2 是 final 的，所以只能使用内部拦截法。

解决办法是：重写 ViewPager2 的根布局 FrameLayout 的 dispatchTouchEvent，判断如果 dx>dy+30，即表明是横向滑动，requestDisallowInterceptTouchEvent(false) 允许 ViewPager2 去拦截处理。详细代码如下：

```java
    int dx = 0;
    int dy = 0;

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {

        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                dx = (int) ev.getRawX();
                dy = (int) ev.getRawY();
                getParent().requestDisallowInterceptTouchEvent(true);
                break;
            case MotionEvent.ACTION_MOVE:
                if (Math.abs(dx - ev.getRawX()) > Math.abs(dy - ev.getRawY()) + 30) {
                    getParent().requestDisallowInterceptTouchEvent(false);
                } else {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }
                break;
            case MotionEvent.ACTION_UP:
                dx = 0;
                dy = 0;
                getParent().requestDisallowInterceptTouchEvent(false);
                break;
            default:
                break;
        }
        return super.dispatchTouchEvent(ev);
    }
```

