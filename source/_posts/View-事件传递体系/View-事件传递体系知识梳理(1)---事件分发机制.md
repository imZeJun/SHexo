---
title: View 事件传递体系知识梳理(1) - 事件分发机制
date: 2017-02-24 18:29
categories : View 事件传递体系知识梳理
---
# 一、事件分发概述
## 1.1 事件分发的关键方法
对于`ViewGroup`来说，与事件分发相关的方法包括：
```
public boolean dispatchTouchEvent(MotionEvent event)
public boolean onInterceptTouchEvent(MotionEvent event)
public boolean onTouchEvent(MotionEvent event)
```
对于`View`来说，与事件分发相关的方法包括：
```
public boolean dispatchTouchEvent(MotionEvent event)
public boolean onTouchEvent(MotionEvent event)
```

## 1.2 `Down`分发的理解
对于`Down`分发的过程，网上看了很多的例子和图，但是都没能好好理解，最后还是自己总结了一种方案，这个方案的核心就是**染色**：
- 初始的时候`View`树上每个节点都是白色的。
- 从**子节点的角度**来看，当它被父节点调用`dispatchTouchEvent`时，在返回之前都会有一个返回值，如果这个返回值为真，那么它自己就会变成红色。
- 从**父节点的角度**来看，当它给子节点调用`dispatchTouchEvent`时，它**既是在尝试给子节点进行染色，也是在尝试给自己着色**，当某个子节点的`dispatchTouchEvent`方法返回时，取该子节点的颜色对自己进行着色；如果遍历完它所有的子节点，它仍然没有变成红色，那么调用它自己的`onTouchEvent`，如果返回`true`，那么把自己染成红色。
- 对于一个`ViewGroup/View`来说，有可能染色成功只包括两种途径：子节点的`dispatchTouchEvent`返回时，取子节点的颜色对自己着色；通过自己的`onTouchEvent`方法来着色。并且，只有前一种途径不成功时才会用后一种途径。
- **只要节点变成了红色，那么就不需要再尝试对自己进行染色了**，也就是上面那条说的：利用子节点的`dispatchTouchEvent`来染色或者通过自己的`onTouch`来染色，它会直接返回。

从伪代码来看：
```

public Color color = white;

public boolean dispatchTouchEvent(MotionEvent event) {
　int childCount = getChildCount;
    for (i = childCount - 1; i >= 0; i--) {
        View child = getChildAt(i);
        boolean result = child.dispatchTouchEvent(event);
        if (result) {
             color = RED;
             return true;
        }
    }
    boolean touchRes = onTouchEvent(event);
    if (touchRes) {
        color = RED;
    }
    return touchRes;
}
```
在`Down`事件分发完后，我们可以发现这么个现象。
- 假如一个节点是红色的，那么它最多只可能有一个红色的子节点。
- 假如一个节点是红色的，那么必然会有一条唯一的路径，该路径会通过该红色节点连接到根节点。
- 假如一个节点是白色的，那么它所有的后代节点都一定是白色的。

## 1.3 `Down`分发的特点
对于一个**没有重写以上关键方法并且位于`View`树上**的`ViewGroup/View`来说，它`Down`事件的分发具有以下几个特点：
### 1.3.1 `dispatchTouchEvent`
 - `dispatchTouchEvent`是否被回调，由它的**父容器决定**的。
 - 假如是它不是叶节点，当它的`dispatchTouchEvent`被调用时，它会先**逆序依次**调用下一级子节点的`dispatchTouchEvent`方法。
 - 如果在以上遍历中间有任何一个子节点的`dispatchTouchEvent`返回了`true`，那么不会继续调用剩余未遍历子节点的`dispatchTouchEvent`，并且它自身的`onTouchEvent`不会被回调。

### 1.3.2 `onInterceptTouchEvent`
只有`ViewGroup`才有，它是否被回调取决于`dispatchTouchEvent`是否被回调。

### 1.3.3`onTouchEvent`
`onTouchEvent`是**由自己回调的**，是否被回调，必须同时满足以下两个条件：
 - `dispatchTouchEvent`被回调。
 - 满足以下两种情况之一：
  - 它是整个`View`树的叶节点
  - 它拥有子节点，但是它所有子节点的`dispatchTouchEvent`都返回`false`。

