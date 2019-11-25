---
title: synchronized相关知识
date: 2018-04-23 20:25
tags: 
- 底层知识
---

首先来说下,为什么写这篇文章。今天在 `review` 代码的过程中，发现下面单例的写法：

```
+ (instancetype)shareInstance{
    static KHPickerUtil *manager = nil;
    @synchronized(manager) {
        manager = [[KHPickerUtil alloc] init];
    }
    return manager;
}
```
先不论这种方式写单例的问题，这里来讨论一下 `@synchronized`。

#### @synchronized 介绍

接触 `iOS` 开发有一段时间的开发者，相信大家都用过，`@synchronized` 是在 `Objective-C` 代码中快读创建互斥锁的一种便捷方式。该`@synchronized` 可以执行其他任何互斥锁都会执行的操作，它可以防止线程同时获取同一个锁。但是，在这种情况下，您不必直接创建互斥锁或锁定对象。相反，只需使用任何 `Objective-C` 对象作为锁定标记即可，如以下示例所示：

```
 - （void）myMethod：（id）anObj
{
    @synchronized（anObj）
    {
        //大括号之间的所有内容都由@synchronized指令保护。
    }
}
```
传递给 `@synchronized` 的对象是用于区分受保护块的唯一标识符。如果您在两个不同的线程中执行上述方法，则为 `anObj` 每个线程上的参数传递一个不同的对象，每个线程都会锁定并继续处理而不被另一个线程阻塞。但是，如果在两种情况下都传递相同的对象，则其中一个线程将首先获取锁，另一个会阻塞，直到第一个线程完成。

#### @synchronized 深入研究
首先先列出几个问题：

- `@synchronized` 是如何实现的
- `@synchronized` 是如何对传入的对象加锁的
- `@synchronized` 如果传入的对象是 `nil`，会如何处理

通过查看 `@synchronized` 的[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW3)，可以知道。`@synchronized` 块隐式地向受保护的代码添加异常处理程序。如果引发异常，该处理程序会自动释放互斥锁。这意味着为了使用 `@synchronized` ，您还必须在代码中启用 `Objective-C` 异常处理。如果您不想由隐式异常处理程序引起的额外开销，则应考虑使用锁类。

```
+ (instancetype)shareIntance
{
    static CFFileManager *manager = nil;
    @synchronized (manager) {
        manager = [[CFFileManager alloc] init];
    };
    return manager;
}
```
以上代码经过 `clang -rewrite-objc` 转换可以得到一下代码:

```
static instancetype _C_CFFileManager_shareIntance(Class self, SEL _cmd) {
    static CFFileManager *manager = __null;
    { id _rethrow = 0; id _sync_obj = (id)manager; objc_sync_enter(_sync_obj);
try {
	struct _SYNC_EXIT { _SYNC_EXIT(id arg) : sync_exit(arg) {}
	~_SYNC_EXIT() {objc_sync_exit(sync_exit);}
	id sync_exit;
	} _sync_exit(_sync_obj);

        manager = ((CFFileManager *(*)(id, SEL))(void *)objc_msgSend)((id)((CFFileManager *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("CFFileManager"), sel_registerName("alloc")), sel_registerName("init"));
    } catch (id e) {_rethrow = e;}
{ struct _FIN { _FIN(id reth) : rethrow(reth) {}
	~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
	id rethrow;
	} _fin_force_rethow(_rethrow);}
}
;
    return manager;
}
```
简化:

```
+ (instancetype)shareIntance
{
    static CFFileManager *manager = nil;
    @try {
        
        objc_sync_enter(manager);
        manager = [[CFFileManager alloc] init];
        
    } @catch (NSException *exception) {
    
    } @finally {
        
        objc_sync_exit(manager);
    }
    return manager;
}
```
从上面代码可以看出，下面研究的主要是 `objc_sync_enter` 和 `objc_sync_exit` 的实现，先来看一下 `objc_sync_enter` 和 `objc_sync_exit` 的介绍:

```
** 
 * Begin synchronizing on 'obj'.  
 * Allocates recursive pthread_mutex associated with 'obj' if needed.
 * 
 * @param obj The object to begin synchronizing on.
 * 
 * @return OBJC_SYNC_SUCCESS once lock is acquired.  
 */
OBJC_EXPORT int
objc_sync_enter(id _Nonnull obj)
    OBJC_AVAILABLE(10.3, 2.0, 9.0, 1.0, 2.0);

/** 
 * End synchronizing on 'obj'. 
 * 
 * @param obj The object to end synchronizing on.
 * 
 * @return OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
 */
OBJC_EXPORT int
objc_sync_exit(id _Nonnull obj)
    OBJC_AVAILABLE(10.3, 2.0, 9.0, 1.0, 2.0); 
```

从以上介绍可以知道,调用 `objc_sync_enter` 会为对象分配一个递归锁，在获得锁之后返回 `OBJC_SYNC_SUCCESS` ，调用 `objc_sync_exit` 之后会为对象解锁，返回  `OBJC_SYNC_SUCCESS` 或者 `OBJC_SYNC_NOT_OWNING_THREAD_ERROR`
从[文件](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-sync.mm.auto.html)你可以找到 `objc_sync_enter` 和 `objc_sync_exit` 的全部实现。

```
// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        assert(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}


// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	
    return result;
}
```
`objc_sync_enter` 首先对传入的对象做一个非空判断，如果对象是不存在( `nil `),会给对象发一个通知 `NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug`，会以通知的形式告诉对象没有进行同步，之后会调用 `objc_sync_nil` ，可以通过在 `objc_sync_nil` 打断点进行调试。如果传入的对象不是 `nil`，会执行结构体 `SyncData` 的初始化，在结构体 `SyncData` 中包含一个指向下一个 `SyncData` 的指针 `nextData`， 一个 `object` ，一个 `threadCount` ( `block` 代码块的线程数量)，`mutex` 递归互斥锁。在调用 `id2data` 获取结构体指针 `data` 的过程中，会把这个 `SyncData` 存在哈希表中，在实现 `id2data` 过程中却发现了 `spinlock_t`，这一点让人费解。如果如果 `data `不存在会触发 `assert` ，存在对对象进行加锁。
在调用 `objc_sync_exit`中，传入的对象是 `nil` ， 不做任何处理。如果非 `nil`，获取加锁的 `SyncData` 指针的 `data`，如果 指针 `data` 存在会进行解锁,解锁成功返回 `OBJC_SYNC_SUCCESS`，解锁失败和指针 `data` 不存在返回`OBJC_SYNC_NOT_OWNING_THREAD_ERROR`。

通过源码可以得出：

- `@synchronized` 中的每个非空对象，都会分配一个递归锁，并且存在哈希表中
- `@synchronized` 中不要传 `nil` ，这样会导致你的代码不是线程安全的，不会加锁，可以通过objc_sync_nil加断点来调试。

#### 结束语
如果发现问题，或者有不懂的地方，请留言。

###### 参考资料

>[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW3)
[源码地址](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-sync.mm.auto.html)









