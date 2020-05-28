---
Title: 我好像不懂 - alloc、init、dealloc
date: 2020-05-28 23:00
tags:
- 基础知识
---



[TOC]

##  一、alloc

### 1. alloc 的底层调用

新建项目并在 `ViewController.m` 中调用 `alloc` 代码。

```
NSObject *obj = [NSObject alloc];
```

这时候如果按照一般调试方式，进入 `alloc` 的实现只能在 `NSObject.h` 中看到 `alloc` 方法。这时可以下载[Runtime 源码](https://opensource.apple.com/tarballs/objc4/)进行查看。

#### 1.1 通过断点查看alloc实现

在 `NSObject *obj = [NSObject alloc];` 断点，然后按住 `Control`， 鼠标点击以下图标：

![image-20200527161949831](http://blog.objccf.com/blog/2020-05-27-081950.png)

> 这种调试属于 `Step controls` ，更多 `Step controls` 请参考该文底部链接。

之后会进入到：

```
Alloc&Init`objc_alloc:
->  0x1090034a0 <+0>: jmpq   *0x2b6a(%rip)             ; (void *)0x00000001090034ea
```

发现会进入到调用 `objc_alloc`。

#### 1.2 反汇编查看实现

在 `NSObject *obj = [NSObject alloc];` 断点，运行当停在断点处，打开 `Debug` 菜单下的 `Debug Workflow` 下的 `Always Show Disassembly` 

![image-20200527163836036](http://blog.objccf.com/blog/2020-05-27-083836.png)

然后 `Xcode` 显示如下：

```
Alloc&Init`-[ViewController viewDidLoad]:
  ...
->  0x107a3cf43 <+51>: movq   0x4426(%rip), %rax        ; (void *)0x00007fff89c1ed00: NSObject
    0x107a3cf4a <+58>: movq   %rax, %rdi
    0x107a3cf4d <+61>: callq  0x107a3d4a0               ; symbol stub for: objc_alloc
    0x107a3cf52 <+66>: xorl   %ecx, %ecx
    0x107a3cf54 <+68>: movl   %ecx, %esi
  ...
```

#### 1.3 符号断点：`objc_alloc`

继续执行会进入到：

```
libobjc.A.dylib`objc_alloc:
->  0x7fff50bbfbc5 <+0>:  testq  %rdi, %rdi
    0x7fff50bbfbc8 <+3>:  je     0x7fff50bbfbe4            ; <+31>
    0x7fff50bbfbca <+5>:  movq   (%rdi), %rax
    0x7fff50bbfbcd <+8>:  testb  $0x40, 0x1d(%rax)
    0x7fff50bbfbd1 <+12>: jne    0x7fff50bb80a3            ; _objc_rootAllocWithZone
    0x7fff50bbfbd7 <+18>: movq   0x3905ef72(%rip), %rsi    ; "alloc"
    0x7fff50bbfbde <+25>: jmpq   *0x36cb060c(%rip)         ; (void *)0x00007fff50ba4400: objc_msgSend
    0x7fff50bbfbe4 <+31>: xorl   %eax, %eax
    0x7fff50bbfbe6 <+33>: retq  
```

`objc_alloc` 来自 `libobjc.A.dylib`

### 2. 开始探索

不管是 `alloc`，还是 `objc_alloc`，在 `Runtime` 源码中，都会调用 `callAlloc`。

#### 2.1 alloc

```
+ (id)alloc {
    return _objc_rootAlloc(self);
}
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```

####  2.2 objc_alloc

```
// Calls [cls alloc].
id
objc_alloc(Class cls)
{
    return callAlloc(cls, true/*checkNil*/, false/*allocWithZone*/);
}
```

### 3. alloc 源码分析

由于 `alloc` 会调用 `objc_alloc`，`objc_alloc` 调用 `calAlloc`，本次分析以 `objc_alloc` 传参为准。

#### 3.1 callAlloc

```
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil = yes, bool allocWithZone=false)
{
#if __OBJC2__
	// 判断传入的 checkNil 和 cls 是否进行判空操作
    if (slowpath(checkNil && !cls)) return nil;
    // 当前类没有自定义的 allocWithZone
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
    	// 调用 _objc_rootAllocWithZone
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    // allocWithZone 最终也会调用 _objc_rootAllocWithZone
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    // alloc 会调用 objc_alloc，进而还是该方法
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}
```

##### 3.1.1 slowpath(x) & fastpath(x)

```
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
```

`__builtin_expect()` 是 `GCC` 提供的，作用是将最有可能执行的分支告诉编译器，编译器就据此将执行概率大的代码紧跟着前面的代码，从而减少指令跳转时 `cpu` 等待取指令的耗时。具体介绍请看参考链接。
`slowpath(x)` : x 很可能为0， 希望编译器进行优化。
`fastpath(x)` : x 很可能为1， 希望编译器进行优化。

##### 3.1.2 _objc_rootAllocWithZone

`_objc_rootAllocWithZone` 内部调用了 ` _class_createInstanceFromZone(cls, 0, nil,OBJECT_CONSTRUCT_CALL_BADALLOC)` 来创建对象 `obj`。

```
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}
```

`OBJECT_CONSTRUCT_CALL_BADALLOC`：构造标记，在创建对象 `obj` 失败的时候会调用 `_objc_callBadAllocHandler` 方法。
经过上面分析，alloc 对象的过程最终在 `_class_createInstanceFromZone` 函数上。

#### 3.2 _class_createInstanceFromZone

以下源码 `construct_flags` 的默认值是 `OBJECT_CONSTRUCT_NONE`， 这里改成 `OBJECT_CONSTRUCT_CALL_BADALLOC`。

```
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_CALL_BADALLOC,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

