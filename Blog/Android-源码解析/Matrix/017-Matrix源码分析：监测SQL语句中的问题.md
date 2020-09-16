---
title: 017-Matrix源码分析：监测SQL语句中的问题
date: 2020-9-2
categories: Matrix
---

该库可以在 APP 运行时对 sql 语句、执行序列、表信息等进行分析检测。其主要代码在 `matrix-sqlite-lint-android-sdk`下。虽然名带 “lint ” ，但并不是代码的静态检查。

这里主要介绍一下 SQLiteLint 的思路，因为需要检测出来哪些 SQL 是可以优化的，所以你就必须要了解 SQL 语句，而我对 SQL 语句的熟悉程度还达不到这一点，所以就很难看懂里面的一些算法。故算法部分就不深入分析了，有兴趣可以看源码以及官方文档。



### 检测流程

![img](https://mmbiz.qpic.cn/mmbiz_png/csvJ6rH9McsI9ibGiaIgT8sOFicBw5GHjWelfLAyI2F8NrWKo3Hp3tpAcMvSzONH3s4SNVxgiaUvNxJmob1H2gTiaWQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，基本流程与 IO 的检测是差不多的，实际上确实如此，源码部分也有很多类似的地方。

我们看下里面的几个流程：

1. 收集 APP 运行时的 sql 执行信息
     包括执行语句、创建的表信息等。其中表相关信息可以通过 pragma 命令得到。对于执行语句，有两种情况：
     a）DB 框架提供了回调接口。比如微信使用的是 WCDB ，很容易就可以通过MMDataBase.setSQLiteTrace 注册回调拿到这些信息。
     b） 若使用 Android 默认的 DB 框架，SQLiteLint 提供了一种无侵入的获取到执行的sql语句及耗时等信息的方式。通过hook的技巧，向 SQLite3 C 层的  api sqlite3_profile 方法注册回调，也能拿到分析所需的信息，从而无需开发者额外的打点统计代码。

2. 预处理
   包括生成对应的 sql 语法树，生成不带实参的 sql ，判断是否 select* 语句等，为后面的分析做准备。预处理和后面的算法调度都在一个单独的处理线程。

3. 调度具体检测算法执行
   checker 就是各种检测算法，也支持扩展。并且检测算法都是以 C++ 实现，方便支持多平台。而调度的时机包括：最近未分析 sql 语句调度，抽样调度，初始化调度，每条 sql 语句调度。

4. 发布问题
   上报问题或者弹框提示。

可以看到重点在第 3 步，而这一步本文暂不分析。第2步也只简单的说一下。

### 收集App运行时的 SQL 执行信息

该库提供了两种方式，一种是 hook，一种是通过回调接口。至于使用哪种，需要我们执行设置：

```java
    private static SQLiteLintConfig initSQLiteLintConfig() {
        try {
            /**
             * HOOK模式下，SQLiteLint会自己去获取所有已执行的sql语句及其耗时(by hooking sqlite3_profile)
             * @see 而另一个模式：SQLiteLint.SqlExecutionCallbackMode.CUSTOM_NOTIFY , 则需要调用 {@link SQLiteLint#notifySqlExecution(String, String, int)}来通知
             * SQLiteLint 需要分析的、已执行的sql语句及其耗时
             * @see TestSQLiteLintActivity#doTest()
             */
            return new SQLiteLintConfig(SQLiteLint.SqlExecutionCallbackMode.HOOK);
        } catch (Throwable t) {
            return new SQLiteLintConfig(SQLiteLint.SqlExecutionCallbackMode.HOOK);
        }
    }
```

使用 hook 在高版本可能会有兼容问题，如果遇到后不要慌。

#### hook

我们看看 hook 实现的相关代码，可以回想一下 IO 的 hook，想必是差不多的。

```c++
xhook_register(".*/libandroid_runtime\\.so$", "sqlite3_profile", (void*)hooked_sqlite3_profile, (void**)&original_sqlite3_profile);
```

hook 的是 sqlite3_profile 函数。

```c++
    void* hooked_sqlite3_profile(sqlite3* db, void(*xProfile)(void*, const char*, sqlite_uint64), void* p) {
        LOGI("hooked_sqlite3_profile call");
        return original_sqlite3_profile(db, SQLiteLintSqlite3ProfileCallback, p);
    }
```

hook 函数也没干啥，就是直接调用了原函数，不过传递了一个 callback 进去。

```c++
    static void SQLiteLintSqlite3ProfileCallback(void *data, const char *sql, sqlite_uint64 tm) {
        if (kStop) {
            return;
        }

        JNIEnv*env = getJNIEnv();
        if (env != nullptr) {
            SQLiteConnection* connection = static_cast<SQLiteConnection*>(data);
            jstring extInfo = (jstring)env->CallStaticObjectMethod(kUtilClass, kMethodIDGetThrowableStack);
            char *ext_info = jstringToChars(env, extInfo);

            NotifySqlExecution(connection->label, sql, tm/1000000, ext_info);

            free(ext_info);
        } else {
            LOGW("SQLiteLintSqlite3ProfileCallback env null");
        }
    }
```

该回调会在 sql 执行后触发，这样我们就可以拿到该 sql 的相关信息。然后调用 NotifySqlExecution 通知 sql 执行了，实际上就是放入队列里面。

里面会有一个生产者-消费者模型，与 IO 是一样的，所以这里扮演的是生产者角色。

#### 手动通知

当我们不使用 hook 的时候，就需要开发者自己在 sql 执行后手动调用了。

```java
    public void notifySqlExecution(String concernedDbPath, String sql, int timeCost) {
        if (!isPluginStarted()) {
            SLog.i(TAG, "notifySqlExecution isPluginStarted not");
            return;
        }

        SQLiteLint.notifySqlExecution(concernedDbPath, sql, timeCost);
    }
```

调用这行代码之后，最终也会进入到 NotifySqlExecution 这行语句中，与 hook 方式是一样的，这样有点类似打点。

我们看看 NotifySqlExecution 最终会调用到如下代码：

```c++
    void Lint::NotifySqlExecution(const char *sql, const long time_cost, const char* ext_info) {
        if (sql == nullptr){
            sError("Lint::NotifySqlExecution sql NULL");
            return;
        }

        if (env_.IsReserveSql(sql)) {
            sDebug("Lint::NotifySqlExecution a reserved sql");
            return;
        }

        SqlInfo *sql_info = new SqlInfo();
        sql_info->sql_ = sql;
		sql_info->execution_time_ = GetSysTimeMillisecond();
        sql_info->ext_info_ = ext_info;
        sql_info->time_cost_ = time_cost;
        sql_info->is_in_main_thread_ = IsInMainThread();

        std::unique_lock<std::mutex> lock(queue_mutex_);
        queue_.push_back(std::unique_ptr<SqlInfo>(sql_info));
        queue_cv_.notify_one();
        lock.unlock();
    }
```

可以看到，这里给 sql_info 赋值了很多字段，就是收集相关信息了，然后就放到 queue_ 里面，然后通知消费者线程。

### 预处理

sql 信心收集完成之后，需要进行处理，我们看源码，直接看消费者线程的逻辑。

```c++
            ...
                
			// 从队列中取一个
            int ret = TakeSqlInfo(sql_info);

			...
                
            // 记录sql数量，用于抽样
            env_.IncSqlCnt();
            // 将 sql 语句进行预处理，去除前后空格，全转小写
            PreProcessSqlString(sql_info->sql_);

			...
                
            // 真正的预处理逻辑
            if (!PreProcessSqlInfo(sql_info.get())) {
                sWarn("Lint::Check PreProcessSqlInfo failed");
                env_.AddToSqlHistory(*sql_info);
                sql_info = nullptr;
                continue;
            }

			...

            // 这里是抽样检测
            ScheduleCheckers(CheckScene::kSample, *sql_info, published_issues);

			...
			
            if (!checked_sql_cache_.Get(wildcard_sql, checked)) {
                // 这里是避免重复检测，与上面的抽象可能会重复
                ScheduleCheckers(CheckScene::kUncheckedSql, *sql_info, published_issues);
                checked_sql_cache_.Put(wildcard_sql, true);
            } 
```

看看 ScheduleCheckers ：

```c++
    void Lint::ScheduleCheckers(const CheckScene check_scene, const SqlInfo& sql_info, std::vector<Issue> *published_issues) {
        // 检测场景不符，就直接返回
        // 每个 Checker 有自己的检测场景
        std::map<CheckScene, std::vector<Checker*>>::iterator it = checkers_.find(check_scene);
        if (it == checkers_.end()) {
            return;
        }

        std::vector<Checker*> scene_checkers = it->second;
        size_t scene_checkers_cnt = scene_checkers.size();

        for (size_t i=0;i < scene_checkers_cnt;i++) {

            Checker* checker = scene_checkers[i];
            // 如果是抽样检测（kSample），则需要满足隔 30 个取一个
            if (check_scene != CheckScene::kSample
                || (env_.GetSqlCnt() % checker->GetSqlCntToSample() == 0)) {
                checker->Check(env_, sql_info, published_issues);
            }

        }
    }
```

主要就是调用了各个 Checker 的 Check 函数。Checker 有很多，里面具体做了啥，这里就不分析了，涉及到语法语义。

### 上报

上报的逻辑与IO差不多。具体逻辑在

> sqlitelint::OnIssuePublish



### 参考文档

https://mp.weixin.qq.com/s/laUgOmAcMiZIOfM2sWrQgw

