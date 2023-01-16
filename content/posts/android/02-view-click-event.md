---
title: "Android事件分发机制详解"
description: "Android事件分发机制源码分析与实例测试"
date: 2018-11-20 22:00:00
draft: false
categories: ["Android"]
tags: ["Android"]
---

本文主要记录了Android中的事件分发机制。通过对源码进行分析和实例测试，对Android事件分发机制有了更深的了解。主要为学习Android时的笔记。

<!--more-->

> 更多文章欢迎访问我的个人博客-->[幻境云图](https://www.lixueduan.com/)

## 1. 触发过程

### 1.1 点击控件

### 1.2 dispatchTouchEven

一定会执行`dispatchTouchEvent`方法，若当前类没有该方法，则向上往父类查找。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
        return true;
    }
    return onTouchEvent(event);
}
```

* 条件1 `mOnTouchListener != null`

```java
public void setOnTouchListener(OnTouchListener l) {
    mOnTouchListener = l;
}
```

给控件设置监听就会给`mOnTouchListener`赋值，则条件1成立。

* 条件2 `(mViewFlags & ENABLED_MASK) == ENABLED` 控件是否是可点击的

* 条件3 `mOnTouchListener.onTouch(this, event)` 回调`onTouch`方法，返回true 则成立

**小结：**`dispatchTouchEvent` 方法中一定会执行`onTouch`方法，如果`onTouch`方法返回true 则`dispatchTouchEvent`方法直接返回true 不会执行if外的 r`eturn onTouchEvent(event)`。

```java
       title_bar.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
			Log.v("Az","onTouch");
                return false;
            }
        });
```

在`setOnTouchListener`时，onTouch方法默认返回false,所以才会执行后面的`onTouchEvent`方法；

### 1.3 onTouchEvent 

```java
public boolean onTouchEvent(MotionEvent event) {
    final int viewFlags = mViewFlags;
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PREPRESSED) != 0;
                if ((mPrivateFlags & PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }
                    if (!mHasPerformedLongPress) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();
                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }
                    if (prepressed) {
                        mPrivateFlags |= PRESSED;
                        refreshDrawableState();
                        postDelayed(mUnsetPressedState,
                                ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        // If the post failed, unpress right now
                        mUnsetPressedState.run();
                    }
                    removeTapCallback();
                }
                break;
            case MotionEvent.ACTION_DOWN:
                if (mPendingCheckForTap == null) {
                    mPendingCheckForTap = new CheckForTap();
                }
                mPrivateFlags |= PREPRESSED;
                mHasPerformedLongPress = false;
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                break;
            case MotionEvent.ACTION_CANCEL:
                mPrivateFlags &= ~PRESSED;
                refreshDrawableState();
                removeTapCallback();
                break;
            case MotionEvent.ACTION_MOVE:
                final int x = (int) event.getX();
                final int y = (int) event.getY();
                // Be lenient about moving outside of buttons
                int slop = mTouchSlop;
                if ((x < 0 - slop) || (x >= getWidth() + slop) ||
                        (y < 0 - slop) || (y >= getHeight() + slop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();
                        // Need to switch from pressed to not pressed
                        mPrivateFlags &= ~PRESSED;
                        refreshDrawableState();
                    }
                }
                break;
        }
        return true;
    }
    return false;
}