## 1.4 `Move/Up`分发
在`Move/Up`事件分发的时候，其实就是根据之前着色的结果来往下传递事件，它的传递只需要遵循下面的原则：**只会分发给红色的节点，遇到白色的节点就停止往下分发**。

## 1.5 简单举例
我们举一个简单的例子：
![](http://upload-images.jianshu.io/upload_images/1949836-5aa0f204f71f2af9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 前提条件：`ViewGroup1`返回`true`。
- 过程：`Root`调用`ViewGroup1`的`dispatchTouchEvent`，而`ViewGroup1`此时是白色，因此它继续调用它的子节点，也就是`View21`的`dispatchTouchEvent`，但是`View21`没有子节点，因此它调用自己的`onTouchEvent`，它的`dispatchTouchEvent`方法返回，而此时，`ViewGroup2`所有的子节点都遍历完了，它依然没有变成红色，因此它调用自己的`onTouchEvent`，由于该方法返回`false`，因此它也返回了，并且在返回时依然是白色。接下来`Root`取`ViewGroup2`的颜色对自己着色，着色完成之后发现自己仍然是白色，那么它就继续调用有可能使自己染色成功的方法，`ViewGroup1`的`dispatchTouchEvent`，由于它的`dispatchTouchEvent`返回`true`，因此它会把自己染成红色，由于它已经变成红色了，那么它也没有权利对子节点进行染色，因此它的`dispatchTouchEvent`返回，`Root`收到返回值时，取`ViewGroup1`的颜色对自己进行着色，结果它发现自己是红色了，那么`Root`也不会调用任何可能对自己染色的方法，而是直接返回了。
- 结果：`ROOT`和`ViewGroup1`变为红色节点。

# 二、示例
只要理解了上面的思想，其实各种情况都可以对应到，下面的例子只是为了让大家能够证明自己的想法是否正确罢了，所以就不过多解释了：
```
<?xml version="1.0" encoding="utf-8"?>
<com.example.lizejun.repoviews.LogFrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:tag="ViewGroup$Root"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.example.lizejun.repoviews.MainActivity">
    
    <com.example.lizejun.repoviews.LogFrameLayout
        android:tag="ViewGroup1"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <com.example.lizejun.repoviews.LogTextView
            android:tag="ViewGroup1$View1"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
        <com.example.lizejun.repoviews.LogTextView
            android:tag="ViewGroup1$View2"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
        
    </com.example.lizejun.repoviews.LogFrameLayout>
    
    <com.example.lizejun.repoviews.LogFrameLayout
        android:tag="ViewGroup2"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        
        <com.example.lizejun.repoviews.LogTextView
            android:tag="ViewGroup2$View"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
        
    </com.example.lizejun.repoviews.LogFrameLayout>
</com.example.lizejun.repoviews.LogFrameLayout>
```
## 2.1 不改变任何函数的返回值
![](http://upload-images.jianshu.io/upload_images/1949836-49201a2b9bb34384.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.2 `ViewGroup2`的`dispatchTouchEvent`返回`false`
![](http://upload-images.jianshu.io/upload_images/1949836-6f369a8250b6bfa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.3 `ViewGroup2`的`dispatchTouchEvent`返回`true`
![](http://upload-images.jianshu.io/upload_images/1949836-62c28937ce7123e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.4 `ViewGroup1`下的孩子节点`View2`返回了`false`
![](http://upload-images.jianshu.io/upload_images/1949836-d5564afa92f90e7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.5 `ViewGroup1`下的孩子节点`View2`返回了`true`
![](http://upload-images.jianshu.io/upload_images/1949836-523fb64c416a5d3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.6 `ViewGroup2`的`onTouchEvent`返回了`false`
![](http://upload-images.jianshu.io/upload_images/1949836-65a074a2c92bc1a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.7 `ViewGroup2`的`onTouchEvent`返回了`true`
![](http://upload-images.jianshu.io/upload_images/1949836-a9d60f989d834de4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.8 `ViewGroup1`下的孩子节点`View2`返回了`false`
![](http://upload-images.jianshu.io/upload_images/1949836-4c7bac940fceed25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.9 `ViewGroup1`下的孩子节点`View2`返回了`true`
![](http://upload-images.jianshu.io/upload_images/1949836-5ed3a745b23b19fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
