---
title: 0010-与 RxJava 配合使用
index_img: /cover/11.jpg
banner_img: /cover/top.jpg
date: 2019-9-11
tags: Android源码解析-Tangram
categories: Tangram
---

想要使用 RxJava，需要先开启：

```java
engine.setSupportRx(true);
```

然后还要提供依赖包，具体可以参考[官方文档](http://tangram.pingguohe.net/docs/android/reactive-tangram).

## 响应式流中数据结构的定义

在响应式流中，传递的都是数据单个对象，而传统的命令式接口里，一个操作往往有多个类型的参数，JAVA 里没有像元组那样的结构用来组合一系列对象，用 `Object[]` 类型的对象也难以理解维护，因此，针对这种情况，Tangram 将相关接口的参数封装成一系列 `TangramOp1`、`TangramOp2`、`TangramOp3` 对象，分别包含一个、两个、三个参数，用来在响应式流中传递信息，也方便原有接口的对接。具体的定义在包 `com.tmall.wireless.tangram.op` 下，包含了插入、更新、异步加载等接口的操作定义。下文中碰到的接口里包含的 `ClickExposureCellOp`、`LoadGroupOp` 等都是在这个背景下定义的结构。

这一段还是应该好好阅读几遍的，在我刚开始接触 Rx 的时候，就不太明白为啥会设计出这样的接口：

> io.reactivex.functions

在 functions 包下，还有 Function3，Function4 这样的东西。有了上面这段话，初学者应该就会比较好理解一些。



关于 Rx 的配合使用，官方文档也说的差不多了，再深入就是 Rx 相关的东西，这里就不说了。这里只说一下 demo 的 RxTangramActivity 显示的效果会是一个一个卡片加载出来的。想要做出依次加载的效果，有哪些思路呢？

- 每次add一个，控制好add时机
- 使用动画

使用动画的话，会有一个问题，就是上下滑出滑入的时候都会执行动画，不好搞。看看 Tangram 是如何做的吧：

```java
        Disposable dsp8 = Observable.create(new ObservableOnSubscribe<JSONArray>() {
            @Override
            public void subscribe(ObservableEmitter<JSONArray> emitter) throws Exception {
                String json = new String(getAssertsFile(getApplicationContext(), "data.json"));
                JSONArray data = null;
                try {
                    data = new JSONArray(json);
                } catch (JSONException e) {
                    e.printStackTrace();
                }
                emitter.onNext(data);
                emitter.onComplete();
            }
        }).flatMap(new Function<JSONArray, ObservableSource<JSONObject>>() {
            @Override
            public ObservableSource<JSONObject> apply(JSONArray jsonArray) throws Exception {
                return JSONArrayObservable.fromJsonArray(jsonArray);
            }
        }).map(new Function<JSONObject, ParseSingleGroupOp>() {
            @Override
            public ParseSingleGroupOp apply(JSONObject jsonObject) throws Exception {
                return new ParseSingleGroupOp(jsonObject, engine);
            }
        }).compose(engine.getSingleGroupTransformer())
        .filter(new Predicate<Card>() {
            @Override
            public boolean test(Card card) throws Exception {
                return card.isValid();
            }
        }).map(new Function<Card, AppendGroupOp>() {
            @Override
            public AppendGroupOp apply(Card card) throws Exception {
                Thread.sleep(300);
                return new AppendGroupOp(card);
            }
        }).subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(engine.asAppendGroupConsumer(), new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                throwable.printStackTrace();
            }
        });
```

里面的逻辑还是挺清晰的：

- 先从 json 模拟数据文件里面读取出数据，将数据转成 JSONArray

- 将 JsonArray 转成 Observerable\<JSONObject>，具体做法就是将 JSONArray 拆成一个一个的 JSONObject 数组，然后发送出去

  > com.alibaba.android.rx.JSONArrayObservable.FromJsonArrayDisposable#run

  ```java
          void run() {
              JSONArray a = array;
              int n = a.length();
  
              for (int i = 0; i < n && !isDisposed(); i++) {
                  T value = (T) a.opt(i);
                  if (value == null) {
                      actual.onError(new NullPointerException("The " + i + "th element is null"));
                      return;
                  }
                  actual.onNext(value);
              }
              if (!isDisposed()) {
                  actual.onComplete();
              }
          }
  ```

- 下游接收到 JSONObject 后解析成 Card

- 校验 Card 是否有效

- 将 Card 封装成一个 AppendGroupOp 对象

- 线程切换

- 下游拿到 AppendGroupOp 对象，执行添加 Card 操作

这样就完成了，那么现在，我们应该就知道了，将 json 数据解析成了 Card 需要耗时一定的时间，所以就有了一个间隔效果，而且调用 notifyItemRangeInserted 还会有有一个自带的动画效果。要是不相信的话，可以自己去写个 demo 跑一下。