```

`首先在第14行我们可以看出，如果该控件是可以点击的就会进入到第16行的switch判断中去，而如果当前的事件是抬起手指，则会进入到MotionEvent.ACTION_UP这个case当中。在经过种种判断之后，会执行到第38行的performClick()方法，那我们进入到这个方法里瞧一瞧：` 

若当前事件为抬手，则进入`performClick`方法

```java
public boolean performClick() {
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    if (mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        mOnClickListener.onClick(this);
        return true;
    }
    return false;
}
```

如果 `mOnClickListener != null` 则会执行`onClick`方法，

```java
public void setOnClickListener(OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    mOnClickListener = l;
}
```

所以只要给控件设置了点击监听，`setOnClickListener`就会给`mOnClickListener`赋值，上面条件就成立，然后回调onClick方法。

到这儿差不多就清楚了分发流程。

`这样View的整个事件分发的流程就让我们搞清楚了！不过别高兴的太早，现在还没结束，还有一个很重要的知识点需要说明，就是touch事件的层级传递。我们都知道如果给一个控件注册了touch事件，每次点击它的时候都会触发一系列的ACTION_DOWN，ACTION_MOVE，ACTION_UP等事件。这里需要注意，如果你在执行ACTION_DOWN的时候返回了false，后面一系列其它的action就不会再得到执行了。简单的说，就是当dispatchTouchEvent在进行事件分发的时候，只有前一个action返回true，才会触发后一个action。` 

```java
public boolean onTouchEvent(MotionEvent event) { 
    //省略...
if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    break;
                case MotionEvent.ACTION_DOWN:      
                    break;
                case MotionEvent.ACTION_CANCEL:             
                    break;
                case MotionEvent.ACTION_MOVE:
                    break;
            }
            return true;
        }
}
```

可以看出在`dispatchTouchEvent`方法中，onTouch方法返回false,然后执行`onTouchEvent`方法，在进入if判断后，不管进入那个case,最后都会return true,所以才会执行后续的action.

**`1. onTouch和onTouchEvent有什么区别，又该如何使用？` **

都是`dispatchTouchEvent`中的方法，`onTouch`优先级高，若`onTouch`返回true，就会消费掉当前事件，`onTouchEvent`就不会执行。

要执行`onTouch`也需要两个条件，1 给控件设置了触摸监听`OnTouchListener `，2该控件是可以点击的。

若控件是非enable的，则不会执行`onTouch`方法，会执行`onTouchEvent`，所以想要监听`ouTouch`事件只能重写`onTouchEvent`方法来实现。

**小结：**

**控件被点击或触摸后一定会执行dispatchTouchEvent方法（当前类没有则去父类找），如果设置了触摸监听且控件是enable的，就执行onTouch方法 。OnTouch方法返回true则消耗掉本次事件，不执行后面的方法，返回false则执行onTouchEvent方法，如果设置了点击监听且控件是enable的，就在抬手的时候执行onClick方法。**

**给一个控件注册了touch事件，每次点击它的时候都会触发一系列的ACTION_DOWN，ACTION_MOVE，ACTION_UP等事件。当dispatchTouchEvent在进行事件分发的时候，只有前一个action返回true，才会触发后一个action。** 

## 2. ViewGroup

`Android中touch事件的传递，绝对是先传递到ViewGroup，再传递到View的` 

`上边说只要你触摸了任何控件，就一定会调用该控件的dispatchTouchEvent方法。这个说法没错，只不过还不完整而已。实际情况是，当你点击了某个控件，首先会去调用该控件所在布局的dispatchTouchEvent方法，然后在布局的dispatchTouchEvent方法中找到被点击的相应控件，再去调用该控件的dispatchTouchEvent方法。` 

`ViewGroup`的`dispatchTouchEvent`方法

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    final int action = ev.getAction();
    final float xf = ev.getX();
    final float yf = ev.getY();
    final float scrolledXFloat = xf + mScrollX;
    final float scrolledYFloat = yf + mScrollY;
    final Rect frame = mTempRect;
    boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (action == MotionEvent.ACTION_DOWN) {
        if (mMotionTarget != null) {
            mMotionTarget = null;
        }
        if (disallowIntercept || !onInterceptTouchEvent(ev)) {
            ev.setAction(MotionEvent.ACTION_DOWN);
            final int scrolledXInt = (int) scrolledXFloat;
            final int scrolledYInt = (int) scrolledYFloat;
            final View[] children = mChildren;
            final int count = mChildrenCount;
            for (int i = count - 1; i >= 0; i--) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                        || child.getAnimation() != null) {
                    child.getHitRect(frame);
                    if (frame.contains(scrolledXInt, scrolledYInt)) {
                        final float xc = scrolledXFloat - child.mLeft;
                        final float yc = scrolledYFloat - child.mTop;
                        ev.setLocation(xc, yc);
                        child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;
                        if (child.dispatchTouchEvent(ev))  {
                            mMotionTarget = child;
                            return true;
                        }
                    }
                }
            }
        }
    }
    boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||
            (action == MotionEvent.ACTION_CANCEL);
    if (isUpOrCancel) {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }
    final View target = mMotionTarget;
    if (target == null) {
        ev.setLocation(xf, yf);
        if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {
            ev.setAction(MotionEvent.ACTION_CANCEL);
            mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;
        }
        return super.dispatchTouchEvent(ev);
    }
    if (!disallowIntercept && onInterceptTouchEvent(ev)) {
        final float xc = scrolledXFloat - (float) target.mLeft;
        final float yc = scrolledYFloat - (float) target.mTop;
        mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;
        ev.setAction(MotionEvent.ACTION_CANCEL);
        ev.setLocation(xc, yc);
        if (!target.dispatchTouchEvent(ev)) {
        }
        mMotionTarget = null;
        return true;
    }
    if (isUpOrCancel) {
        mMotionTarget = null;
    }
    final float xc = scrolledXFloat - (float) target.mLeft;
    final float yc = scrolledYFloat - (float) target.mTop;
    ev.setLocation(xc, yc);
    if ((target.mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {
        ev.setAction(MotionEvent.ACTION_CANCEL);
        target.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;
        mMotionTarget = null;
    }
    return target.dispatchTouchEvent(ev);
}

```

