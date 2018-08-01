---
layout: post
title: NSAutoreleasePool 实现 
tags: [runtime,源码阅读]
---

## 概览

> An object that supports Cocoa’s reference-counted memory management system.  

一个支持 Cocoa 引用计数内存管理系统的对象。

自动释放池提供了一种机制，可以放弃对象的所有权，但避免立即释放他（例如从方法返回对象时）。通常来说，不需要自己创建自动释放池，但有些情况下除外。

[Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html) 给了需要手动创建自动释放池的情况

- 基于非 UI framework 的，例如 cli 工具
- 循环里包含非常多的临时对象时，因为多个临时对象可能占用过多的内存
- 创建了辅助线程的时候

### high level overview

首先来看看 NSAutoreleasePool 的接口，来自 [NSAutoreleasePool](https://developer.apple.com/documentation/foundation/nsautoreleasepool#//apple_ref/occ/cl/NSAutoreleasePool)

```objc
// managing a pool
- release
- drain
- autorelease
- retain
// adding an object to a pool
+ addObject
- addObject
// debugging
showPools
```

然后配合 [Let's Build NSAutoreleasePool](https://www.mikeash.com/pyblog/friday-qa-2011-09-02-lets-build-nsautoreleasepool.html)，使用效果更佳。

当 autorelease 消息被发送到 object 时，动作就被触发了。调用 [NSAutoreleasePool addObject: self] 添加对象，用来追踪正在讨论的那个实例。

每个线程的栈上都存储了一个 NSAutoreleasePool 实例。当一个 pool 被创建时，它被压到栈顶，当 pool 被销毁时，它被出栈。需要查看当前 pool 的时候，抓取线程栈顶的 pool 返回。

一旦 pool 被发现后，调用 `addObject:` 来添加 object。当 object 被添加 pool 里后，然后维护一份 object 的 list.

当 pool 被销毁，轮训 list 逐个释放里面的 object。这里稍微复杂的地方在于，如果销毁的 pool 不是位于 pools 的栈顶的话，那么所有在这个 pool 上面的也需要同步被销毁，例如有 pool 嵌套的情况，实际上使用哨兵对象来区分的。

## 原理

### @autoreleasepool 真实调用

```c
int main(int argc, const char * argv[]) {
	@autoreleasepool {
	}
	return 0;
}
```

我们在 main 里面调用了 @autoreleasepool ，然后看看 rewrite 后是什么

```c++
int main(int argc, const char * argv[]) {
 /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
 }
 return 0;
}
```

然后找到 __AtAutoreleasePool 结构体



```c++
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

发现里面的构造函数和析构函数，调用的是

- objc_autoreleasePoolPush
- objc_autoreleasePoolPop

然后就清晰了，在开始的时候调用 objc_autoreleasePoolPush，在销毁的时候调用 objc_autoreleasePoolPop。最后就变成了

```c++
	void *pool = objc_autoreleasePoolPush();

	...

	objc_autoreleasePoolPop(pool);
```

### 实现

#### AutoreleasePoolPage 结构

```c
class AutoreleasePoolPage {

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage *const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
}

```

- next 指向 add 对象的下一个位置
- thread 是当前线程
- parent 链表，指向父级 pool
- child 链表，指向下一级 pool
- depth 当前深度

![autoreleasePool struct](https://raw.githubusercontent.com/zlanchun/blogImages/master/NSAutoreleasePool/autoreleasePool.png)

#### 实现概览

这里结合工程源码里的注释看更容易理解。

> Autorelease pool implementation  
> A thread's autorelease pool is a stack of pointers.  
> Each pointer is either an object to release, or POOL_BOUNDARY which is an autorelease pool boundary.  
> A pool token is a pointer to the POOL_BOUNDARY for that pool. When  
> the pool is popped, every object hotter than the sentinel is released.   
> The stack is divided into a doubly-linked list of pages. Pages are added and deleted as necessary.  
> Thread-local storage points to the hot page, where newly autoreleased objects are stored.  

- 线程的 autorelease pool 是一个指针的栈结构
- 每个指针或者是对象（可以被释放的），或者是边界值 (哨兵对象)
- 一个 pool token 是指向该池的 POOL_BOUNDARY 的指针，当 pool 被弹出的时候，每一个比哨兵对象热的都会被释放（结合1里面来看，也就是说在栈哨兵上面的都会被释放）
- 栈使用双链表组成的，根据需要去添加或删除
- TLS (Thread-local storage) 指针指向 hot page，里面存储了新的自动释放对象。每一个线程都会有一个 TLS ，在上面 Let's Build NSAutoreleasePool 文章里面，使用字典实现的。 

什么是 hot page ？

hot page 是当前正在用的 autorelease pool page.

#### objc_autoreleasePoolPush

```c++

#define POOL_BOUNDARY nil

void *
objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

 static inline void *push() {
    id *dest;
    if (DebugPoolAllocation) {
        // Each autorelease pool starts on a new pool page.
        dest = autoreleaseNewPage(POOL_BOUNDARY);
    } else {
		  // push 的时候，这里插入的是哨兵对象 POOL_BOUNDARY
        dest = autoreleaseFast(POOL_BOUNDARY);
    }
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    ret    urn dest;
}

static inline id *autoreleaseFast(id obj) {
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}

// 入栈操作
id *add(id obj) {
    assert(!full());
    unprotect();
    id *ret = next; // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}

// page 满时，添加 object 的操作
static __attribute__((noinline))
id *
autoreleaseFullPage(id obj, AutoreleasePoolPage *page) {
    do {
        if (page->child)
            page = page->child;
        else
            page = new AutoreleasePoolPage(page);
    } while (page->full());
	  // 这里设定为 hotpage
    setHotPage(page);
    return page->add(obj);
}

// 没有 page 的时候创建一个
static __attribute__((noinline))
id *
autoreleaseNoPage(id obj) {

    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
	  // 设定这个新建的 page 是 hot page
    setHotPage(page);

    // Push a boundary on behalf of the previously-placeholder'd pool.
    if (pushExtraBoundary) {
        page->add(POOL_BOUNDARY);
    }

    // Push the requested object or pool.
    return page->add(obj);
}

```

push 的时候，首先插入的是一个哨兵。然后分为三种情况：

- 有 page 并且 page 没有满的时候，直接加入。因为当前 page 就是 hot 的，所以不需要设定
- 有 page 满了，新创建一个 page 然后加进去。因为是新建了一个 page 所以需要把这个 page 设定为 hot page
- 没有 page，创建一个 page 然后加进去，这个也是新建的，所以也需要设定为 hot page

#### objc_autoreleasePoolPop

```c++
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}

static inline void pop(void *token) {
    AutoreleasePoolPage *page;
    id *stop;

    if (token == (void *) EMPTY_POOL_PLACEHOLDER) {
        // Popping the top-level placeholder pool.
        if (hotPage()) {
            // Pool was used. Pop its contents normally.
            // Pool pages remain allocated for re-use as usual.
            pop(coldPage()->begin());
        } else {
            // Pool was never used. Clear the placeholder.
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);
    stop = (id *) token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin() && !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not
            // fatal in executables built with old SDKs.
            return badPop(token);
        }
    }

    if (PrintPoolHiwat) printHiwat();

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation && page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools && page->empty() && !page->parent) {
        // special case: delete everything for pop(top)
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}

```

pop 的时候，找到 hotPage，然后找到哨兵位置，比哨兵 hot 的都会一一释放掉。

## 引用

- [Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)
- [Let's Build NSAutoreleasePool](https://www.mikeash.com/pyblog/friday-qa-2011-09-02-lets-build-nsautoreleasepool.html)
- [黑幕背后的Autorelease · sunnyxx的技术博客](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)
- [自动释放池的前世今生 —— 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)

