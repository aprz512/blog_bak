---
title: 子线程更新UI的问题
index_img: /cover/21.jpg
banner_img: /cover/top.jpg
date: 2020-12-21
categories: View
---

> 这篇文章，只能算是一个开头吧，后面准备研究一下，facebook 的那个  [Components for Android](https://engineering.fb.com/android/components-for-android-a-declarative-framework-for-efficient-uis/) 里面提到的基于组件的细粒度回收复用机制。因为整体来说，这都属于 RecycleView 的优化，今天介绍的也是其中的一部分，或者说比较通用的一部分。
>
> ----
>
> 了解了之后才发现，与 tangram-view 是一样的思想。但是我想的细粒度应该是复杂卡片拆解那样的，后来发现这应该属于自定义 LayoutManager 的内容，然后我就又想起了 vLayout 这个东西。绕来绕去，原来都是我以前都看过了的......

通常情况下，页面卡顿，都是发生在较为复杂的页面。比如，一个非常大的 xml，它加载起来就会比较耗时，这样，打开该页面的时候，就会造成卡顿。

### 异步加载 xml 引起的思考

一种解决的方式就是使用 [AsyncLayoutInflater](https://developer.android.com/reference/androidx/asynclayoutinflater/view/AsyncLayoutInflater) 这个类来做布局的预加载（在子线程中）。

那么，有趣的问题来了：

> 我们刚刚入门 Android 时，便知道「只能在主线程更新 UI」的规定。那为啥可以使用子线程来加载布局呢？

要解答这个问题，我们探究一下，为啥说只能在主线程更新UI。其实是因为，我们在子线程更新 UI 的时候，会报一个异常：

> android.view.ViewRootImpl#checkThread

```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

我们先不考虑为啥这里它要检查一下，其实也是因为绝大部分 GUI 系统都是只允许「单个线程对某块区域做绘制操作的」，如果允许多个线程对同一块区域做绘制的话，很有可能导致同步问题。除非加锁，然而加锁极有可能导致死锁，性能等问题。

那么，我们能不能绕过或者是跳过这个检查逻辑呢？我们看下面这个例子：

```java
public class T extends AppCompatActivity {

    private ImageView iv;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        iv = new ImageView(this);
        iv.setImageResource(R.drawable.ic_launcher_background);
        setContentView(iv);
        new Thread(() -> iv.setImageResource(R.drawable.ic_launcher_foreground)).start();
    }
}

```

当我们，打开这个页面的时候，程序并不会出问题，完美运行。这是为什么呢？

这是因为我们的Thread执行的时候，ViewRootImpl还没有对view tree的根节点DecorView执行performTraversals，view tree里的所有View都没有被赋值mAttachInfo。

在onCreate完成时，Activity并没有完成初始化view tree。view tree的初始化是从ViewRootImpl执行performTraversals开始，这个过程会对view tree进行从根节点DecorView开始的遍历，对所有视图完成初始化，初始化包括视图的大小布局，以及AttachInfo，ViewParent等域的初始化。

执行ImageView.setImageResource，调用的过程是

```java
ImageView.setImageResource 
-> View.invalidate 
-> View.invalidateInternal 
-> ViewGroup.invalidateChild
-> ViewParent.invalidateChildInParent //这里会不断Loop去取上一个结点的mParent
-> ViewRootImpl.invalidateChildInParent //DecorView的mParent是ViewRootImpl
-> ViewRootImpl.checkThread //在这里执行checkThread，如果非UI线程则抛出异常
```


但是在Thread执行setImageResource时，此时Activity还在初始化，ViewRoot没有初始化整个view tree，ImageView的mAttachInfo是空的（mAttachInfo包含了Window的token等Binder）。而View.invalidateInternal调用ViewGroup.invalidateChild要判断是否存在ViewParent和AttachInfo：

```java
final AttachInfo ai = mAttachInfo;
final ViewParent p = mParent;
if (p != null && ai != null && l < r && t < b) {
    //....
    p.invalidateChild(this, damage);
}
```


也就是说，**此时因为不存在ViewParent，invalidate的过程中止而没有完全执行，也即没有发生checkThread。**

所以说，在 View tree 初始化完成之前，我们可以在子线程里面做很多事，比如去加载布局。

说的更加直白一点，checkThread 是 ViewRootImpl 做的，到了它这里，布局早就已经被加载出来了，也就是说，它**管不到布局是哪个线程加载出来**的。

这个问题，我们搞定了，那么就再深入一点，我们看 checkThread 方法：

```java
    void checkThread() {
        // 这里是不是有点奇怪
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

看上面的 if 判断条件，它判断的居然不是主线程，而是当前线程，这说明了什么呢？这是不是意味着，如果我在子线程里面去创建布局，那么是不是就可以在子线程里面去更新布局了呢？

### 子线程更新UI - Dialog

试试吧！！！

我们先找到 ViewRootImpl 创建的位置：

```
->android.app.ActivityThread#handleResumeActivity
-->android.view.WindowManagerGlobal#addView
```

在 addView 里面创建了 ViewRootImpl 对象，由于，handleResumeActivity 是工作在主线程，所以，addView 也是在主线程，那就暂时放弃测试 activity。

我们看看 dialog，它的 window 的 addView 方法是在 show 里面调用的，所以，我们可以拿它来测试：

```java
public class MainActivity extends AppCompatActivity {

    private HandlerThread handlerThread = new HandlerThread("ht");
    private Handler handler;
    private EditText editText;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        handlerThread.start();
        handler = new Handler(handlerThread.getLooper());

        // 子线程去展示一个弹窗
        handler.post(new Runnable() {
            @Override
            public void run() {
                Log.i("Thread", Thread.currentThread().getName());
                AlertDialog.Builder customizeDialog =
                        new AlertDialog.Builder(MainActivity.this);
                final View dialogView = LayoutInflater.from(MainActivity.this)
                        .inflate(R.layout.dialog_customize, null);
                // 获取EditView中的输入内容
                editText = (EditText) dialogView.findViewById(R.id.edit_text);
                customizeDialog.setTitle("我是一个自定义Dialog");
                customizeDialog.setView(dialogView);
                customizeDialog.setPositiveButton("确定",
                        new DialogInterface.OnClickListener() {
                            @Override
                            public void onClick(DialogInterface dialog, int which) {
                                Toast.makeText(MainActivity.this,
                                        editText.getText().toString(),
                                        Toast.LENGTH_SHORT).show();
                            }
                        });
                customizeDialog.show();
            }
        });

        // 在子线程里面去更新弹窗里面的内容
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                Log.i("Thread", Thread.currentThread().getName());
                editText.setText("hello from handler thread: " + SystemClock.currentThreadTimeMillis());
            }
        }, 3000);

    }

}
```

实际上，完美运行！！！



### 子线程更新UI - Activity

想要在子线程更新 UI，那么就需要搞定里面的 checkThread 这个方法。最粗暴的方式，就是使用反射替换掉里面的 mThread 值。

首先，需要找到界面对应的 ViewRootImpl，经过一番寻找之后，我是这样做的：

View 有个 getViewRootImpl 方法，但是该方法需要 mAttachInfo 这个值不为 null。那么什么时候这个值不为 null 呢？view attach 到 window上的时候。那么我们的思路就出来了：

```java
    private void hookViewRootImpl() {
        getWindow().getDecorView().addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
            @Override
            public void onViewAttachedToWindow(View v) {
                v.removeOnAttachStateChangeListener(this);
                try {
                    Method getViewRootImplMethod = v.getClass().getMethod("getViewRootImpl");
                    Object viewRootImplObject = getViewRootImplMethod.invoke(v);
                    Class<?> viewRootImplClass = viewRootImplObject.getClass();
                    Field thread = viewRootImplClass.getDeclaredField("mThread");
                    thread.setAccessible(true);
                    thread.set(viewRootImplObject, handlerThread);
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                } catch (NoSuchFieldException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onViewDetachedFromWindow(View v) {
            }
        });
    }
```

这里，我们在 view 添加到 window 上的时候，将 ViewRootImpl 里面的 mThread 字段设置为子线程。

这样，我们就可以在子线程里面更新UI了。为了防止出啥意外，写一个 UiDispatcher，自动切换 handler：

```java
    static class UiDispatcher {

        Handler target;

        public UiDispatcher(final View root, Handler main, final Handler child) {
            target = main;
            root.addOnAttachStateChangeListener(new View.OnAttachStateChangeListener() {
                @Override
                public void onViewAttachedToWindow(View v) {
                    root.removeOnAttachStateChangeListener(this);
                    target = child;
                }

                @Override
                public void onViewDetachedFromWindow(View v) {
                }
            });
        }

        public void post(Runnable runnable) {
            target.post(runnable);
        }
    }
```

然后，我们使用这个类，来更新UI：

```java
        md5.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                uiDispatcher.post(new Runnable() {
                    @Override
                    public void run() {
                        text.setText("msg from " + Thread.currentThread().getName());
                    }
                });
            }
        });
```

正常运行！！！当然这个例子只是说明了，可以在子线程来更新UI。

所以，最终测试的结果是：子线程也可以更新UI，但是还是有很多限制，比如 activity 就比较麻烦，因为 activityThread 已经限制了创建 ViewRootImpl 的线程在主线程。

总的来说，只有主线程可以更新UI这句话**大体上还是对的**，但是有两种例外：

- 在 View tree 还没有创建出来的时候，是可以在子线程更新的，比如 setText 等操作，这是因为，ViewRootImpl 还没有创建出来，自然也就不会报错。而且它子所以可以更新成功，也是由于 setText 方法储存了 set 的值。
- 在子线程X里面创建了 ViewRootImpl ，自然可以在子线程X里面更新UI。