# 简单的ScrollView
用于学习View滑动控制
##  系统常量类 ViewConfiguration
ViewConfiguration中存储了大量的系统常量，比如 点击时间，长按时间，最小拖动距离等，从中获取可以获得和系统控件一样的体验

   ```
      final ViewConfiguration configuration = ViewConfiguration.get(getContext());
            // 最小滑动距离
            mTouchSlop = configuration.getScaledTouchSlop();
            //  获得允许执行fling （抛）的最小速度值
            mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
            //  获得允许执行fling （抛）的最大速度值
            mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
            //显示边缘效果时视图应过度滚动的最大距离
            mOverscrollDistance = configuration.getScaledOverscrollDistance();
            //fling距离
            mOverflingDistance = configuration.getScaledOverflingDistance();
  ```

## ScrollView 的测量
   ScrollView使用时，只能拥有一个直接子View,但是这个子View可能是个ViewGroup，所以要重写measureChild()和measureChildWithMargins()方法，重写测量子View的高度，
   否则测量不出ScrollView的真实高度。
   measureChild()和measureChildWithMargins()的测量代码都是一样的，我就只贴一个了：

   ```
     @Override
       protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
           final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

           final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                   getPaddingLeft() + getPaddingRight() + lp.leftMargin + lp.rightMargin
                           + widthUsed, lp.width);
           final int usedTotal = getPaddingTop() + getPaddingBottom() + lp.topMargin + lp.bottomMargin +
                   heightUsed;
           final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                   Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
                   MeasureSpec.UNSPECIFIED);

           child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
       }
   ```
## 滑动和多指滑动
 ScrollView 的滑动分为三个部分：
   1. 多指滑动：除了一根手指拖动外，还要处理多跟手指滑动的情况。
   2.fling 加速度：当滑动速度达到一定阈值时，手指离开也会再滑动一段距离

 ###  单指滑动  onTouchEvent(MotionEvent ev)：
 onTouchEvent事件是处理View滑动的地方，手指触摸到View上时触发。
 主要是通过对手指的动作(action)判断，对View进行滑动操作

 单指滑动使用  int action = ev.getAction() 来获取动作。

onTouchEvent 主要有一下几个动作：

MotionEvent.ACTION_DOWN: 手指按下
MotionEvent.ACTION_MOVE： 手指移动
MotionEvent.ACTION_UP: 手指抬起
MotionEvent.ACTION_CANCEL：触摸事件结束
MotionEvent.ACTION_POINTER_DOWN: 多指滑动触发，有另外一只手指按下时触发
MotionEvent.ACTION_POINTER_UP: 多指滑动触发，有另外一只手指抬起时触发
基本纵向滑动方案就是在 ：
1. MotionEvent.ACTION_DOWN 的时候记录Y 坐标
2. MotionEvent.ACTION_MOVE 获取新的Y坐标，和按下时的坐标做差，获取滑动的距离。调用scrollby（）或者scrollTo（）进行滑动
3. MotionEvent.ACTION_UP   判断滑动速度，判断是否需要执行fling。

