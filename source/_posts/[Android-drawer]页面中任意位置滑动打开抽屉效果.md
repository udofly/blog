﻿---
title: [Android-drawer]页面中任意位置滑动打开抽屉效果
---

>  DrawerLayout侧滑手势必须在屏幕边缘才可以，效果不错，但是实际用起来比较费力。我们现在要实现全屏手势侧滑，即：在Activity中，任意位置滑动打开抽屉效果
> 分析转自 https://www.jianshu.com/p/432780e4749a

**效果图如下：**

![在这里插入图片描述](https://img-blog.csdnimg.cn/0cf921e1a0bd49d7bce21707ca9448fa.gif)

实现方式：
去掉ViewDragCallback的onEdgeTouch的实现
重写onInterceptTouchEvent添加自己的拦截逻辑
修改ViewDragHelper的mEdgeSize
ViewDragHelper.Callback 是个抽象类，DrawerLayout有个实现类ViewDragCallback，里面重写了onEdgeTouched方法，没有可以修改的API，所以直接复制源码比较直接（分分钟搞定）。

使用MyDrawerLayout替换DrawerLayout
直接贴代码

```java

import android.annotation.SuppressLint;
import android.content.Context;
import android.content.res.TypedArray;
import android.graphics.Canvas;
import android.graphics.Matrix;
import android.graphics.Paint;
import android.graphics.Rect;
import android.graphics.drawable.ColorDrawable;
import android.graphics.drawable.Drawable;
import android.os.Build;
import android.os.Parcel;
import android.os.Parcelable;
import android.os.SystemClock;
import android.util.AttributeSet;
import android.view.KeyEvent;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewGroup;
import android.view.ViewParent;
import android.view.WindowInsets;
import android.view.accessibility.AccessibilityEvent;

import java.util.ArrayList;
import java.util.List;

import androidx.annotation.ColorInt;
import androidx.annotation.DrawableRes;
import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.annotation.RestrictTo;
import androidx.core.content.ContextCompat;
import androidx.core.graphics.drawable.DrawableCompat;
import androidx.core.view.AccessibilityDelegateCompat;
import androidx.core.view.GravityCompat;
import androidx.core.view.ViewCompat;
import androidx.core.view.accessibility.AccessibilityNodeInfoCompat;
import androidx.customview.view.AbsSavedState;
import androidx.customview.widget.ViewDragHelper;

//https://www.jianshu.com/p/432780e4749a

public class MyDrawerLayout extends ViewGroup {
    private static final String                                    TAG                         = "MyDrawerLayout";
    private static final int[]                                     THEME_ATTRS                 = new int[]{16843828};
    public static final  int                                       STATE_IDLE                  = 0;
    public static final  int                                       STATE_DRAGGING              = 1;
    public static final  int                                       STATE_SETTLING              = 2;
    public static final  int                                       LOCK_MODE_UNLOCKED          = 0;
    public static final  int                                       LOCK_MODE_LOCKED_CLOSED     = 1;
    public static final  int                                       LOCK_MODE_LOCKED_OPEN       = 2;
    public static final  int                                       LOCK_MODE_UNDEFINED         = 3;
    private static final int                                       MIN_DRAWER_MARGIN           = 64;
    private static final int                                       DRAWER_ELEVATION            = 10;
    private static final int                                       DEFAULT_SCRIM_COLOR         = -1728053248;
    private static final int                                       PEEK_DELAY                  = 160;
    private static final int                                       MIN_FLING_VELOCITY          = 400;
    private static final boolean                                   ALLOW_EDGE_LOCK             = false;
    private static final boolean                                   CHILDREN_DISALLOW_INTERCEPT = true;
    private static final float                                     TOUCH_SLOP_SENSITIVITY      = 1.0F;
    static final         int[]                                     LAYOUT_ATTRS                = new int[]{16842931};
    static final         boolean                                   CAN_HIDE_DESCENDANTS;
    private static final boolean                                   SET_DRAWER_SHADOW_FROM_ELEVATION;
    private final        MyDrawerLayout.ChildAccessibilityDelegate mChildAccessibilityDelegate;
    private              float                                     mDrawerElevation;
    private              int                                       mMinDrawerMargin;
    private              int                                       mScrimColor;
    private              float                                     mScrimOpacity;
    private              Paint                                     mScrimPaint;
    private final        ViewDragHelper                            mLeftDragger;
    private final        ViewDragHelper                            mRightDragger;
    private final        MyDrawerLayout.ViewDragCallback           mLeftCallback;
    private final        MyDrawerLayout.ViewDragCallback           mRightCallback;
    private              int                                       mDrawerState;
    private              boolean                                   mInLayout;
    private              boolean                                   mFirstLayout;
    private              int                                       mLockModeLeft;
    private              int                                       mLockModeRight;
    private              int                                       mLockModeStart;
    private              int                                       mLockModeEnd;
    private              boolean                                   mDisallowInterceptRequested;
    private              boolean                                   mChildrenCanceledTouch;
    @Nullable
    private              MyDrawerLayout.DrawerListener             mListener;
    private              List<MyDrawerLayout.DrawerListener>       mListeners;
    private              float                                     mInitialMotionX;
    private              float                                     mInitialMotionY;
    private              Drawable                                  mStatusBarBackground;
    private              Drawable                                  mShadowLeftResolved;
    private              Drawable                                  mShadowRightResolved;
    private              CharSequence                              mTitleLeft;
    private              CharSequence                              mTitleRight;
    private              Object                                    mLastInsets;
    private              boolean                                   mDrawStatusBarBackground;
    private              Drawable                                  mShadowStart;
    private              Drawable                                  mShadowEnd;
    private              Drawable                                  mShadowLeft;
    private              Drawable                                  mShadowRight;
    private final        ArrayList<View>                           mNonDrawerViews;
    private              Rect                                      mChildHitRect;
    private              Matrix                                    mChildInvertedMatrix;

    public MyDrawerLayout(@NonNull Context context) {
        this(context, (AttributeSet) null);
    }

    public MyDrawerLayout(@NonNull Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MyDrawerLayout(@NonNull Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        this.mChildAccessibilityDelegate = new MyDrawerLayout.ChildAccessibilityDelegate();
        this.mScrimColor = -1728053248;
        this.mScrimPaint = new Paint();
        this.mFirstLayout = true;
        this.mLockModeLeft = 3;
        this.mLockModeRight = 3;
        this.mLockModeStart = 3;
        this.mLockModeEnd = 3;
        this.mShadowStart = null;
        this.mShadowEnd = null;
        this.mShadowLeft = null;
        this.mShadowRight = null;
        this.setDescendantFocusability(262144);
        float density = this.getResources().getDisplayMetrics().density;
        this.mMinDrawerMargin = (int) (64.0F * density + 0.5F);
        float minVel = 400.0F * density;
        this.mLeftCallback = new MyDrawerLayout.ViewDragCallback(3);
        this.mRightCallback = new MyDrawerLayout.ViewDragCallback(5);
        this.mLeftDragger = ViewDragHelper.create(this, 1.0F, this.mLeftCallback);
        this.mLeftDragger.setEdgeTrackingEnabled(1);
        this.mLeftDragger.setMinVelocity(minVel);
        this.mLeftCallback.setDragger(this.mLeftDragger);
        this.mRightDragger = ViewDragHelper.create(this, 1.0F, this.mRightCallback);
        this.mRightDragger.setEdgeTrackingEnabled(2);
        this.mRightDragger.setMinVelocity(minVel);
        this.mRightCallback.setDragger(this.mRightDragger);
        this.setFocusableInTouchMode(true);
        ViewCompat.setImportantForAccessibility(this, 1);
        ViewCompat.setAccessibilityDelegate(this, new MyDrawerLayout.AccessibilityDelegate());
        this.setMotionEventSplittingEnabled(false);
        if (ViewCompat.getFitsSystemWindows(this)) {
            if (Build.VERSION.SDK_INT >= 21) {
                this.setOnApplyWindowInsetsListener(new OnApplyWindowInsetsListener() {
                    public WindowInsets onApplyWindowInsets(View view, WindowInsets insets) {
                        MyDrawerLayout MyDrawerLayout = (MyDrawerLayout) view;
                        MyDrawerLayout.setChildInsets(insets, insets.getSystemWindowInsetTop() > 0);
                        return insets.consumeSystemWindowInsets();
                    }
                });
                this.setSystemUiVisibility(1280);
                TypedArray a = context.obtainStyledAttributes(THEME_ATTRS);

                try {
                    this.mStatusBarBackground = a.getDrawable(0);
                } finally {
                    a.recycle();
                }
            } else {
                this.mStatusBarBackground = null;
            }
        }

        this.mDrawerElevation = 10.0F * density;
        this.mNonDrawerViews = new ArrayList();
    }

    public void setDrawerElevation(float elevation) {
        this.mDrawerElevation = elevation;

        for (int i = 0; i < this.getChildCount(); ++i) {
            View child = this.getChildAt(i);
            if (this.isDrawerView(child)) {
                ViewCompat.setElevation(child, this.mDrawerElevation);
            }
        }

    }

    public float getDrawerElevation() {
        return SET_DRAWER_SHADOW_FROM_ELEVATION ? this.mDrawerElevation : 0.0F;
    }

    @RestrictTo({RestrictTo.Scope.LIBRARY_GROUP})
    public void setChildInsets(Object insets, boolean draw) {
        this.mLastInsets = insets;
        this.mDrawStatusBarBackground = draw;
        this.setWillNotDraw(!draw && this.getBackground() == null);
        this.requestLayout();
    }

    public void setDrawerShadow(Drawable shadowDrawable, int gravity) {
        if (!SET_DRAWER_SHADOW_FROM_ELEVATION) {
            if ((gravity & 8388611) == 8388611) {
                this.mShadowStart = shadowDrawable;
            } else if ((gravity & 8388613) == 8388613) {
                this.mShadowEnd = shadowDrawable;
            } else if ((gravity & 3) == 3) {
                this.mShadowLeft = shadowDrawable;
            } else {
                if ((gravity & 5) != 5) {
                    return;
                }

                this.mShadowRight = shadowDrawable;
            }

            this.resolveShadowDrawables();
            this.invalidate();
        }
    }

    public void setDrawerShadow(@DrawableRes int resId, int gravity) {
        this.setDrawerShadow(ContextCompat.getDrawable(this.getContext(), resId), gravity);
    }

    public void setScrimColor(@ColorInt int color) {
        this.mScrimColor = color;
        this.invalidate();
    }

    /**
     * @deprecated
     */
    @Deprecated
    public void setDrawerListener(MyDrawerLayout.DrawerListener listener) {
        if (this.mListener != null) {
            this.removeDrawerListener(this.mListener);
        }

        if (listener != null) {
            this.addDrawerListener(listener);
        }

        this.mListener = listener;
    }

    public void addDrawerListener(@NonNull MyDrawerLayout.DrawerListener listener) {
        if (listener != null) {
            if (this.mListeners == null) {
                this.mListeners = new ArrayList();
            }

            this.mListeners.add(listener);
        }
    }

    public void removeDrawerListener(@NonNull MyDrawerLayout.DrawerListener listener) {
        if (listener != null) {
            if (this.mListeners != null) {
                this.mListeners.remove(listener);
            }
        }
    }

    public void setDrawerLockMode(int lockMode) {
        this.setDrawerLockMode(lockMode, 3);
        this.setDrawerLockMode(lockMode, 5);
    }

    public void setDrawerLockMode(int lockMode, int edgeGravity) {
        int absGravity = GravityCompat.getAbsoluteGravity(edgeGravity, ViewCompat.getLayoutDirection(this));
        switch (edgeGravity) {
            case 3:
                this.mLockModeLeft = lockMode;
                break;
            case 5:
                this.mLockModeRight = lockMode;
                break;
            case 8388611:
                this.mLockModeStart = lockMode;
                break;
            case 8388613:
                this.mLockModeEnd = lockMode;
        }

        if (lockMode != 0) {
            ViewDragHelper helper = absGravity == 3 ? this.mLeftDragger : this.mRightDragger;
            helper.cancel();
        }

        switch (lockMode) {
            case 1:
                View toClose = this.findDrawerWithGravity(absGravity);
                if (toClose != null) {
                    this.closeDrawer(toClose);
                }
                break;
            case 2:
                View toOpen = this.findDrawerWithGravity(absGravity);
                if (toOpen != null) {
                    this.openDrawer(toOpen);
                }
        }

    }

    public void setDrawerLockMode(int lockMode, @NonNull View drawerView) {
        if (!this.isDrawerView(drawerView)) {
            throw new IllegalArgumentException("View " + drawerView + " is not a " + "drawer with appropriate layout_gravity");
        } else {
            int gravity = ((MyDrawerLayout.LayoutParams) drawerView.getLayoutParams()).gravity;
            this.setDrawerLockMode(lockMode, gravity);
        }
    }

    public int getDrawerLockMode(int edgeGravity) {
        int layoutDirection = ViewCompat.getLayoutDirection(this);
        switch (edgeGravity) {
            case 3:
                if (this.mLockModeLeft != 3) {
                    return this.mLockModeLeft;
                }

                int leftLockMode = layoutDirection == 0 ? this.mLockModeStart : this.mLockModeEnd;
                if (leftLockMode != 3) {
                    return leftLockMode;
                }
                break;
            case 5:
                if (this.mLockModeRight != 3) {
                    return this.mLockModeRight;
                }

                int rightLockMode = layoutDirection == 0 ? this.mLockModeEnd : this.mLockModeStart;
                if (rightLockMode != 3) {
                    return rightLockMode;
                }
                break;
            case 8388611:
                if (this.mLockModeStart != 3) {
                    return this.mLockModeStart;
                }

                int startLockMode = layoutDirection == 0 ? this.mLockModeLeft : this.mLockModeRight;
                if (startLockMode != 3) {
                    return startLockMode;
                }
                break;
            case 8388613:
                if (this.mLockModeEnd != 3) {
                    return this.mLockModeEnd;
                }

                int endLockMode = layoutDirection == 0 ? this.mLockModeRight : this.mLockModeLeft;
                if (endLockMode != 3) {
                    return endLockMode;
                }
        }

        return 0;
    }

    public int getDrawerLockMode(@NonNull View drawerView) {
        if (!this.isDrawerView(drawerView)) {
            throw new IllegalArgumentException("View " + drawerView + " is not a drawer");
        } else {
            int drawerGravity = ((MyDrawerLayout.LayoutParams) drawerView.getLayoutParams()).gravity;
            return this.getDrawerLockMode(drawerGravity);
        }
    }

    public void setDrawerTitle(int edgeGravity, @Nullable CharSequence title) {
        int absGravity = GravityCompat.getAbsoluteGravity(edgeGravity, ViewCompat.getLayoutDirection(this));
        if (absGravity == 3) {
            this.mTitleLeft = title;
        } else if (absGravity == 5) {
            this.mTitleRight = title;
        }

    }

    @Nullable
    public CharSequence getDrawerTitle(int edgeGravity) {
        int absGravity = GravityCompat.getAbsoluteGravity(edgeGravity, ViewCompat.getLayoutDirection(this));
        if (absGravity == 3) {
            return this.mTitleLeft;
        } else {
            return absGravity == 5 ? this.mTitleRight : null;
        }
    }

    private boolean isInBoundsOfChild(float x, float y, View child) {
        if (this.mChildHitRect == null) {
            this.mChildHitRect = new Rect();
        }

        child.getHitRect(this.mChildHitRect);
        return this.mChildHitRect.contains((int) x, (int) y);
    }

    private boolean dispatchTransformedGenericPointerEvent(MotionEvent event, View child) {
        Matrix  childMatrix = child.getMatrix();
        boolean handled;
        if (!childMatrix.isIdentity()) {
            MotionEvent transformedEvent = this.getTransformedMotionEvent(event, child);
            handled = child.dispatchGenericMotionEvent(transformedEvent);
            transformedEvent.recycle();
        } else {
            float offsetX = (float) (this.getScrollX() - child.getLeft());
            float offsetY = (float) (this.getScrollY() - child.getTop());
            event.offsetLocation(offsetX, offsetY);
            handled = child.dispatchGenericMotionEvent(event);
            event.offsetLocation(-offsetX, -offsetY);
        }

        return handled;
    }

    private MotionEvent getTransformedMotionEvent(MotionEvent event, View child) {
        float       offsetX          = (float) (this.getScrollX() - child.getLeft());
        float       offsetY          = (float) (this.getScrollY() - child.getTop());
        MotionEvent transformedEvent = MotionEvent.obtain(event);
        transformedEvent.offsetLocation(offsetX, offsetY);
        Matrix childMatrix = child.getMatrix();
        if (!childMatrix.isIdentity()) {
            if (this.mChildInvertedMatrix == null) {
                this.mChildInvertedMatrix = new Matrix();
            }

            childMatrix.invert(this.mChildInvertedMatrix);
            transformedEvent.transform(this.mChildInvertedMatrix);
        }

        return transformedEvent;
    }

    void updateDrawerState(int forGravity, int activeState, View activeDrawer) {
        int  leftState  = this.mLeftDragger.getViewDragState();
        int  rightState = this.mRightDragger.getViewDragState();
        byte state;
        if (leftState != 1 && rightState != 1) {
            if (leftState != 2 && rightState != 2) {
                state = 0;
            } else {
                state = 2;
            }
        } else {
            state = 1;
        }

        if (activeDrawer != null && activeState == 0) {
            MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) activeDrawer.getLayoutParams();
            if (lp.onScreen == 0.0F) {
                this.dispatchOnDrawerClosed(activeDrawer);
            } else if (lp.onScreen == 1.0F) {
                this.dispatchOnDrawerOpened(activeDrawer);
            }
        }

        if (state != this.mDrawerState) {
            this.mDrawerState = state;
            if (this.mListeners != null) {
                int listenerCount = this.mListeners.size();

                for (int i = listenerCount - 1; i >= 0; --i) {
                    ((MyDrawerLayout.DrawerListener) this.mListeners.get(i)).onDrawerStateChanged(state);
                }
            }
        }

    }

    void dispatchOnDrawerClosed(View drawerView) {
        MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) drawerView.getLayoutParams();
        if ((lp.openState & 1) == 1) {
            lp.openState = 0;
            if (this.mListeners != null) {
                int listenerCount = this.mListeners.size();

                for (int i = listenerCount - 1; i >= 0; --i) {
                    ((MyDrawerLayout.DrawerListener) this.mListeners.get(i)).onDrawerClosed(drawerView);
                }
            }

            this.updateChildrenImportantForAccessibility(drawerView, false);
            if (this.hasWindowFocus()) {
                View rootView = this.getRootView();
                if (rootView != null) {
                    rootView.sendAccessibilityEvent(32);
                }
            }
        }

    }

    void dispatchOnDrawerOpened(View drawerView) {
        MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) drawerView.getLayoutParams();
        if ((lp.openState & 1) == 0) {
            lp.openState = 1;
            if (this.mListeners != null) {
                int listenerCount = this.mListeners.size();

                for (int i = listenerCount - 1; i >= 0; --i) {
                    ((MyDrawerLayout.DrawerListener) this.mListeners.get(i)).onDrawerOpened(drawerView);
                }
            }

            this.updateChildrenImportantForAccessibility(drawerView, true);
            if (this.hasWindowFocus()) {
                this.sendAccessibilityEvent(32);
            }
        }

    }

    private void updateChildrenImportantForAccessibility(View drawerView, boolean isDrawerOpen) {
        int childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View child = this.getChildAt(i);
            if ((isDrawerOpen || this.isDrawerView(child)) && (!isDrawerOpen || child != drawerView)) {
                ViewCompat.setImportantForAccessibility(child, 4);
            } else {
                ViewCompat.setImportantForAccessibility(child, 1);
            }
        }

    }

    void dispatchOnDrawerSlide(View drawerView, float slideOffset) {
        if (this.mListeners != null) {
            int listenerCount = this.mListeners.size();

            for (int i = listenerCount - 1; i >= 0; --i) {
                ((MyDrawerLayout.DrawerListener) this.mListeners.get(i)).onDrawerSlide(drawerView, slideOffset);
            }
        }

    }

    void setDrawerViewOffset(View drawerView, float slideOffset) {
        MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) drawerView.getLayoutParams();
        if (slideOffset != lp.onScreen) {
            lp.onScreen = slideOffset;
            this.dispatchOnDrawerSlide(drawerView, slideOffset);
        }
    }

    float getDrawerViewOffset(View drawerView) {
        return ((MyDrawerLayout.LayoutParams) drawerView.getLayoutParams()).onScreen;
    }

    int getDrawerViewAbsoluteGravity(View drawerView) {
        int gravity = ((MyDrawerLayout.LayoutParams) drawerView.getLayoutParams()).gravity;
        return GravityCompat.getAbsoluteGravity(gravity, ViewCompat.getLayoutDirection(this));
    }

    boolean checkDrawerViewAbsoluteGravity(View drawerView, int checkFor) {
        int absGravity = this.getDrawerViewAbsoluteGravity(drawerView);
        return (absGravity & checkFor) == checkFor;
    }

    View findOpenDrawer() {
        int childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View                        child   = this.getChildAt(i);
            MyDrawerLayout.LayoutParams childLp = (MyDrawerLayout.LayoutParams) child.getLayoutParams();
            if ((childLp.openState & 1) == 1) {
                return child;
            }
        }

        return null;
    }

    void moveDrawerToOffset(View drawerView, float slideOffset) {
        float oldOffset = this.getDrawerViewOffset(drawerView);
        int   width     = drawerView.getWidth();
        int   oldPos    = (int) ((float) width * oldOffset);
        int   newPos    = (int) ((float) width * slideOffset);
        int   dx        = newPos - oldPos;
        drawerView.offsetLeftAndRight(this.checkDrawerViewAbsoluteGravity(drawerView, 3) ? dx : -dx);
        this.setDrawerViewOffset(drawerView, slideOffset);
    }

    View findDrawerWithGravity(int gravity) {
        int absHorizGravity = GravityCompat.getAbsoluteGravity(gravity, ViewCompat.getLayoutDirection(this)) & 7;
        int childCount      = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View child           = this.getChildAt(i);
            int  childAbsGravity = this.getDrawerViewAbsoluteGravity(child);
            if ((childAbsGravity & 7) == absHorizGravity) {
                return child;
            }
        }

        return null;
    }

    static String gravityToString(int gravity) {
        if ((gravity & 3) == 3) {
            return "LEFT";
        } else {
            return (gravity & 5) == 5 ? "RIGHT" : Integer.toHexString(gravity);
        }
    }

    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        this.mFirstLayout = true;
    }

    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        this.mFirstLayout = true;
    }

    @SuppressLint({"WrongConstant"})
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode  = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize  = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        if (widthMode != 1073741824 || heightMode != 1073741824) {
            if (!this.isInEditMode()) {
                throw new IllegalArgumentException("MyDrawerLayout must be measured with MeasureSpec.EXACTLY.");
            }

            if (widthMode == -2147483648) {
                widthMode = 1073741824;
            } else if (widthMode == 0) {
                widthMode = 1073741824;
                widthSize = 300;
            }

            if (heightMode == -2147483648) {
                heightMode = 1073741824;
            } else if (heightMode == 0) {
                heightMode = 1073741824;
                heightSize = 300;
            }
        }

        this.setMeasuredDimension(widthSize, heightSize);
        boolean applyInsets          = this.mLastInsets != null && ViewCompat.getFitsSystemWindows(this);
        int     layoutDirection      = ViewCompat.getLayoutDirection(this);
        boolean hasDrawerOnLeftEdge  = false;
        boolean hasDrawerOnRightEdge = false;
        int     childCount           = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View child = this.getChildAt(i);
            if (child.getVisibility() != 8) {
                MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) child.getLayoutParams();
                int                         childGravity;
                if (applyInsets) {
                    childGravity = GravityCompat.getAbsoluteGravity(lp.gravity, layoutDirection);
                    WindowInsets wi;
                    if (ViewCompat.getFitsSystemWindows(child)) {
                        if (Build.VERSION.SDK_INT >= 21) {
                            wi = (WindowInsets) this.mLastInsets;
                            if (childGravity == 3) {
                                wi = wi.replaceSystemWindowInsets(wi.getSystemWindowInsetLeft(), wi.getSystemWindowInsetTop(), 0, wi.getSystemWindowInsetBottom());
                            } else if (childGravity == 5) {
                                wi = wi.replaceSystemWindowInsets(0, wi.getSystemWindowInsetTop(), wi.getSystemWindowInsetRight(), wi.getSystemWindowInsetBottom());
                            }

                            child.dispatchApplyWindowInsets(wi);
                        }
                    } else if (Build.VERSION.SDK_INT >= 21) {
                        wi = (WindowInsets) this.mLastInsets;
                        if (childGravity == 3) {
                            wi = wi.replaceSystemWindowInsets(wi.getSystemWindowInsetLeft(), wi.getSystemWindowInsetTop(), 0, wi.getSystemWindowInsetBottom());
                        } else if (childGravity == 5) {
                            wi = wi.replaceSystemWindowInsets(0, wi.getSystemWindowInsetTop(), wi.getSystemWindowInsetRight(), wi.getSystemWindowInsetBottom());
                        }

                        lp.leftMargin = wi.getSystemWindowInsetLeft();
                        lp.topMargin = wi.getSystemWindowInsetTop();
                        lp.rightMargin = wi.getSystemWindowInsetRight();
                        lp.bottomMargin = wi.getSystemWindowInsetBottom();
                    }
                }

                if (this.isContentView(child)) {
                    childGravity = MeasureSpec.makeMeasureSpec(widthSize - lp.leftMargin - lp.rightMargin, 1073741824);
                    int contentHeightSpec = MeasureSpec.makeMeasureSpec(heightSize - lp.topMargin - lp.bottomMargin, 1073741824);
                    child.measure(childGravity, contentHeightSpec);
                } else {
                    if (!this.isDrawerView(child)) {
                        throw new IllegalStateException("Child " + child + " at index " + i + " does not have a valid layout_gravity - must be Gravity.LEFT, " + "Gravity.RIGHT or Gravity.NO_GRAVITY");
                    }

                    if (SET_DRAWER_SHADOW_FROM_ELEVATION && ViewCompat.getElevation(child) != this.mDrawerElevation) {
                        ViewCompat.setElevation(child, this.mDrawerElevation);
                    }

                    childGravity = this.getDrawerViewAbsoluteGravity(child) & 7;
                    boolean isLeftEdgeDrawer = childGravity == 3;
                    if (isLeftEdgeDrawer && hasDrawerOnLeftEdge || !isLeftEdgeDrawer && hasDrawerOnRightEdge) {
                        throw new IllegalStateException("Child drawer has absolute gravity " + gravityToString(childGravity) + " but this " + "MyDrawerLayout" + " already has a " + "drawer view along that edge");
                    }

                    if (isLeftEdgeDrawer) {
                        hasDrawerOnLeftEdge = true;
                    } else {
                        hasDrawerOnRightEdge = true;
                    }

                    int drawerWidthSpec  = getChildMeasureSpec(widthMeasureSpec, this.mMinDrawerMargin + lp.leftMargin + lp.rightMargin, lp.width);
                    int drawerHeightSpec = getChildMeasureSpec(heightMeasureSpec, lp.topMargin + lp.bottomMargin, lp.height);
                    child.measure(drawerWidthSpec, drawerHeightSpec);
                }
            }
        }

    }

    private void resolveShadowDrawables() {
        if (!SET_DRAWER_SHADOW_FROM_ELEVATION) {
            this.mShadowLeftResolved = this.resolveLeftShadow();
            this.mShadowRightResolved = this.resolveRightShadow();
        }
    }

    private Drawable resolveLeftShadow() {
        int layoutDirection = ViewCompat.getLayoutDirection(this);
        if (layoutDirection == 0) {
            if (this.mShadowStart != null) {
                this.mirror(this.mShadowStart, layoutDirection);
                return this.mShadowStart;
            }
        } else if (this.mShadowEnd != null) {
            this.mirror(this.mShadowEnd, layoutDirection);
            return this.mShadowEnd;
        }

        return this.mShadowLeft;
    }

    private Drawable resolveRightShadow() {
        int layoutDirection = ViewCompat.getLayoutDirection(this);
        if (layoutDirection == 0) {
            if (this.mShadowEnd != null) {
                this.mirror(this.mShadowEnd, layoutDirection);
                return this.mShadowEnd;
            }
        } else if (this.mShadowStart != null) {
            this.mirror(this.mShadowStart, layoutDirection);
            return this.mShadowStart;
        }

        return this.mShadowRight;
    }

    private boolean mirror(Drawable drawable, int layoutDirection) {
        if (drawable != null && DrawableCompat.isAutoMirrored(drawable)) {
            DrawableCompat.setLayoutDirection(drawable, layoutDirection);
            return true;
        } else {
            return false;
        }
    }

    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        this.mInLayout = true;
        int width      = r - l;
        int childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View child = this.getChildAt(i);
            if (child.getVisibility() != 8) {
                MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) child.getLayoutParams();
                if (this.isContentView(child)) {
                    child.layout(lp.leftMargin, lp.topMargin, lp.leftMargin + child.getMeasuredWidth(), lp.topMargin + child.getMeasuredHeight());
                } else {
                    int   childWidth  = child.getMeasuredWidth();
                    int   childHeight = child.getMeasuredHeight();
                    int   childLeft;
                    float newOffset;
                    if (this.checkDrawerViewAbsoluteGravity(child, 3)) {
                        childLeft = -childWidth + (int) ((float) childWidth * lp.onScreen);
                        newOffset = (float) (childWidth + childLeft) / (float) childWidth;
                    } else {
                        childLeft = width - (int) ((float) childWidth * lp.onScreen);
                        newOffset = (float) (width - childLeft) / (float) childWidth;
                    }

                    boolean changeOffset = newOffset != lp.onScreen;
                    int     vgrav        = lp.gravity & 112;
                    int     newVisibility;
                    switch (vgrav) {
                        case 16:
                            newVisibility = b - t;
                            int childTop = (newVisibility - childHeight) / 2;
                            if (childTop < lp.topMargin) {
                                childTop = lp.topMargin;
                            } else if (childTop + childHeight > newVisibility - lp.bottomMargin) {
                                childTop = newVisibility - lp.bottomMargin - childHeight;
                            }

                            child.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
                            break;
                        case 48:
                        default:
                            child.layout(childLeft, lp.topMargin, childLeft + childWidth, lp.topMargin + childHeight);
                            break;
                        case 80:
                            newVisibility = b - t;
                            child.layout(childLeft, newVisibility - lp.bottomMargin - child.getMeasuredHeight(), childLeft + childWidth, newVisibility - lp.bottomMargin);
                    }

                    if (changeOffset) {
                        this.setDrawerViewOffset(child, newOffset);
                    }

                    newVisibility = lp.onScreen > 0.0F ? 0 : 4;
                    if (child.getVisibility() != newVisibility) {
                        child.setVisibility(newVisibility);
                    }
                }
            }
        }

        this.mInLayout = false;
        this.mFirstLayout = false;
    }

    public void requestLayout() {
        if (!this.mInLayout) {
            super.requestLayout();
        }

    }

    public void computeScroll() {
        int   childCount   = this.getChildCount();
        float scrimOpacity = 0.0F;

        for (int i = 0; i < childCount; ++i) {
            float onscreen = ((MyDrawerLayout.LayoutParams) this.getChildAt(i).getLayoutParams()).onScreen;
            scrimOpacity = Math.max(scrimOpacity, onscreen);
        }

        this.mScrimOpacity = scrimOpacity;
        boolean leftDraggerSettling  = this.mLeftDragger.continueSettling(true);
        boolean rightDraggerSettling = this.mRightDragger.continueSettling(true);
        if (leftDraggerSettling || rightDraggerSettling) {
            ViewCompat.postInvalidateOnAnimation(this);
        }

    }

    private static boolean hasOpaqueBackground(View v) {
        Drawable bg = v.getBackground();
        if (bg != null) {
            return bg.getOpacity() == -1;
        } else {
            return false;
        }
    }

    public void setStatusBarBackground(@Nullable Drawable bg) {
        this.mStatusBarBackground = bg;
        this.invalidate();
    }

    @Nullable
    public Drawable getStatusBarBackgroundDrawable() {
        return this.mStatusBarBackground;
    }

    public void setStatusBarBackground(int resId) {
        this.mStatusBarBackground = resId != 0 ? ContextCompat.getDrawable(this.getContext(), resId) : null;
        this.invalidate();
    }

    public void setStatusBarBackgroundColor(@ColorInt int color) {
        this.mStatusBarBackground = new ColorDrawable(color);
        this.invalidate();
    }

    public void onRtlPropertiesChanged(int layoutDirection) {
        this.resolveShadowDrawables();
    }

    public void onDraw(Canvas c) {
        super.onDraw(c);
        if (this.mDrawStatusBarBackground && this.mStatusBarBackground != null) {
            int inset;
            if (Build.VERSION.SDK_INT >= 21) {
                inset = this.mLastInsets != null ? ((WindowInsets) this.mLastInsets).getSystemWindowInsetTop() : 0;
            } else {
                inset = 0;
            }

            if (inset > 0) {
                this.mStatusBarBackground.setBounds(0, 0, this.getWidth(), inset);
                this.mStatusBarBackground.draw(c);
            }
        }

    }

    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        int     height         = this.getHeight();
        boolean drawingContent = this.isContentView(child);
        int     clipLeft       = 0;
        int     clipRight      = this.getWidth();
        int     restoreCount   = canvas.save();
        int     shadowWidth;
        int     vright;
        if (drawingContent) {
            int childCount = this.getChildCount();

            for (shadowWidth = 0; shadowWidth < childCount; ++shadowWidth) {
                View v = this.getChildAt(shadowWidth);
                if (v != child && v.getVisibility() == 0 && hasOpaqueBackground(v) && this.isDrawerView(v) && v.getHeight() >= height) {
                    if (this.checkDrawerViewAbsoluteGravity(v, 3)) {
                        vright = v.getRight();
                        if (vright > clipLeft) {
                            clipLeft = vright;
                        }
                    } else {
                        vright = v.getLeft();
                        if (vright < clipRight) {
                            clipRight = vright;
                        }
                    }
                }
            }

            canvas.clipRect(clipLeft, 0, clipRight, this.getHeight());
        }

        boolean result = super.drawChild(canvas, child, drawingTime);
        canvas.restoreToCount(restoreCount);
        int childLeft;
        if (this.mScrimOpacity > 0.0F && drawingContent) {
            shadowWidth = (this.mScrimColor & -16777216) >>> 24;
            childLeft = (int) ((float) shadowWidth * this.mScrimOpacity);
            vright = childLeft << 24 | this.mScrimColor & 16777215;
            this.mScrimPaint.setColor(vright);
            canvas.drawRect((float) clipLeft, 0.0F, (float) clipRight, (float) this.getHeight(), this.mScrimPaint);
        } else if (this.mShadowLeftResolved != null && this.checkDrawerViewAbsoluteGravity(child, 3)) {
            shadowWidth = this.mShadowLeftResolved.getIntrinsicWidth();
            childLeft = child.getRight();
            vright = this.mLeftDragger.getEdgeSize();
            float alpha = Math.max(0.0F, Math.min((float) childLeft / (float) vright, 1.0F));
            this.mShadowLeftResolved.setBounds(childLeft, child.getTop(), childLeft + shadowWidth, child.getBottom());
            this.mShadowLeftResolved.setAlpha((int) (255.0F * alpha));
            this.mShadowLeftResolved.draw(canvas);
        } else if (this.mShadowRightResolved != null && this.checkDrawerViewAbsoluteGravity(child, 5)) {
            shadowWidth = this.mShadowRightResolved.getIntrinsicWidth();
            childLeft = child.getLeft();
            vright = this.getWidth() - childLeft;
            int   drawerPeekDistance = this.mRightDragger.getEdgeSize();
            float alpha              = Math.max(0.0F, Math.min((float) vright / (float) drawerPeekDistance, 1.0F));
            this.mShadowRightResolved.setBounds(childLeft - shadowWidth, child.getTop(), childLeft, child.getBottom());
            this.mShadowRightResolved.setAlpha((int) (255.0F * alpha));
            this.mShadowRightResolved.draw(canvas);
        }

        return result;
    }

    boolean isContentView(View child) {
        return ((MyDrawerLayout.LayoutParams) child.getLayoutParams()).gravity == 0;
    }

    boolean isDrawerView(View child) {
        int gravity    = ((MyDrawerLayout.LayoutParams) child.getLayoutParams()).gravity;
        int absGravity = GravityCompat.getAbsoluteGravity(gravity, ViewCompat.getLayoutDirection(child));
        if ((absGravity & 3) != 0) {
            return true;
        } else {
            return (absGravity & 5) != 0;
        }
    }

    public boolean onInterceptTouchEvent(MotionEvent ev) {
//        int     action           = ev.getActionMasked();
//        boolean interceptForDrag = this.mLeftDragger.shouldInterceptTouchEvent(ev) | this.mRightDragger.shouldInterceptTouchEvent(ev);
//        boolean interceptForTap  = false;
//        switch (action) {
//            case 0:
//                float x = ev.getX();
//                float y = ev.getY();
//                this.mInitialMotionX = x;
//                this.mInitialMotionY = y;
//                if (this.mScrimOpacity > 0.0F) {
//                    View child = this.mLeftDragger.findTopChildUnder((int) x, (int) y);
//                    if (child != null && this.isContentView(child)) {
//                        interceptForTap = true;
//                    }
//                }
//
//                this.mDisallowInterceptRequested = false;
//                this.mChildrenCanceledTouch = false;
//                break;
//            case 1:
//            case 3:
//                this.closeDrawers(true);
//                this.mDisallowInterceptRequested = false;
//                this.mChildrenCanceledTouch = false;
//                break;
//            case 2:
//                if (this.mLeftDragger.checkTouchSlop(3)) {
//                    this.mLeftCallback.removeCallbacks();
//                    this.mRightCallback.removeCallbacks();
//                }
//        }
//
//        return interceptForDrag || interceptForTap || this.hasPeekingDrawer() || this.mChildrenCanceledTouch;


        //其实就是吧原来的实现放到一个新的方法里，然后添加自己的逻辑。
        try {
            final float x = ev.getX();
            final float y = ev.getY();

            switch (ev.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    this.mInitialMotionX = x;
                    this.mInitialMotionY = y;
                    break;

                case MotionEvent.ACTION_MOVE:
                    //这里的判断拦截的逻辑是滑动的角度小于等于30°就是横向滑动，肯定拦截
                    //否者使用原来的逻辑，调用interceptTouchEvent
                    //这样写主要是有垂直滚动的RecyclewrView
                    //具体怎么处理，看自己具体需求
                    float xDiff = Math.abs(x - this.mInitialMotionX);
                    float yDiff = Math.abs(y - this.mInitialMotionY);
                    return xDiff > 0 && xDiff >= yDiff * Math.sqrt(3);
            }
            return interceptTouchEvent(ev);
        } catch (IllegalArgumentException ex) {
            ex.printStackTrace();
            return false;
        }


    }

    //这就是原来的onInterceptTouchEvent
    private boolean interceptTouchEvent(MotionEvent ev) {
        final int action = ev.getActionMasked();

        // "|" used deliberately here; both methods should be invoked.
        final boolean interceptForDrag = mLeftDragger.shouldInterceptTouchEvent(ev)
                | mRightDragger.shouldInterceptTouchEvent(ev);

        boolean interceptForTap = false;

        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                mInitialMotionX = x;
                mInitialMotionY = y;
                if (mScrimOpacity > 0) {
                    final View child = mLeftDragger.findTopChildUnder((int) x, (int) y);
                    if (child != null && isContentView(child)) {
                        interceptForTap = true;
                    }
                }
                mDisallowInterceptRequested = false;
                mChildrenCanceledTouch = false;
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                // If we cross the touch slop, don't perform the delayed peek for an edge touch.
                if (mLeftDragger.checkTouchSlop(ViewDragHelper.DIRECTION_ALL)) {
                    mLeftCallback.removeCallbacks();
                    mRightCallback.removeCallbacks();
                }
                break;
            }

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP: {
                closeDrawers(true);
                mDisallowInterceptRequested = false;
                mChildrenCanceledTouch = false;
            }
        }

        return interceptForDrag || interceptForTap || hasPeekingDrawer() || mChildrenCanceledTouch;
    }

    public boolean dispatchGenericMotionEvent(MotionEvent event) {
        if ((event.getSource() & 2) != 0 && event.getAction() != 10 && this.mScrimOpacity > 0.0F) {
            int childrenCount = this.getChildCount();
            if (childrenCount != 0) {
                float x = event.getX();
                float y = event.getY();

                for (int i = childrenCount - 1; i >= 0; --i) {
                    View child = this.getChildAt(i);
                    if (this.isInBoundsOfChild(x, y, child) && !this.isContentView(child) && this.dispatchTransformedGenericPointerEvent(event, child)) {
                        return true;
                    }
                }
            }

            return false;
        } else {
            return super.dispatchGenericMotionEvent(event);
        }
    }

    public boolean onTouchEvent(MotionEvent ev) {
        this.mLeftDragger.processTouchEvent(ev);
        this.mRightDragger.processTouchEvent(ev);
        int     action          = ev.getAction();
        boolean wantTouchEvents = true;
        float   x;
        float   y;
        switch (action & 255) {
            case 0:
                x = ev.getX();
                y = ev.getY();
                this.mInitialMotionX = x;
                this.mInitialMotionY = y;
                this.mDisallowInterceptRequested = false;
                this.mChildrenCanceledTouch = false;
                break;
            case 1:
                x = ev.getX();
                y = ev.getY();
                boolean peekingOnly = true;
                View touchedView = this.mLeftDragger.findTopChildUnder((int) x, (int) y);
                if (touchedView != null && this.isContentView(touchedView)) {
                    float dx   = x - this.mInitialMotionX;
                    float dy   = y - this.mInitialMotionY;
                    int   slop = this.mLeftDragger.getTouchSlop();
                    if (dx * dx + dy * dy < (float) (slop * slop)) {
                        View openDrawer = this.findOpenDrawer();
                        if (openDrawer != null) {
                            peekingOnly = this.getDrawerLockMode(openDrawer) == 2;
                        }
                    }
                }

                this.closeDrawers(peekingOnly);
                this.mDisallowInterceptRequested = false;
            case 2:
            default:
                break;
            case 3:
                this.closeDrawers(true);
                this.mDisallowInterceptRequested = false;
                this.mChildrenCanceledTouch = false;
        }

        return wantTouchEvents;
    }

    public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
        super.requestDisallowInterceptTouchEvent(disallowIntercept);
        this.mDisallowInterceptRequested = disallowIntercept;
        if (disallowIntercept) {
            this.closeDrawers(true);
        }

    }

    public void closeDrawers() {
        this.closeDrawers(false);
    }

    void closeDrawers(boolean peekingOnly) {
        boolean needsInvalidate = false;
        int     childCount      = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View                        child = this.getChildAt(i);
            MyDrawerLayout.LayoutParams lp    = (MyDrawerLayout.LayoutParams) child.getLayoutParams();
            if (this.isDrawerView(child) && (!peekingOnly || lp.isPeeking)) {
                int childWidth = child.getWidth();
                if (this.checkDrawerViewAbsoluteGravity(child, 3)) {
                    needsInvalidate |= this.mLeftDragger.smoothSlideViewTo(child, -childWidth, child.getTop());
                } else {
                    needsInvalidate |= this.mRightDragger.smoothSlideViewTo(child, this.getWidth(), child.getTop());
                }

                lp.isPeeking = false;
            }
        }

        this.mLeftCallback.removeCallbacks();
        this.mRightCallback.removeCallbacks();
        if (needsInvalidate) {
            this.invalidate();
        }

    }

    public void openDrawer(@NonNull View drawerView) {
        this.openDrawer(drawerView, true);
    }

    public void openDrawer(@NonNull View drawerView, boolean animate) {
        if (!this.isDrawerView(drawerView)) {
            throw new IllegalArgumentException("View " + drawerView + " is not a sliding drawer");
        } else {
            MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) drawerView.getLayoutParams();
            if (this.mFirstLayout) {
                lp.onScreen = 1.0F;
                lp.openState = 1;
                this.updateChildrenImportantForAccessibility(drawerView, true);
            } else if (animate) {
                lp.openState |= 2;
                if (this.checkDrawerViewAbsoluteGravity(drawerView, 3)) {
                    this.mLeftDragger.smoothSlideViewTo(drawerView, 0, drawerView.getTop());
                } else {
                    this.mRightDragger.smoothSlideViewTo(drawerView, this.getWidth() - drawerView.getWidth(), drawerView.getTop());
                }
            } else {
                this.moveDrawerToOffset(drawerView, 1.0F);
                this.updateDrawerState(lp.gravity, 0, drawerView);
                drawerView.setVisibility(0);
            }

            this.invalidate();
        }
    }

    public void openDrawer(int gravity) {
        this.openDrawer(gravity, true);
    }

    public void openDrawer(int gravity, boolean animate) {
        View drawerView = this.findDrawerWithGravity(gravity);
        if (drawerView == null) {
            throw new IllegalArgumentException("No drawer view found with gravity " + gravityToString(gravity));
        } else {
            this.openDrawer(drawerView, animate);
        }
    }

    public void closeDrawer(@NonNull View drawerView) {
        this.closeDrawer(drawerView, true);
    }

    public void closeDrawer(@NonNull View drawerView, boolean animate) {
        if (!this.isDrawerView(drawerView)) {
            throw new IllegalArgumentException("View " + drawerView + " is not a sliding drawer");
        } else {
            MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) drawerView.getLayoutParams();
            if (this.mFirstLayout) {
                lp.onScreen = 0.0F;
                lp.openState = 0;
            } else if (animate) {
                lp.openState |= 4;
                if (this.checkDrawerViewAbsoluteGravity(drawerView, 3)) {
                    this.mLeftDragger.smoothSlideViewTo(drawerView, -drawerView.getWidth(), drawerView.getTop());
                } else {
                    this.mRightDragger.smoothSlideViewTo(drawerView, this.getWidth(), drawerView.getTop());
                }
            } else {
                this.moveDrawerToOffset(drawerView, 0.0F);
                this.updateDrawerState(lp.gravity, 0, drawerView);
                drawerView.setVisibility(4);
            }

            this.invalidate();
        }
    }

    public void closeDrawer(int gravity) {
        this.closeDrawer(gravity, true);
    }

    public void closeDrawer(int gravity, boolean animate) {
        View drawerView = this.findDrawerWithGravity(gravity);
        if (drawerView == null) {
            throw new IllegalArgumentException("No drawer view found with gravity " + gravityToString(gravity));
        } else {
            this.closeDrawer(drawerView, animate);
        }
    }

    public boolean isDrawerOpen(@NonNull View drawer) {
        if (!this.isDrawerView(drawer)) {
            throw new IllegalArgumentException("View " + drawer + " is not a drawer");
        } else {
            MyDrawerLayout.LayoutParams drawerLp = (MyDrawerLayout.LayoutParams) drawer.getLayoutParams();
            return (drawerLp.openState & 1) == 1;
        }
    }

    public boolean isDrawerOpen(int drawerGravity) {
        View drawerView = this.findDrawerWithGravity(drawerGravity);
        return drawerView != null ? this.isDrawerOpen(drawerView) : false;
    }

    public boolean isDrawerVisible(@NonNull View drawer) {
        if (!this.isDrawerView(drawer)) {
            throw new IllegalArgumentException("View " + drawer + " is not a drawer");
        } else {
            return ((MyDrawerLayout.LayoutParams) drawer.getLayoutParams()).onScreen > 0.0F;
        }
    }

    public boolean isDrawerVisible(int drawerGravity) {
        View drawerView = this.findDrawerWithGravity(drawerGravity);
        return drawerView != null ? this.isDrawerVisible(drawerView) : false;
    }

    private boolean hasPeekingDrawer() {
        int childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) this.getChildAt(i).getLayoutParams();
            if (lp.isPeeking) {
                return true;
            }
        }

        return false;
    }

    protected android.view.ViewGroup.LayoutParams generateDefaultLayoutParams() {
        return new MyDrawerLayout.LayoutParams(-1, -1);
    }

    protected android.view.ViewGroup.LayoutParams generateLayoutParams(android.view.ViewGroup.LayoutParams p) {
        return p instanceof MyDrawerLayout.LayoutParams ? new MyDrawerLayout.LayoutParams((MyDrawerLayout.LayoutParams) p) : (p instanceof MarginLayoutParams ? new MyDrawerLayout.LayoutParams((MarginLayoutParams) p) : new MyDrawerLayout.LayoutParams(p));
    }

    protected boolean checkLayoutParams(android.view.ViewGroup.LayoutParams p) {
        return p instanceof MyDrawerLayout.LayoutParams && super.checkLayoutParams(p);
    }

    public android.view.ViewGroup.LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MyDrawerLayout.LayoutParams(this.getContext(), attrs);
    }

    public void addFocusables(ArrayList<View> views, int direction, int focusableMode) {
        if (this.getDescendantFocusability() != 393216) {
            int     childCount   = this.getChildCount();
            boolean isDrawerOpen = false;

            int nonDrawerViewsCount;
            for (nonDrawerViewsCount = 0; nonDrawerViewsCount < childCount; ++nonDrawerViewsCount) {
                View child = this.getChildAt(nonDrawerViewsCount);
                if (this.isDrawerView(child)) {
                    if (this.isDrawerOpen(child)) {
                        isDrawerOpen = true;
                        child.addFocusables(views, direction, focusableMode);
                    }
                } else {
                    this.mNonDrawerViews.add(child);
                }
            }

            if (!isDrawerOpen) {
                nonDrawerViewsCount = this.mNonDrawerViews.size();

                for (int i = 0; i < nonDrawerViewsCount; ++i) {
                    View child = (View) this.mNonDrawerViews.get(i);
                    if (child.getVisibility() == 0) {
                        child.addFocusables(views, direction, focusableMode);
                    }
                }
            }

            this.mNonDrawerViews.clear();
        }
    }

    private boolean hasVisibleDrawer() {
        return this.findVisibleDrawer() != null;
    }

    View findVisibleDrawer() {
        int childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View child = this.getChildAt(i);
            if (this.isDrawerView(child) && this.isDrawerVisible(child)) {
                return child;
            }
        }

        return null;
    }

    void cancelChildViewTouch() {
        if (!this.mChildrenCanceledTouch) {
            long        now         = SystemClock.uptimeMillis();
            MotionEvent cancelEvent = MotionEvent.obtain(now, now, 3, 0.0F, 0.0F, 0);
            int         childCount  = this.getChildCount();

            for (int i = 0; i < childCount; ++i) {
                this.getChildAt(i).dispatchTouchEvent(cancelEvent);
            }

            cancelEvent.recycle();
            this.mChildrenCanceledTouch = true;
        }

    }

    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == 4 && this.hasVisibleDrawer()) {
            event.startTracking();
            return true;
        } else {
            return super.onKeyDown(keyCode, event);
        }
    }

    public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (keyCode == 4) {
            View visibleDrawer = this.findVisibleDrawer();
            if (visibleDrawer != null && this.getDrawerLockMode(visibleDrawer) == 0) {
                this.closeDrawers();
            }

            return visibleDrawer != null;
        } else {
            return super.onKeyUp(keyCode, event);
        }
    }

    protected void onRestoreInstanceState(Parcelable state) {
        if (!(state instanceof MyDrawerLayout.SavedState)) {
            super.onRestoreInstanceState(state);
        } else {
            MyDrawerLayout.SavedState ss = (MyDrawerLayout.SavedState) state;
            super.onRestoreInstanceState(ss.getSuperState());
            if (ss.openDrawerGravity != 0) {
                View toOpen = this.findDrawerWithGravity(ss.openDrawerGravity);
                if (toOpen != null) {
                    this.openDrawer(toOpen);
                }
            }

            if (ss.lockModeLeft != 3) {
                this.setDrawerLockMode(ss.lockModeLeft, 3);
            }

            if (ss.lockModeRight != 3) {
                this.setDrawerLockMode(ss.lockModeRight, 5);
            }

            if (ss.lockModeStart != 3) {
                this.setDrawerLockMode(ss.lockModeStart, 8388611);
            }

            if (ss.lockModeEnd != 3) {
                this.setDrawerLockMode(ss.lockModeEnd, 8388613);
            }

        }
    }

    protected Parcelable onSaveInstanceState() {
        Parcelable                superState = super.onSaveInstanceState();
        MyDrawerLayout.SavedState ss         = new MyDrawerLayout.SavedState(superState);
        int                       childCount = this.getChildCount();

        for (int i = 0; i < childCount; ++i) {
            View                        child                 = this.getChildAt(i);
            MyDrawerLayout.LayoutParams lp                    = (MyDrawerLayout.LayoutParams) child.getLayoutParams();
            boolean                     isOpenedAndNotClosing = lp.openState == 1;
            boolean                     isClosedAndOpening    = lp.openState == 2;
            if (isOpenedAndNotClosing || isClosedAndOpening) {
                ss.openDrawerGravity = lp.gravity;
                break;
            }
        }

        ss.lockModeLeft = this.mLockModeLeft;
        ss.lockModeRight = this.mLockModeRight;
        ss.lockModeStart = this.mLockModeStart;
        ss.lockModeEnd = this.mLockModeEnd;
        return ss;
    }

    public void addView(View child, int index, android.view.ViewGroup.LayoutParams params) {
        super.addView(child, index, params);
        View openDrawer = this.findOpenDrawer();
        if (openDrawer == null && !this.isDrawerView(child)) {
            ViewCompat.setImportantForAccessibility(child, 1);
        } else {
            ViewCompat.setImportantForAccessibility(child, 4);
        }

        if (!CAN_HIDE_DESCENDANTS) {
            ViewCompat.setAccessibilityDelegate(child, this.mChildAccessibilityDelegate);
        }

    }

    static boolean includeChildForAccessibility(View child) {
        return ViewCompat.getImportantForAccessibility(child) != 4 && ViewCompat.getImportantForAccessibility(child) != 2;
    }

    static {
        CAN_HIDE_DESCENDANTS = Build.VERSION.SDK_INT >= 19;
        SET_DRAWER_SHADOW_FROM_ELEVATION = Build.VERSION.SDK_INT >= 21;
    }

    static final class ChildAccessibilityDelegate extends AccessibilityDelegateCompat {
        ChildAccessibilityDelegate() {
        }

        public void onInitializeAccessibilityNodeInfo(View child, AccessibilityNodeInfoCompat info) {
            super.onInitializeAccessibilityNodeInfo(child, info);
            if (!MyDrawerLayout.includeChildForAccessibility(child)) {
                info.setParent((View) null);
            }

        }
    }

    class AccessibilityDelegate extends AccessibilityDelegateCompat {
        private final Rect mTmpRect = new Rect();

        AccessibilityDelegate() {
        }

        public void onInitializeAccessibilityNodeInfo(View host, AccessibilityNodeInfoCompat info) {
            if (MyDrawerLayout.CAN_HIDE_DESCENDANTS) {
                super.onInitializeAccessibilityNodeInfo(host, info);
            } else {
                AccessibilityNodeInfoCompat superNode = AccessibilityNodeInfoCompat.obtain(info);
                super.onInitializeAccessibilityNodeInfo(host, superNode);
                info.setSource(host);
                ViewParent parent = ViewCompat.getParentForAccessibility(host);
                if (parent instanceof View) {
                    info.setParent((View) parent);
                }

                this.copyNodeInfoNoChildren(info, superNode);
                superNode.recycle();
                this.addChildrenForAccessibility(info, (ViewGroup) host);
            }

            info.setClassName(MyDrawerLayout.class.getName());
            info.setFocusable(false);
            info.setFocused(false);
            info.removeAction(AccessibilityNodeInfoCompat.AccessibilityActionCompat.ACTION_FOCUS);
            info.removeAction(AccessibilityNodeInfoCompat.AccessibilityActionCompat.ACTION_CLEAR_FOCUS);
        }

        public void onInitializeAccessibilityEvent(View host, AccessibilityEvent event) {
            super.onInitializeAccessibilityEvent(host, event);
            event.setClassName(MyDrawerLayout.class.getName());
        }

        public boolean dispatchPopulateAccessibilityEvent(View host, AccessibilityEvent event) {
            if (event.getEventType() == 32) {
                List<CharSequence> eventText     = event.getText();
                View               visibleDrawer = MyDrawerLayout.this.findVisibleDrawer();
                if (visibleDrawer != null) {
                    int          edgeGravity = MyDrawerLayout.this.getDrawerViewAbsoluteGravity(visibleDrawer);
                    CharSequence title       = MyDrawerLayout.this.getDrawerTitle(edgeGravity);
                    if (title != null) {
                        eventText.add(title);
                    }
                }

                return true;
            } else {
                return super.dispatchPopulateAccessibilityEvent(host, event);
            }
        }

        public boolean onRequestSendAccessibilityEvent(ViewGroup host, View child, AccessibilityEvent event) {
            return !MyDrawerLayout.CAN_HIDE_DESCENDANTS && !MyDrawerLayout.includeChildForAccessibility(child) ? false : super.onRequestSendAccessibilityEvent(host, child, event);
        }

        private void addChildrenForAccessibility(AccessibilityNodeInfoCompat info, ViewGroup v) {
            int childCount = v.getChildCount();

            for (int i = 0; i < childCount; ++i) {
                View child = v.getChildAt(i);
                if (MyDrawerLayout.includeChildForAccessibility(child)) {
                    info.addChild(child);
                }
            }

        }

        private void copyNodeInfoNoChildren(AccessibilityNodeInfoCompat dest, AccessibilityNodeInfoCompat src) {
            Rect rect = this.mTmpRect;
            src.getBoundsInParent(rect);
            dest.setBoundsInParent(rect);
            src.getBoundsInScreen(rect);
            dest.setBoundsInScreen(rect);
            dest.setVisibleToUser(src.isVisibleToUser());
            dest.setPackageName(src.getPackageName());
            dest.setClassName(src.getClassName());
            dest.setContentDescription(src.getContentDescription());
            dest.setEnabled(src.isEnabled());
            dest.setClickable(src.isClickable());
            dest.setFocusable(src.isFocusable());
            dest.setFocused(src.isFocused());
            dest.setAccessibilityFocused(src.isAccessibilityFocused());
            dest.setSelected(src.isSelected());
            dest.setLongClickable(src.isLongClickable());
            dest.addAction(src.getActions());
        }
    }

    public static class LayoutParams extends MarginLayoutParams {
        private static final int FLAG_IS_OPENED  = 1;
        private static final int FLAG_IS_OPENING = 2;
        private static final int FLAG_IS_CLOSING = 4;
        public               int gravity;
        float   onScreen;
        boolean isPeeking;
        int     openState;

        public LayoutParams(@NonNull Context c, @Nullable AttributeSet attrs) {
            super(c, attrs);
            this.gravity = 0;
            TypedArray a = c.obtainStyledAttributes(attrs, MyDrawerLayout.LAYOUT_ATTRS);
            this.gravity = a.getInt(0, 0);
            a.recycle();
        }

        public LayoutParams(int width, int height) {
            super(width, height);
            this.gravity = 0;
        }

        public LayoutParams(int width, int height, int gravity) {
            this(width, height);
            this.gravity = gravity;
        }

        public LayoutParams(@NonNull MyDrawerLayout.LayoutParams source) {
            super(source);
            this.gravity = 0;
            this.gravity = source.gravity;
        }

        public LayoutParams(@NonNull android.view.ViewGroup.LayoutParams source) {
            super(source);
            this.gravity = 0;
        }

        public LayoutParams(@NonNull MarginLayoutParams source) {
            super(source);
            this.gravity = 0;
        }
    }

    private class ViewDragCallback extends ViewDragHelper.Callback {
        private final int            mAbsGravity;
        private       ViewDragHelper mDragger;
        private final Runnable       mPeekRunnable = new Runnable() {
            public void run() {
                MyDrawerLayout.ViewDragCallback.this.peekDrawer();
            }
        };

        ViewDragCallback(int gravity) {
            this.mAbsGravity = gravity;
        }

        public void setDragger(ViewDragHelper dragger) {
            this.mDragger = dragger;
        }

        public void removeCallbacks() {
            MyDrawerLayout.this.removeCallbacks(this.mPeekRunnable);
        }

        public boolean tryCaptureView(View child, int pointerId) {
            return MyDrawerLayout.this.isDrawerView(child) && MyDrawerLayout.this.checkDrawerViewAbsoluteGravity(child, this.mAbsGravity) && MyDrawerLayout.this.getDrawerLockMode(child) == 0;
        }

        public void onViewDragStateChanged(int state) {
            MyDrawerLayout.this.updateDrawerState(this.mAbsGravity, state, this.mDragger.getCapturedView());
        }

        public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
            int   childWidth = changedView.getWidth();
            float offset;
            if (MyDrawerLayout.this.checkDrawerViewAbsoluteGravity(changedView, 3)) {
                offset = (float) (childWidth + left) / (float) childWidth;
            } else {
                int width = MyDrawerLayout.this.getWidth();
                offset = (float) (width - left) / (float) childWidth;
            }

            MyDrawerLayout.this.setDrawerViewOffset(changedView, offset);
            changedView.setVisibility(offset == 0.0F ? 4 : 0);
            MyDrawerLayout.this.invalidate();
        }

        public void onViewCaptured(View capturedChild, int activePointerId) {
            MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) capturedChild.getLayoutParams();
            lp.isPeeking = false;
            this.closeOtherDrawer();
        }

        private void closeOtherDrawer() {
            int  otherGrav = this.mAbsGravity == 3 ? 5 : 3;
            View toClose   = MyDrawerLayout.this.findDrawerWithGravity(otherGrav);
            if (toClose != null) {
                MyDrawerLayout.this.closeDrawer(toClose);
            }

        }

        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            float offset     = MyDrawerLayout.this.getDrawerViewOffset(releasedChild);
            int   childWidth = releasedChild.getWidth();
            int   left;
            if (MyDrawerLayout.this.checkDrawerViewAbsoluteGravity(releasedChild, 3)) {
                left = xvel <= 0.0F && (xvel != 0.0F || offset <= 0.5F) ? -childWidth : 0;
            } else {
                int width = MyDrawerLayout.this.getWidth();
                left = xvel >= 0.0F && (xvel != 0.0F || offset <= 0.5F) ? width : width - childWidth;
            }

            this.mDragger.settleCapturedViewAt(left, releasedChild.getTop());
            MyDrawerLayout.this.invalidate();
        }

//        public void onEdgeTouched(int edgeFlags, int pointerId) {
//            MyDrawerLayout.this.postDelayed(this.mPeekRunnable, 160L);
//        }

        void peekDrawer() {
            int     peekDistance = this.mDragger.getEdgeSize();
            boolean leftEdge     = this.mAbsGravity == 3;
            View    toCapture;
            int     childLeft;
            if (leftEdge) {
                toCapture = MyDrawerLayout.this.findDrawerWithGravity(3);
                childLeft = (toCapture != null ? -toCapture.getWidth() : 0) + peekDistance;
            } else {
                toCapture = MyDrawerLayout.this.findDrawerWithGravity(5);
                childLeft = MyDrawerLayout.this.getWidth() - peekDistance;
            }

            if (toCapture != null && (leftEdge && toCapture.getLeft() < childLeft || !leftEdge && toCapture.getLeft() > childLeft) && MyDrawerLayout.this.getDrawerLockMode(toCapture) == 0) {
                MyDrawerLayout.LayoutParams lp = (MyDrawerLayout.LayoutParams) toCapture.getLayoutParams();
                this.mDragger.smoothSlideViewTo(toCapture, childLeft, toCapture.getTop());
                lp.isPeeking = true;
                MyDrawerLayout.this.invalidate();
                this.closeOtherDrawer();
                MyDrawerLayout.this.cancelChildViewTouch();
            }

        }

        public boolean onEdgeLock(int edgeFlags) {
            return false;
        }

        public void onEdgeDragStarted(int edgeFlags, int pointerId) {
            View toCapture;
            if ((edgeFlags & 1) == 1) {
                toCapture = MyDrawerLayout.this.findDrawerWithGravity(3);
            } else {
                toCapture = MyDrawerLayout.this.findDrawerWithGravity(5);
            }

            if (toCapture != null && MyDrawerLayout.this.getDrawerLockMode(toCapture) == 0) {
                this.mDragger.captureChildView(toCapture, pointerId);
            }

        }

        public int getViewHorizontalDragRange(View child) {
            return MyDrawerLayout.this.isDrawerView(child) ? child.getWidth() : 0;
        }

        public int clampViewPositionHorizontal(View child, int left, int dx) {
            if (MyDrawerLayout.this.checkDrawerViewAbsoluteGravity(child, 3)) {
                return Math.max(-child.getWidth(), Math.min(left, 0));
            } else {
                int width = MyDrawerLayout.this.getWidth();
                return Math.max(width - child.getWidth(), Math.min(left, width));
            }
        }

        public int clampViewPositionVertical(View child, int top, int dy) {
            return child.getTop();
        }
    }

    protected static class SavedState extends AbsSavedState {
        int openDrawerGravity = 0;
        int lockModeLeft;
        int lockModeRight;
        int lockModeStart;
        int lockModeEnd;
        public static final Creator<MyDrawerLayout.SavedState> CREATOR = new ClassLoaderCreator<MyDrawerLayout.SavedState>() {
            public MyDrawerLayout.SavedState createFromParcel(Parcel in, ClassLoader loader) {
                return new MyDrawerLayout.SavedState(in, loader);
            }

            public MyDrawerLayout.SavedState createFromParcel(Parcel in) {
                return new MyDrawerLayout.SavedState(in, (ClassLoader) null);
            }

            public MyDrawerLayout.SavedState[] newArray(int size) {
                return new MyDrawerLayout.SavedState[size];
            }
        };

        public SavedState(@NonNull Parcel in, @Nullable ClassLoader loader) {
            super(in, loader);
            this.openDrawerGravity = in.readInt();
            this.lockModeLeft = in.readInt();
            this.lockModeRight = in.readInt();
            this.lockModeStart = in.readInt();
            this.lockModeEnd = in.readInt();
        }

        public SavedState(@NonNull Parcelable superState) {
            super(superState);
        }

        public void writeToParcel(Parcel dest, int flags) {
            super.writeToParcel(dest, flags);
            dest.writeInt(this.openDrawerGravity);
            dest.writeInt(this.lockModeLeft);
            dest.writeInt(this.lockModeRight);
            dest.writeInt(this.lockModeStart);
            dest.writeInt(this.lockModeEnd);
        }
    }

    public abstract static class SimpleDrawerListener implements MyDrawerLayout.DrawerListener {
        public SimpleDrawerListener() {
        }

        public void onDrawerSlide(View drawerView, float slideOffset) {
        }

        public void onDrawerOpened(View drawerView) {
        }

        public void onDrawerClosed(View drawerView) {
        }

        public void onDrawerStateChanged(int newState) {
        }
    }

    public interface DrawerListener {
        void onDrawerSlide(@NonNull View var1, float var2);

        void onDrawerOpened(@NonNull View var1);

        void onDrawerClosed(@NonNull View var1);

        void onDrawerStateChanged(int var1);
    }
}

```

反射修改edgeSize

```java
public void setCustomLeftEdgeSize(@NonNull MyDrawerLayout drawerLayout, float displayWidthPercentage) {
        try {
            // find ViewDragHelper and set it accessible
            Field leftDraggerField = drawerLayout.getClass().getDeclaredField("mLeftDragger");
            if (leftDraggerField == null) {
                return;
            }
            leftDraggerField.setAccessible(true);
            ViewDragHelper leftDragger = (ViewDragHelper) leftDraggerField.get(drawerLayout);
            // find edgesize and set is accessible
            Field edgeSizeField = leftDragger.getClass().getDeclaredField("mEdgeSize");
            edgeSizeField.setAccessible(true);
            int edgeSize = edgeSizeField.getInt(leftDragger);
            // set new edgesize

            DisplayMetrics displayMetrics = new DisplayMetrics();
            WindowManager  wm             = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
            wm.getDefaultDisplay().getRealMetrics(displayMetrics);
            int widthPixels = displayMetrics.widthPixels;
            edgeSizeField.setInt(leftDragger, Math.max(edgeSize, (int) (widthPixels * displayWidthPercentage)));
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
```

注意：由于用了反射，如果你使用了代码混淆，一定要keep MyDrawerLayout，不然会失效的

```java
#MyDrawerLayout反射
-keepclasseswithmembernames class [packagespace].MyDrawerLayout{
    <fields>;
}
```