- `hasCxxCtor` & `hasCxxDtor()`：

     `class` 或 `superclass` 是否实现 `.cxx_construct` / `.cxx_destruct` 方法， `hasCxxCto`r 对应 `FAST_CACHE_HAS_CXX_CTOR`，`hasCxxDtor` 对应 `FAST_CACHE_HAS_CXX_DTOR`。

- `canAllocNonpointer`:

  对 `isa` 类型的区分，如果一个类是否支持优化的 `isa`，其实就是一个类的实例是否能使用 `isa_t` 类型的 `isa`，不能就返回 `false`。

- `size = cls->instanceSize(extraBytes)`

  这里是获取创建对象所需的内存大小。

  ```
  size_t instanceSize(size_t extraBytes) const {
  	if (fastpath(cache.hasFastInstanceSize(extraBytes))) {
  	return cache.fastInstanceSize(extraBytes);
  	}
  
  	size_t size = alignedInstanceSize() + extraBytes;
  	// CF requires all objects be at least 16 bytes.
  	if (size < 16) size = 16;
  	return size;
  }
  ```

  这里获取的最小需分配的内存大小是 16(CF要求所有对象最小是 16 个字节)。至于具体分配在内存对齐相关文章。

- `outAllocatedSize`: 如果有传入大小，就更改当前值为创建对象所需的内存大小

- `obj = (id)calloc(1, size);`

  由于传入的 `zone` 是 `nil`，所以会调用当前行代码，动态分配 `1` 个连续长度为 `size` 的内存空间。