### 多指滑动
   和单指滑动流程类似，只是获取action时，需要获取新按下的手指的action，并且
  在MotionEvent.ACTION_POINTER_DOW时，
  使用新手指的坐标进行滑动，MotionEvent.ACTION_UP 时，切换回上个手指的坐标
 多指滑动使用   final int actionMasked = ev.getActionMasked()  来获取动作。


  ACTION_DOWN:记录按下手指的坐标和id
  ```
         case MotionEvent.ACTION_DOWN:{
             mLastMotionY = (int) ev.getY();
             mActivePointerId = ev.getPointerId(0);
         }
   ```

 MotionEvent.ACTION_MOVE: 进行滑动
  ```
         case MotionEvent.ACTION_MOVE:{

            // 根据手指id，获取手指标志位
             final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
            // 根据手指标志位获取y坐标
             final int y = (int) ev.getY(activePointerIndex);
            // 计算滑动的长度
             int deltaY = mLastMotionY - y;
            // 如果滑动距离超过最小滑动距离，视为滑动
             if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                mIsBeingDragged = true;
                if (deltaY > 0) {
                    deltaY -= mTouchSlop;
                } else {
                    deltaY += mTouchSlop;
                }
                // 更改上次按下的坐标为当前滑动到的位置
                 mLastMotionY = y;
                // 调用overScrollBy（）方法进行滑动
                if (overScrollBy(0, deltaY, 0, getScrollY(), 0, getScrollRange(),
                                          0, mOverscrollDistance, true)) {
                                      // Break our velocity if we hit a scroll barrier.
                                      mVelocityTracker.clear();
                }

             }

         }
  ```
  MotionEvent.ACTION_POINTER_DOWN: 处理新手指按下

  ```
   case MotionEvent.ACTION_POINTER_DOWN: {
                       final int index = ev.getActionIndex();
                       // 更改按下坐标和手指id
                       mLastMotionY = (int) ev.getY(index);
                       mActivePointerId = ev.getPointerId(index);
                       break;
                   }
   ```
   MotionEvent.ACTION_POINTER_UP：处理多手指抬起
   ```
        case MotionEvent.ACTION_POINTER_UP:{
            onSecondaryPointerUp(ev);
            mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
            break;
        }

        //多指滑动时，抬起手指时，寻找另外一只还在屏幕上的手指
        private void onSecondaryPointerUp(MotionEvent ev) {
               final int pointerIndex = (ev.getAction() & MotionEvent.ACTION_POINTER_INDEX_MASK) >>MotionEvent.ACTION_POINTER_INDEX_SHIFT;
               final int pointerId = ev.getPointerId(pointerIndex);
               if (pointerId == mActivePointerId) {
                   final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                   mLastMotionY = (int) ev.getY(newPointerIndex);
                   mActivePointerId = ev.getPointerId(newPointerIndex);
                   if (mVelocityTracker != null) {
                       mVelocityTracker.clear();
                   }
               }
           }

   ```

需要注意的是：MotionEvent.ACTION_MOVE 中调用   overScrollBy（）方法进行滑动，该方法并没有真正调用scrollBy/scrollTo 滑动，只是进行了一些边界判断，然后调用了 onOverScrolled（）方法，所以真正的滑动需要我们重写  onOverScrolled（） 进行滑动

 ```
     @Override
        protected void onOverScrolled(int scrollX, int scrollY, boolean clampedX, boolean clampedY) {
            if (!mScroller.isFinished()) {
                final int oldX = getScrollX();
                final int oldY = getScrollY();
                setScrollX(scrollX);
                setScrollY(scrollY);
                onScrollChanged(getScrollX(), getScrollY(), oldX, oldY);
                if (clampedY) {
                    mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, getScrollRange());
                }
            } else {
            // 调用scrollTo 进行滑动
                super.scrollTo(scrollX, scrollY);
            }
        }
```


## fling滑动
fling滑动，就是我们在使用scrollView时，快速滑动时，抬起手指，还会再滑动一段距离。
所以是在手指抬起的时候，对滑动速度进行判断，如果超过阈值，就进行fling。

1. 滑动速度计算，通过 VelocityTracker 统计计算。
   ```
       // 初始化 VelocityTracker
         private void initVelocityTrackerIfNotExists() {
                if (mVelocityTracker == null) {
                    mVelocityTracker = VelocityTracker.obtain();
                }
          }

          @Override
              public boolean onTouchEvent(MotionEvent ev) {
              //touch事件开始时初始化  VelocityTracker
                initVelocityTrackerIfNotExists();
                。。。省略代码

                // 添加滑动数据统计
                if (mVelocityTracker != null) {
                          mVelocityTracker.addMovement(ev);
                }

            }
   ```
2. MotionEvent.ACTION_UP时，判断速度，实现fling
```
 case MotionEvent.ACTION_UP:{
                 if (mIsBeingDragged) {
                     final VelocityTracker velocityTracker = mVelocityTracker;
                     //设置移动速度单位为：像素/10000ms，即1000毫秒内移动的像素
                     velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                     //获取手指在界面滑动的速度。
                     int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);

                     if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                        // 如果滑动速度超过最小滑动速度，加速滑动
                        fling(-initialVelocity);
                     }
                 }
```
3. fling 实现代码
```
// fling 滑动实现代码
public void fling(int velocityY) {
    if (getChildCount() > 0) {
         int height = getHeight() - getPaddingBottom() - getPaddingTop();
         int bottom = getChildAt(0).getHeight();
         mScroller.fling(getScrollX(), getScrollY(), 0, velocityY, 0, 0, 0,Math.max(0, bottom - height), 0, height/2);
         postInvalidateOnAnimation();
    }
}
```


















