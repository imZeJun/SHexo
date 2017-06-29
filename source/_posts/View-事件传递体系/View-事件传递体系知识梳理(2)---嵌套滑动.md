---
title: View 事件传递体系知识梳理(2) - 嵌套滑动
date: 2017-02-20 20:52
categories : View 事件传递体系知识梳理
---
# 一、引言
- 嵌套滑动处理的难点在于：当子控件消费了事件，那么父控件就不会再有机会处理事件了。
- 嵌套滑动的基本原理是在子控件接收到滑动一段距离的请求时，先**询问父控件是否要滑动**，如果滑动了父控件就通知子控件它消耗了一部分滑动距离，子控件就只**处理剩下的滑动距离**，然后子控件滑动完毕后再**把剩余的滑动距离传给父控件**。
- 这样父控件和子控件就有机会对滑动操作作出响应，尤其父控件能够分别在**子控件处理滑动距离之前和之后**对滑动距离进行响应。

# 二、兼容性问题
- 在`SDK21`之后，嵌套滑动相关的逻辑被写入了`View`和`ViewGroup`类。
- 在`android.support.v4`中提供了接口`NestedScrollingChild`和`NestedScrollingParent`，他们分别定义了`View`和`ViewParent`中新增的方法，还有两个相关辅助类`NestedScrollingChildHelper`和`NestedScrollingParentHelper`。
- 如果版本是`SDK21`之前，那么就会判断控件是否实现了接口，然后调用接口的方法，如果是`SDK21`之后，那么就可以直接调用对应的方法。

# 三、默认处理逻辑
虽然`View`和`ViewGroup`本身就具有嵌套滑动的相关方法，但是默认情况是不会调用，因为`View`和`ViewGroup`本身不支持滑动，即**本身不支持滑动的控件即使有嵌套滑动的相关方法也不能进行嵌套滑动**。
因此，要让控件支持嵌套滑动，那么要满足：
- 控件类具有嵌套滑动的相关方法，要么仅支持`21`之后的版本，要么实现对应的接口。
- 控件要在合适的位置主动调起嵌套滑动方法。

# 四、相关方法

## 4.1 `NestedScrollingChild`
- `startNestedScroll`：起始方法，主要作用是找到接收滑动距离信息的外控件。
- `dispatchNestedPreScroll`：在内控件处理滑动前把滑动信息分发给外控件。
- `dispatchNestedScroll`：在内控件处理完滑动后把剩下的距离信息分发给外控件。
- `stopNestedScroll`：结束方法，主要作用是清空嵌套滑动的相关状态。
- `setNestedScrollingEnabled`和`isNestedScrollingEnabled`：用来判断控件是否支持嵌套滑动。
- `dispatchNestedPreFling`和`dispatchNestedFling`：和`Scroll`的对应方法类似，但是分发的是`Fling`信息。

## 4.2 `NestedScrollingParent`
因为内控件是发起者，所以外控件的大部分方法都是被内控件的对应方法所回调的。
- `onStartNestedScroll`：对应`startNestedScroll`，内控件通过调用外控件的这个方法来确定外控件是否接收滑动信息。
- `onNestedScrollAccepted`：当外控件确定接收滑动信息后该方法被回调，可以让外控件做一些前期工作。
- `onNestedPreScroll`：关键方法，接收内控件处理滑动前的距离信息，在这里外控件可以优先响应滑动操作，消耗部分或者全部滑动距离。
- `onNestedScroll`：关键方法，接收内控件处理完滑动后的距离信息，在这里外控件可以选择是否处理剩余的滑动信息。
- `onStopNestedScroll`：对应`stopNestedScroll`，用来做一些收尾工作。
- `getNestedScrollAxes`：返回嵌套滑动的方向。
- `onNestedPreFling`和`onNestedFling`：同上。

# 五、`NestedScrollView`

## 5.1 收到`down`事件，寻找外控件
`NestedScrollView`实际上是一个`FrameLayout`，同时它实现了`NestedScrollingParent、NestedScrollingChild、ScrollingView`这三个接口，它既可以用来作为外控件，也可以用来作为内控件。

我们先从入口函数`startNestedScroll`方法看起，它在`NestedScrollView`中调用的地方有以下三处：
- `public boolean onInterceptTouchEvent(MotionEvent ev)`
- `public boolean onTouchEvent(MotionEvent ev)`
- `public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes)`