- 初始化 `ISA` 指针

  因为 `zone` 是 `nil`, 在 `fast` 不同的情况下，会调用下面  `2`  个方法:

  ```
  obj->initInstanceIsa(cls, hasCxxDtor);
  obj->initIsa(cls);
  
  inline void 
  objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
  {
      ASSERT(!cls->instancesRequireRawIsa());
      ASSERT(hasCxxDtor == cls->hasCxxDtor());
  
      initIsa(cls, true, hasCxxDtor);
  }
  
  inline void 
  objc_object::initIsa(Class cls)
  {
      initIsa(cls, false, false);
  }
  ```

  从上述源码中，我们也能看出，最终都是调用了 `initIsa` 函数，只不过入参不同。

  ```
  inline void 
  objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
  { 
      ASSERT(!isTaggedPointer()); 
      
      if (!nonpointer) {
          isa = isa_t((uintptr_t)cls);
      } else {
          ASSERT(!DisableNonpointerIsa);
          ASSERT(!cls->instancesRequireRawIsa());
  
          isa_t newisa(0);
  
  #if SUPPORT_INDEXED_ISA
          ASSERT(cls->classArrayIndex() > 0);
          newisa.bits = ISA_INDEX_MAGIC_VALUE;
          // isa.magic is part of ISA_MAGIC_VALUE
          // isa.nonpointer is part of ISA_MAGIC_VALUE
          newisa.has_cxx_dtor = hasCxxDtor;
          newisa.indexcls = (uintptr_t)cls->classArrayIndex();
  #else
          newisa.bits = ISA_MAGIC_VALUE;
          // isa.magic is part of ISA_MAGIC_VALUE
          // isa.nonpointer is part of ISA_MAGIC_VALUE
          newisa.has_cxx_dtor = hasCxxDtor;
          newisa.shiftcls = (uintptr_t)cls >> 3;
  #endif
  
          // This write must be performed in a single store in some cases
          // (for example when realizing a class because other threads
          // may simultaneously try to use the class).
          // fixme use atomics here to guarantee single-store and to
          // guarantee memory order w.r.t. the class index table
          // ...but not too atomic because we don't want to hurt instantiation
          isa = newisa;
      }
  }
  ```

  - 如果该 `cls` 不支持 `isa` 优化，直接把当前 `cls` 作为 `isa`。

  - 如果 `cls` 支持 `isa` 优化，先初始化所有位为 `0` 的 `newisa`，其类型为 `isa_t`。
    `SUPPORT_INDEXED_ISA` 的值来源于下面源码:

    ```
    #if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
    #   define SUPPORT_INDEXED_ISA 1
    #else
    #   define SUPPORT_INDEXED_ISA 0
    #endif
    ```

    `__ARM_ARCH_7K__`  根据名称来看，是 `CPU` 的一个架构标志，命名为 `ARM 7k`，在 `LLVM` 开源的[ARM.cpp](https://clang.llvm.org/doxygen/Basic_2Targets_2ARM_8cpp_source.html)找到了其定义：

    ```
    // Unfortunately, __ARM_ARCH_7K__ is now more of an ABI descriptor. The CPU
    // happens to be Cortex-A7 though, so it should still get __ARM_ARCH_7A__.
    if (getTriple().isWatchABI())
    Builder.defineMacro("__ARM_ARCH_7K__", "2");
    ```

    `__arm64__` 是定义在目标为 `ARM 64` 架构的 `CPU` 的代码中；`_LP64_` 是 `64` 位系统的标志，意味着 `Long Pointer`。
    综上，在 `iOS` 的真机设备中，`SUPPORT_INDEXED_ISA` 的值始终是 `0`。

  - 将 `newisa` 的 `bits` 赋值为常量 `ISA_MAGIC_VALUE`，里面包含了 `magic` 和 `nonpointer`。
  - 标记 `newisa` 是否有析构函数。
  - 对  `cls` 右移 `3` 位，赋值给 `newida` 的 `shiftcls`，这里是为了节省位的使用(指针是 `8` 字节对齐，对应的是 `0001000`，右移之后是 `0001`)。
  - 最后把 `newisa` 赋值 给 `isa`。

- 拿到 `isa`，返回当前创建的对象。

- 流程图：

  ![image-20200528160044623](http://blog.objccf.com/blog/2020-05-28-080044.png)

## 二、init

```
- (id)init {
    return _objc_rootInit(self);
}
id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

`init ` 的作用就是返回当前对象。这里有个问题既然`init`只是返回当前对象，为什么要多此一举呢？
Apple给出的注释：

> In practice, it will be hard to rely on this function. Many classes do not properly chain -init calls.

意思是在实践中，很难依靠这个功能。许多类没有正确链接`init`调用。
难道就这？当然不是！！！！！！！
**类使用该方法来确保其属性在创建时具有合适的初始值。init方法所做的最重要的事情是创建自变量。**
因此，如果要使用 alloc 方法创建的实例，则必须对其进行初始化。

## 三、dealloc

源码：

```
- (void)dealloc {
    _objc_rootDealloc(self);
}

void
_objc_rootDealloc(id obj)
{
    ASSERT(obj);

    obj->rootDealloc();
}

inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;  // fixme necessary?

    if (fastpath(isa.nonpointer  &&  
                 !isa.weakly_referenced  &&  
                 !isa.has_assoc  &&  
                 !isa.has_cxx_dtor  &&  
                 !isa.has_sidetable_rc))
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}
```

如果当前释放对象时 `TaggedPointer`，直接返回。
`fastpath` 上面已解释过，不在阐述。

- `fastpath` 相关判断，当满足下面所有条件时，直接调用 `free` 释放。其实说白了，没有创建额外的实例变量会直接 `free`。

  - 当前对象的支持 `isa` 优化
  - 当前对象没有被弱引用
  - 没有关联对象
  - 没有 `c++` 或 `oc` 的析构器，就是有没有调用析构方法
  - 是否存储额外的引用计数

- 调用 `object_dispose((id)this)` 进行释放。源码如下：

  ```
  id 
  object_dispose(id obj)
  {
      if (!obj) return nil;
  
      objc_destructInstance(obj);    
      free(obj);
  
      return nil;
  }
  ```

  - 调用 `objc_destructInstance(obj)` 释放当前实例对象。
  - 释放当前分配内存。

- objc_destructInstance 源码：

  ```
  /// dealloc 的核心实现，内部会进行判断和析构操作
  void *objc_destructInstance(id obj) 
  {
      if (obj) {
          // Read all of the flags at once for performance.
          /// 是否有 OC/C++的析构函数
          bool cxx = obj->hasCxxDtor();
          /// 是否有关联对象
          bool assoc = obj->hasAssociatedObjects();
  
          // This order is important.
          /// 对对象进行析构
          if (cxx) object_cxxDestruct(obj);
          /// 移除所有关联关系
          if (assoc) _object_remove_assocations(obj);
          obj->clearDeallocating();
      }
  
      return obj;
  }
  ```

  - 如果当前对象被标记成析构，对当前对象进行析构，调用析构函数 `.cxx_destruct` 函数，先调用当前`cls`，在调用 `superclass`。在函数内部会进行对应的 `release` 操作。更改关于 ``.cxx_destruct`，参考底部文档。
  - 移除当前对象所有的关联关系。
  - 进行 `clear` 操作。

  具体函数解析：

  - `object_cxxDestruct`:

  - 
    ```
    void object_cxxDestruct(id obj)
    {
        if (!obj) return;
        if (obj->isTaggedPointer()) return;
        object_cxxDestructFromClass(obj, obj->ISA());
    }
    
    static void object_cxxDestructFromClass(id obj, Class cls)
    {
        void (*dtor)(id);
    
        // Call cls's dtor first, then superclasses's dtors.
    	/// 从当前类开始遍历，一直遍历到根类
        for ( ; cls; cls = cls->superclass) {
            if (!cls->hasCxxDtor()) return; 
            /// SEL_cxx_destruct 也就是 .cxx_destruct
            dtor = (void(*)(id))
                lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
            if (dtor != (void(*)(id))_objc_msgForward_impcache) {
                if (PrintCxxCtors) {
                    _objc_inform("CXX: calling C++ destructors for class %s", 
                                 cls->nameForLogging());
                }
                /// 获取到 .cxx_destruct 函数指针并调用
                (*dtor)(obj);
            }
        }
    }
    ```

    对象会调用 object_cxxDestruct 进行析构，内部会调用 object_cxxDestructFromClass 函数进行详细析构。函数内部会从当前类开始遍历，一直遍历到根类，在遍历的过程中，不断的的执行 SEL_cxx_destruct 进行析构。SEL_cxx_destruct 也就是 .cxx_destruct。

- 流程图：

  ![deallloc](http://blog.objccf.com/blog/2020-05-28-154838.png)


## 最后 & 纠正

**又不对的地方，欢迎指出，谢谢**

## 参考

[debugging_with_xcode](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/debugging_with_xcode/chapters/debugging_tools.html)
[__builtin_expect()](<https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>)
[ARM.cpp](https://clang.llvm.org/doxygen/Basic_2Targets_2ARM_8cpp_source.html)
[操作系统和编译器预定义宏](https://blog.virbox.com/?p=54)
[Objc 对象的今生今世](https://halfrost.com/objc_life/)
[apple working with Objects](<https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html#//apple_ref/doc/uid/TP40011210-CH5-SW2>)
[ARC下dealloc过程及.cxx_destruct的探究](https://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)





