第二个if语句  `if (disallowIntercept || !onInterceptTouchEvent(ev)`

第一个条件`disallowIntercept`是否禁用掉事件拦截的功能，默认是`false`。所以是否进入if内部就由第二个条件决定了。 

`ViewGroup`中有一个`onInterceptTouchEvent`方法  是否拦截触摸事件 默认返回false 即不拦截

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    return false;
}
```

第二个条件  `!onInterceptTouchEvent(ev)`对返回值取反 即返回false不拦截触摸事件时进入if内部，返回true拦截时不进入if内部

```java
//省略。。。
if (disallowIntercept || !onInterceptTouchEvent(ev)) {
            for (int i = count - 1; i >= 0; i--) {//遍历当前ViewGroup下的所有子View
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                        || child.getAnimation() != null) {
                    if (frame.contains(scrolledXInt, scrolledYInt)) {//判断当前遍历的View是不是正在点击的View
                        if (child.dispatchTouchEvent(ev))  {//是则调用子View的dispatchTouchEvent
                            mMotionTarget = child;
                            return true;
                        }
                    }
                }
            }
        }
```

if内部对子View进行了遍历，最终调用子View的`dispatchTouchEvent`，然后控件可点击那么`dispatchTouchEvent`一定会返回true，所以后面的代码就执行不了。

即 `ViewGroup` 的`onInterceptTouchEvent`返回false,不拦截触摸事件时，最终会执行子View的`dispatchTouchEvent`。

 `ViewGroup`的`onInterceptTouchEvent`返回true,拦截触摸事件，就不会进入if内部，则会执行到后面的程序

```java
if (target == null) {
        ev.setLocation(xf, yf);
        if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {
            ev.setAction(MotionEvent.ACTION_CANCEL);
            mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;
        }
        return super.dispatchTouchEvent(ev);
    }
```

可以看到，最后会执行`super.dispatchTouchEvent(ev)`，执行父类即View的`dispatchTouchEvent`。

`View`的`dispatchTouchEvent`如下：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
        return true;
    }
    return onTouchEvent(event);
}
```

然后又和前面的一样了。执行onTouch或者onTouchEvent。。

## 3. 总结

### 传递顺序

`Activity －> PhoneWindow －> DecorView －> ViewGroup －> … －> View `

通俗语言总结一下，事件来的时候，

Activity会询问Window，Window这个事件你能不能消耗，

Window一看，你先等等，我去问问DecorView他能不能消耗，

DecorView一看，onInterceptTouchEvent返回false啊，不让我拦截啊，

**(DecorView继承自FrameLayout,FrameLayout是ViewGroup的子类，所以DecorView也是ViewGroup的子类，事件从Activity传到了ViewGroup)**

遍历一下子View吧，问问他们能不能消耗，那个谁，事件按在你的身上了，你看看你能不能消耗，

**假如子View为RelativeLayout**

RelativeLayout一看，也没有让我拦截啊，我也得遍历看看这个事件发生在那个子View上面，

**到这儿事件从ViewGroup传到View上了**

那个TextView,事件在你身上，你能不能消耗了他。TextView一看，消耗不了啊，

RelativeLayout一看TextView消耗不了啊，mFirstTouchTarget==null啊，得，我自己消耗吧，嗯！一看自己的onTouchEvent也消耗不了啊！那个DecorView事件我消耗不了，

DecorView一看自己，我也消耗不了，继续往上传，那个Window啊。事件我消耗不了啊，

Window再告诉Activity事件消耗不了啊。

Activity还得我自己来啊。调用自己的onTouchEvent，还是消耗不了，算了，不要了。

**最后Activity的onTouchEvent无论返回什么，事件分发都结束。（如果事件在边界范围外默认会返回false）**

## 参考

`https://blog.csdn.net/guolin_blog/article/details/9097463`

`https://blog.csdn.net/guolin_blog/article/details/9153747`