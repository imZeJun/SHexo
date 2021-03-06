---
title: 性能优化工具知识梳理(6) - Memory Monitor & Heap Viewer & Allocation Tracker
date: 2017-03-28 16:38
categories : 性能优化工具知识梳理
---
> [性能优化工具知识梳理(1) - TraceView](http://www.jianshu.com/p/37c263f9886b)
[性能优化工具知识梳理(2) - Systrace](http://www.jianshu.com/p/41bb27235921)
[性能优化工具知识梳理(3) - 调试GPU过度绘制 & GPU呈现模式分析](http://www.jianshu.com/p/ac2d58666106)
[性能优化工具知识梳理(4) - Hierarchy Viewer](http://www.jianshu.com/p/7ac6a2b8d740)
[性能优化工具知识梳理(5) - MAT](http://www.jianshu.com/p/fa016c32360f)
[性能优化工具知识梳理(6) - Memory Monitor & Heap Viewer & Allocation Tracker](http://www.jianshu.com/p/29a539bca730)
[性能优化工具知识梳理(7) - LeakCanary](http://www.jianshu.com/p/3c055862f353)
[性能优化工具知识梳理(8) - Lint](http://www.jianshu.com/p/4ebe5d502842)

# 一、概述
`Memory Profilers`是分析内存工具的集合，它包括以下三部分：
- `Memory Monitor Tool`
- `Heap Viewer`
- `Allocation Tracker`

# 二、`Memory Monitor`
`Memory Monitor`是`Android Studio`中自带的内存检测工具，它的作用有：
- 实时检测应用的内存占用情况。
- 检测卡顿是否是由于正在`Gc`引起。
- 定位崩溃问题是否由内存问题引起。

这个工具位于`Android Studio/Monitor`一栏当中，前面我们在介绍`MAT`的时候曾经使用过它，首先编写一个简单的`demo`，通过它可以分配和回收内存：
```
public class TrackerObject {

    List<Bitmap> mBitmaps = new ArrayList<>();

    public void allocBitmaps() {
        for (int i = 0; i < 100; i++) {
            Bitmap bitmap = Bitmap.createBitmap(200, 200, Bitmap.Config.ARGB_8888);
            mBitmaps.add(bitmap);
        }
    }

    public void releaseBitmaps() {
        for (Bitmap bitmap : mBitmaps) {
            bitmap.recycle();
        }
        mBitmaps.clear();
    }
}

public class TrackerActivity extends Activity {

    private TrackerObject mTrackerObject;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_tracker);
        mTrackerObject = new TrackerObject();
    }

    public void alloc(View view) {
        mTrackerObject.allocBitmaps();
    }

    public void release(View view) {
        mTrackerObject.releaseBitmaps();
    }
}
```
![](http://upload-images.jianshu.io/upload_images/1949836-1d1b01ffd9608ab1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 当我们点击`alloc`之后，内存不断上涨。
- 而当我们点击`release`之后，内存并不会立刻下降，而是需要点击左边的“垃圾车”按钮来主动触发垃圾回收，这时候可以看到曲线立刻下降，说明此时触发了垃圾回收过程。
- 视图中分为两个部分：
 - 深蓝色：`App`当前使用的内存。
 - 淡蓝色：已经分配给`App`，但是当前没有使用的内存。
- 当我们不断点击`alloc`，最后就会抛出`OOM`异常错误：
![](http://upload-images.jianshu.io/upload_images/1949836-437f343a684ddb1b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 三、`Heap Viewer`
`Heap Viewer`有点像是`MAT`的简化版，它是`Android Device Monitor`中的一个工具：
![](http://upload-images.jianshu.io/upload_images/1949836-c6ebc86b08465dab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
它的使用方式很简单，按照上图的步骤进行操作就可以了，需要特别注意的是，如果我们希望获得最新的内存占用情况时，那么需要做两件事：
- 保证`2`中的开关是打开的
- 点击`5`来触发一次`Gc`，这样才能得到最新的内存使用情况。

# 四、`Allocation Tracker`
`Allocation Tracker`是用来记录一段时间内的内存分配情况，并且它可以列出分配对象的大小，以及是由哪个函数分配的。
下面，我们先看一下如何使用：
![](http://upload-images.jianshu.io/upload_images/1949836-909f1c4c39f1da6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其所处位置和上面的`Heap Viewer`类似，其展现结果在`Heap`的右边，当我们需要获得一段时间的内存分配，那么需要以下几步：
- 点击`start Tracking`
- 操作`App`，这里我们点击`alloc`按钮分配一些`Bitmap`
- 点击`Get Locations`，获得从开始到结束的内存分配情况

各列值的含义：
- `Alloc Order`：分配的顺序
- `Allocation Size`：分配的大小
- `Allocated Class`：分配对象的类名
- `Thread id`：分配的线程`id`
- `Allocated in`：分配到哪个对象当中。

在整个区域的最下方，则是分配该对象的函数调用堆栈信息，这也是这个工具最有用的地方，通过它我们就可以分析出是代码中哪一段逻辑导致了某个对象的分配。

# 五、小结
下面，我们来总结一下这三个工具各自的特点：
## `Memory Monitor`
- 显示内存占用、分配和回收情况。
- 判断`GC`是否是造成应用卡顿的原因。
- 判断是否是由于内存问题导致了`App`的崩溃。
- 呈现的结果是实时的。
- 能够有效地帮助分析内存泄露。
- 定位`Gc`发生的时间，并分析这是否是合适的时间。
- 没有列出具体的分配对象。

## `Heap Viewer`
- 在垃圾回收发生时，呈现出某一时刻的内存快照。
- 帮助我们分析有可能是哪个对象引起了内存泄露。

## `Allocation Tracker`
- 分析出一段时间内对象的分配情况，并列出是由什么逻辑导致了这个对象的分配。
- 和`Heap Viwer`一起使用，来分析大对象产生的原因。

# 六、参考文献
> [`http://android.xsoftlab.net/tools/performance/comparison.html`](http://android.xsoftlab.net/tools/performance/comparison.html)