而在`startNestedScroll`又会调用`mChildHelper/View`的`startNestedScroll`方法，下面我们来看一下它的实现，它遍历它所有的祖先节点，并调用每个节点的`onStartNestedScroll(child, this,axes)`方法，如果该方法返回了`true`，那么就将他作为嵌套滑动的外控件记录下来，之后所有和外控件的交互都是通过`mNestedScrollingParent`来实现的，接下来调用它的`onNestedScrollAccepted(child, this, axes)`方法，并停止遍历，返回`true`。如果它所有的祖先结点都不满足嵌套滑动的条件，那么最终返回`false`。
```
    public boolean startNestedScroll(int axes) {
        if (hasNestedScrollingParent()) {
            // Already in progress
            return true;
        }
        if (isNestedScrollingEnabled()) {
            ViewParent p = getParent();
            View child = this;
            while (p != null) {
                try {
                    if (p.onStartNestedScroll(child, this, axes)) {
                        mNestedScrollingParent = p;
                        p.onNestedScrollAccepted(child, this, axes);
                        return true;
                    }
                } catch (AbstractMethodError e) {
                    Log.e(VIEW_LOG_TAG, "ViewParent " + p + " does not implement interface " +
                            "method onStartNestedScroll", e);
                    // Allow the search upward to continue
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```
接下来，我们看一下`mParentHelper/ViewGroup`的`public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes)`，它在`ViewGroup`默认值是返回`false`：
```
    @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        return false;
    }
```
而在`NestedScrollView`中的条件是：
```
    @Override
    public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
        return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
    }
```
在接着调用的`onNestedScrollAccepted`中，`ViewGroup`记录下`axes`的值：
```
    @Override
    public void onNestedScrollAccepted(View child, View target, int axes) {
        mNestedScrollAxes = axes;
    }
```
而`NestedScrollView`则会继续调用`startNestedScroll`来寻找它的外控件：
```
    @Override
    public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
        mParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
    }
```

总结：第一个阶段主要是为了寻找到嵌套滑动的外控件，并确定滑动的方向。

## 5.2 收到`move`事件，交给外控件处理一部分的滑动距离 
之后的滑动就需要通过`public boolean onTouchEvent(MotionEvent ev)`中的`ACTION_MOVE`来处理了，我们来看一下`NestedScrollView`的处理逻辑：
```
            case MotionEvent.ACTION_MOVE:
                final int activePointerIndex = MotionEventCompat.findPointerIndex(ev,
                        mActivePointerId);
                if (activePointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                    break;
                }
                //1.获得当前的y坐标
                final int y = (int) MotionEventCompat.getY(ev, activePointerIndex);
                //2.记录该次滑动的距离
                int deltaY = mLastMotionY - y;
                //3.如果有外控件,那么交给它先处理滑动事件,这里传入了3个参数:
                if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
                    deltaY -= mScrollConsumed[1];
                    vtev.offsetLocation(0, mScrollOffset[1]);
                    mNestedYOffset += mScrollOffset[1];
                }
                if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                    mIsBeingDragged = true;
                    if (deltaY > 0) {
                        deltaY -= mTouchSlop;
                    } else {
                        deltaY += mTouchSlop;
                    }
                }
                //.....
```
在`View`的`dispatchNestedPreScroll`，它通过先前保存下来的外控件变量，把当前滑动的距离传给它来处理，在`ViewGroup`中这个函数什么事情也没有做，如果我们要实现自己的嵌套滑动逻辑，那么就要在这里面进行处理：
```
    public boolean dispatchNestedPreScroll(int dx, int dy,
            @Nullable @Size(2) int[] consumed, @Nullable @Size(2) int[] offsetInWindow) {
        if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
            if (dx != 0 || dy != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                if (consumed == null) {
                    if (mTempNestedScrollConsumed == null) {
                        mTempNestedScrollConsumed = new int[2];
                    }
                    consumed = mTempNestedScrollConsumed;
                }
                consumed[0] = 0;
                consumed[1] = 0;
                //调用父控件的接口,询问它是否要消耗滑动事件.
                mNestedScrollingParent.onNestedPreScroll(this, dx, dy, consumed);

                if (offsetInWindow != null) {
                    getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return consumed[0] != 0 || consumed[1] != 0;
            } else if (offsetInWindow != null) {
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```
这个阶段的过程，可以理解为：
- 得到当前`y`坐标的值
- 根据上次`y`坐标的值计算出这次滑动的距离`deltaY`
- 把这个`deltaY`值交给外控件处理
- 外控件返回两个数组，`mScrollConsumed`表示该阶段外控件消耗的距离，`mScrollOffset`表示本次交给外控件之后，内控件窗口变动的坐标值，如果消耗的`x`或`y`值不为0，那么该函数返回`true`。
- `deltaY - mScrollConsumed[1]`得到内控件接下来要处理的距离。

