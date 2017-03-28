<b>分析android ui为什么在子线程中更新会崩溃</b><br>
很多初学者肯定有这样一个经验，在Activity的一个子线程中更新UI，发现会报错。很多人知道这个错误，但却不知道是什么原因引起的。今天我们来分析一下是什么原因引起的。我今天以textview更新ui为例。
首先看代码
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.generalwei.mytest.ThreadActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="hello"/>

    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:id="@+id/btn_change"
        android:text="更换UI"/>

</RelativeLayout>
```
```java
public class ThreadActivity extends AppCompatActivity {

    private android.widget.TextView tv;
    private android.widget.Button btnchange;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_thread);
        this.btnchange = (Button) findViewById(R.id.btn_change);
        this.tv = (TextView) findViewById(R.id.tv);
        btnchange.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        tv.setText(new Date().getTime()+"");
                    }
                }).start();

            }
        });
    }
}
```
运行app，点击btnchange按钮，发现app崩溃。查看日志,发现这样一个错误：

![](http://upload-images.jianshu.io/upload_images/2362201-5e675277792ce1bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们来查看一下tv.setText()方法的源码。
```java
public final void setText(CharSequence text) {
        setText(text, mBufferType);
}

 public void setText(CharSequence text, BufferType type) {
        setText(text, type, true, 0);
        if (mCharWrapper != null) {
            mCharWrapper.mChars = null;
        }
    }


 private void setText(CharSequence text, BufferType type,
                         boolean notifyBefore, int oldlen) {
        ...
        if (mLayout != null) {
            checkForRelayout();
        }
        ...
    }
```
这里我们还是没有看见什么原因引起的，继续追查checkForRelayout()方法。
```
 private void checkForRelayout() {
       ... 
       invalidate();
       ...
}
```
查看invalidate()方法,我们会看见这样一个注释:
```
  /**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
  public void invalidate() {
        invalidate(true);
    }
```
意思是说如果view是可见的，这个方法会刷新view。但是必须发生在ui线程上。看到这边我们发现我们的追踪是正确。那么接着看，为什么一定要在ui线程上更新ui。
```
  void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }


void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
            ...     
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }  
        ...
    }
```
ViewParent 是一个接口,ViewRootImpl是它的实现类,那么我们继续追查，代码如下:
```java
@Override
    public void invalidateChild(View child, Rect dirty) {
        invalidateChildInParent(null, dirty);
    }
    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        checkThread();
      ...
    }
```
你会发现一个检查线程的方法，那么查看方法checkThread()。
```java
 void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```
查看代码后发现有一个mThread线程，它是会是UI线程吗，我们就查看Thread的来源。
```
    public ViewRootImpl(Context context, Display display) {
      ...
        mThread = Thread.currentThread();
     ...
```
#####为什么在onResume之前非UI线程也能更新UI
发现mThread是在ViewRootImpl创建的时候赋值，这个的线程一定是UI线程。所以当前线程不是UI线程的时候会抛异常。
但是有时候在也能在非UI线程中更新，后来我们发现在onResume之前用非UI线程更新能UI，而onResume之后就不行了。这是因为onResume之前还没有创建ViewRootImpl这个类，ActivityThread类中有一个handleResumeActivity方法，这个方法是用来回调Activity的onResume方法，具体的看如下代码:
```java
 final void handleResumeActivity(IBinder token,boolean clearHide, boolean   isForward, boolean reallyResume, int seq, String reason) {
    r = performResumeActivity(token, clearHide, reason);
    if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                ...
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    // Normally the ViewRoot sets up callbacks with the Activity
                    // in addView->ViewRootImpl#setView. If we are instead reusing
                    // the decor view we have to notify the view root that the
                    // callbacks may have changed.
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
            ...
            // Tell the activity manager we have resumed.
           if (reallyResume) {
                try {
                    ActivityManagerNative.getDefault().activityResumed(token);
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            }

        } else {
            // If an exception was thrown when trying to resume, then
            // just end this activity.
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
    }
```
你可以看到这样一个注释 // Tell the activity manager we have resumed.这个方法是可以回调Activity的onResume。具体怎么回调这里就不解释，我会在下几篇博客中去分析Activity的生命周期。
在代码中我们可以看见一个WindowManager类，这个类是用来控制窗口显示的，而它的addView是用用来添加窗口。WindowManagerImpl是WindowManager的实现类，WindowManagerImpl的addView方法代码如下:
```
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```
mGlobal是WindowManagerGlobal的对象，继续看mGlobal.addView的代码:
```
 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }
       ...
    }
```
你可以看做ViewRootImpl的初始化时在这里进行的，这就是为什么在onResume之前可以更新UI了。
#####为什么要这么设计呢？
因为所有的UI控件都是非线程安全的，如果在非UI线程更新UI会造成UI混乱。所以一般我们会在Handler中更新UI。
如有写的不当之处，请多指教。