## 5.3 外控件处理完滑动距离后，交给内控件滚动
```
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    mLastMotionY = y - mScrollOffset[1];

                    final int oldY = getScrollY();
                    final int range = getScrollRange();
                    final int overscrollMode = ViewCompat.getOverScrollMode(this);
                    boolean canOverscroll = overscrollMode == ViewCompat.OVER_SCROLL_ALWAYS ||
                            (overscrollMode == ViewCompat.OVER_SCROLL_IF_CONTENT_SCROLLS &&
                                    range > 0);

                    // Calling overScrollByCompat will call onOverScrolled, which
                    // calls onScrollChanged if applicable.
                    if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                            0, true) && !hasNestedScrollingParent()) {
                        // Break our velocity if we hit a scroll barrier.
                        mVelocityTracker.clear();
                    }
                    //.....
                 }
```
## 5.4 内控件滚动完毕后，交给外控件继续处理
```
                    final int scrolledDeltaY = getScrollY() - oldY;
                    final int unconsumedY = deltaY - scrolledDeltaY;
                    if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
                        mLastMotionY -= mScrollOffset[1];
                        vtev.offsetLocation(0, mScrollOffset[1]);
                        mNestedYOffset += mScrollOffset[1];
                    } else if (canOverscroll) {
                        //..
                    }
```
这里调用了`mChildHelper/View`的`dispatchNestedScroll`方法，它里面会通过`mNestedScrollingParent`来通知外控件来处理剩余的距离，在`ViewGroup`的`onNestedScroll`方法中，什么也没有做：
```
    public boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, @Nullable @Size(2) int[] offsetInWindow) {
        if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
            if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                mNestedScrollingParent.onNestedScroll(this, dxConsumed, dyConsumed,
                        dxUnconsumed, dyUnconsumed);

                if (offsetInWindow != null) {
                    getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return true;
            } else if (offsetInWindow != null) {
                // No motion, no dispatch. Keep offsetInWindow up to date.
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

## 5.5 收到`up`事件，停止嵌套滑动

通过调用`stopNestedScroll`方法来停止滑动：
- `public boolean onInterceptTouchEvent(MotionEvent ev)`的`ACTION_UP`
- `public boolean onTouchEvent(MotionEvent ev)`的`ACTION_UP`和`ACTION_CANCEL`

在`View`的`stopNestedScroll`方法中，调用外控件的`onStopNestedScroll`方法来通知它整个滑动结束：
```
    public void stopNestedScroll() {
        if (mNestedScrollingParent != null) {
            mNestedScrollingParent.onStopNestedScroll(this);
            mNestedScrollingParent = null;
        }
    }
```
# 六、运用`NestedScrollView`
下面，我们再通过一个简单的例子，来看一下使用`NestedScrollView`的效果，布局文件：
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    
    <!-- 标题部分 -->
    <android.support.design.widget.AppBarLayout
        android:id="@+id/appbar"
        android:layout_height="wrap_content"
        android:layout_width="match_parent">
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            app:layout_scrollFlags="scroll|enterAlways"
            android:background="@android:color/holo_blue_dark"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize">
        </android.support.v7.widget.Toolbar>
    </android.support.design.widget.AppBarLayout>

    <!-- 内容部分 -->
    <android.support.v4.widget.NestedScrollView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">
        <LinearLayout
            android:orientation="vertical"
            android:layout_width="match_parent"
            android:layout_height="wrap_content">
            <TextView
                android:text="1"
                android:layout_width="match_parent"
                android:layout_height="200dp"/>
            <TextView
                android:text="2"
                android:layout_width="match_parent"
                android:layout_height="200dp"/>
            <TextView
                android:text="3"
                android:layout_width="match_parent"
                android:layout_height="200dp"/>
            <TextView
                android:text="4"
                android:layout_width="match_parent"
                android:layout_height="200dp"/>
        </LinearLayout>
    </android.support.v4.widget.NestedScrollView>
</android.support.design.widget.CoordinatorLayout>
```
我们通过`CoordinatorLayout`把标题部分和内容部分包裹起来，这样再滑动下面的`NestedScrollView`时，可以实现标题栏的隐藏和显示。
