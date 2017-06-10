---

title: 老司机出品——源码解析之RunLoop详解
layout: post
date: 2017-05-02 07:33:44
updated: 2017-06-05 07:33:44
tags: 
- RunLoop 
categories: 源码解析

---

![RunLoop详解](http://upload-images.jianshu.io/upload_images/1835430-cccdcd8278846522.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


不得不说，人的**惰性**是真可怕啊。
从上周六就到写runLoop的建议开始，星期三告诉自己从星期四开始着手写这篇博客。然而现在戳个时间戳，现在是4.30星期日。写完发出去又不知道是什么时候啦，哈哈哈😁

![懒癌](http://upload-images.jianshu.io/upload_images/1835430-eb798302dcb50708.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这一期讲什么呢？这一期讲runLoop哟。一直以来，runLoop这个**玄而又玄**的东西似乎被当做了公司面试挑人的终极话题，原因不难想，日常开发用到`runLoop的地方少之又少`，没有时间的积累这方面的知识应该还是相对较于匮乏的，`所以runLoop的了解侧面也能发应开发者的开发经验`，当然就被当做甄选人才的最后杀器。你可能一脸愤怒的说平时有用不到，我不会也不影响开发啊！的确，用不到，但这只是一个过滤器而已。**但是蛋疼的是，在于国内的环境下，runLoop的相关资料又是少之又少，开发者又难以有一个深入的了解**。

出于以上原因，老司机今天就以**老司机个人的角度**，尽可能将老司机所了解到的runLoop知识。

在今天的文章中你可能会看到以下内容：

- runLoop相关知识

<!-- more -->

- - -
# runLoop是什么

直译以下，跑圈。翻译以下，`事件循环`吧。
为什么要有这个事件循环呢？我们知道，任何程序如果执行到程序的最后一句之后都会结束运行。然而对于我们要的手机应用程序而言，他显然不可以执行一个事件后就结束运行，他应该具有`持续接受事件`的能力从而不断地处理事件。所以最基本的思路就是用于个`while循环`让程序不能走到最后一句结束，而是在循环体内不断的接受事件。所以我们需要runLoop。不过值得注意的是，runLoop并不是iOS独有的概念，因为准去的来说`runLoop应该是一个模式，在其他平台同样存在这种模式`，不过叫不叫runLoop我就不知道了。


![循环](http://upload-images.jianshu.io/upload_images/1835430-9f5079fdc39bf56f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- - - 
# runLoop是如何实现的

首先要明确的一点事，在平时我们使用的是Foundation框架的NSRunLoop类去做一些实现，而其实NSRunLoop是基于CoreFoundation框架中的CFRunLoop进行的一层简单的封装。所以我们这里着重介绍**CFRunLoop**，毕竟我们能拿到[CFRunLoop的源码](http://opensource.apple.com/tarballs/CF/CF-855.17.tar.gz)。

## runLoop的组成

```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFTypeRef _counterpart;
};
```
我们大概可以CFRunLoop是这么一个结构体。
我们可以看到结构体重有用来保证线程安全的锁`_lock`，有用来唤醒runLoop的端口`_wakeUpPort`（这里后面会说到，不用执着），有线程对象`_pthread`，还有一个模式集合`_modes`以及一些其他辅助的属性。


### _pthread
这里我要说的是，runLoop与线程是一一对应的。也就是说**一个runLoop对应着一个线程，一个线程对应着一个runLoop**。这里我们从runLoop的构造函数和获取函数即可看出：

```
static CFRunLoopRef __CFRunLoopCreate(pthread_t t) {
    CFRunLoopRef loop = NULL;
    CFRunLoopModeRef rlm;
    uint32_t size = sizeof(struct __CFRunLoop) - sizeof(CFRuntimeBase);
    loop = (CFRunLoopRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, __kCFRunLoopTypeID, size, NULL);
    if (NULL == loop) {
	return NULL;
    }
    (void)__CFRunLoopPushPerRunData(loop);
    __CFRunLoopLockInit(&loop->_lock);
    loop->_wakeUpPort = __CFPortAllocate();
    if (CFPORT_NULL == loop->_wakeUpPort) HALT;
    __CFRunLoopSetIgnoreWakeUps(loop);
    loop->_commonModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    CFSetAddValue(loop->_commonModes, kCFRunLoopDefaultMode);
    loop->_commonModeItems = NULL;
    loop->_currentMode = NULL;
    loop->_modes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    loop->_blocks_head = NULL;
    loop->_blocks_tail = NULL;
    loop->_counterpart = NULL;
    loop->_pthread = t;
#if DEPLOYMENT_TARGET_WINDOWS
    loop->_winthread = GetCurrentThreadId();
#else
    loop->_winthread = 0;
#endif
    rlm = __CFRunLoopFindMode(loop, kCFRunLoopDefaultMode, true);
    if (NULL != rlm) __CFRunLoopModeUnlock(rlm);
    return loop;
}
```
可以看出构造一个runLoop对象仅需要一个**pthread_t**线程即可。即一个runLoop对应一个线程。

```
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {//如果传入线程为空指针则默认取主线程对应的runLoop
	t = pthread_main_thread_np();
    }
    __CFSpinLock(&loopsLock);
    if (!__CFRunLoops) {//__CFRunLoops就是一个全局字典，以下代码为如果全局字典不存在则创建全局字典，并将主线程对应的mainLoop存入字典中
        __CFSpinUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
        __CFSpinLock(&loopsLock);
    }
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));//从全局字典中，取出对应线程的runLoop
    __CFSpinUnlock(&loopsLock);
    if (!loop) {//若对应线程的runLoop为空，则创建对应相乘的runLoop并保存在全局字典中
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFSpinLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFSpinUnlock(&loopsLock);
        CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```

这是runLoop的获取函数，我们看到系统从一个**全局字典中取出runLoop**，key就是一个线程，这足以说明runLoop与线程是`一一对应`的关系。

值得一提的是，`一个线程最开始是没有对应的runLoop`的，是`在调用获取函数的时候才对应了一个runLoop的`。**因为本身这个对应关系是有runLoop类管理的，而不是线程**。


当然上述两个为私有api，CF真正对外暴露的只有两个接口：

```
CF_EXPORT CFRunLoopRef CFRunLoopGetCurrent(void);
CF_EXPORT CFRunLoopRef CFRunLoopGetMain(void);
```

两个方法的实现很简单，只要把对应的线程传入获取函数即可：

```
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```

- - -
### _modes

我们看到，一个runLoop中同时还维护着一个集合，_modes。那么这个modes是做什么的呢？应该说，_modes才是`runLoop的核心`。咳咳（敲黑板），划重点了啊。

首先我们看一下这个_modes里面到底都装了些什么？
答案是`__CFRunLoopMode`对象。那么他又是什么呢？

```
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```
这里老司机挑出了几个重点，有用来标志runLoopMode的标志`_name`，有两个事件源的集合`_sources0、_sources1`，有一组观察者`_obeserver`，有一组被加入到runLoop中的`_timers`，还有Mode本身维护着的一个用于计时的`_timerSource`，`_timerPort`。这两个一个是GCD时钟一个是内核时钟。

至于runLoopMode为什么长这样，老司机会在下面runLoopRun的实现中结合代码讲到。
- - - 
## runLoop代码实现

恩，接下来代码有点长，先给你们看一下大概流程，然后对着流程去看一下代码。

![图是我盗的](http://upload-images.jianshu.io/upload_images/1835430-a84a8802abc125cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

前方高能预警，代码很多！

runLoop核心代码

```
/* rl, rlm are locked on entrance and exit */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();//获取当前内核时间
    
    if (__CFRunLoopIsStopped(rl)) {//如果当前runLoop或者runLoopMode为停止状态的话直接返回
        __CFRunLoopUnsetStopped(rl);
        return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        rlm->_stopped = false;
        return kCFRunLoopRunStopped;
    }
    
    //判断是否是第一次在主线程中启动RunLoop,如果是且当前RunLoop为主线程的RunLoop，那么就给分发一个队列调度端口
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS

	//给当前模式分发队列端口
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    //初始化一个GCD计时器，用于管理当前模式的超时
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
        dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, DISPATCH_QUEUE_OVERCOMMIT);
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
        timeout_context->ds = timeout_timer;
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }
    
    // 第一步，进入循环
    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#endif
        __CFPortSet waitSet = rlm->_portSet;
        
        //设置当前循环监听端口的唤醒
        __CFRunLoopUnsetIgnoreWakeUps(rl);
        
        // 第二步，通知观察者准备开始处理Timer源事件
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        
        // 第三步，通知观察者准备开始处理Source源事件
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        //执行提交到runLoop中的block
        __CFRunLoopDoBlocks(rl, rlm);
        
        // 第四步，执行source0中的源事件
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        
        //如果当前source0源事件处理完成后执行提交到runLoop中的block
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        //标志是否等待端口唤醒
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        
        // 第五步，检测端口，如果端口有事件则跳转至handle_msg（首次执行不会进入判断，因为didDispatchPortLastTime为true）
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }
        
        didDispatchPortLastTime = false;
        
        // 第六步，通知观察者线程进入休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        
        // 标志当前runLoop为休眠状态
        __CFRunLoopSetSleeping(rl);
        
        // do not do any user callouts after this point (after notifying of sleeping)
        
        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.
        
        __CFPortSetInsert(dispatchPort, waitSet);
        
        __CFRunLoopModeUnlock(rlm);
        __CFRunLoopUnlock(rl);
   
   
   		// 第七步，进入循环开始不断的读取端口信息，如果端口有唤醒信息则唤醒当前runLoop     
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        do {
            if (kCFUseCollectableAllocator) {
                objc_clear_stack(0);
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator) {
            objc_clear_stack(0);
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY);
#endif
        
        
#elif DEPLOYMENT_TARGET_WINDOWS
        // Here, use the app-supplied message queue mask. They will set this if they are interested in having this run loop receive windows messages.
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);
        
        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.
        
        __CFPortSetRemove(dispatchPort, waitSet);
        
        //标志当前runLoop为唤醒状态
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        // user callouts now OK again
        __CFRunLoopUnsetSleeping(rl);
        
        // 第八步，通知观察者线程被唤醒了
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
        
        //执行端口的事件
    handle_msg:;
    
    	//设置此时runLoop忽略端口唤醒（保证线程安全）
        __CFRunLoopSetIgnoreWakeUps(rl);
        
#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            // These Win32 APIs cause a callout, so make sure we're unlocked first and relocked after
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            
            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            
            // To prevent starvation of sources other than the message queue, we check again to see if any other sources need to be serviced
            // Use 0 for the mask so windows messages are ignored this time. Also use 0 for the timeout, because we're just checking to see if the things are signalled right now -- we will wait on them again later.
            // NOTE: Ignore the dispatch source (it's not in the wait set anymore) and also don't run the observers here since we are polling.
            __CFRunLoopSetSleeping(rl);
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            
            __CFRunLoopWaitForMultipleObjects(waitSet, NULL, 0, 0, &livePort, NULL);
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            __CFRunLoopUnsetSleeping(rl);
            // If we have a new live port then it will be handled below as normal
        }
        
        
#endif

		// 第九步，处理端口事件
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
#if DEPLOYMENT_TARGET_WINDOWS
            // Always reset the wake up port, or risk spinning forever
            ResetEvent(rl->_wakeUpPort);
#endif
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {//处理定时器事件
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
            // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
		//处理有GCD提交到主线程唤醒的事件
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
#if DEPLOYMENT_TARGET_WINDOWS
            void *msg = 0;
#endif
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
        
	        //处理source1唤醒的事件
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            // Despite the name, this works for windows handles as well
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                mach_msg_header_t *reply = NULL;
                // 处理Source1(基于端口的源)
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
            }
        }
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif
        
        __CFRunLoopDoBlocks(rl, rlm);
        
        //返回对应的返回值并跳出循环
        if (sourceHandledThisLoop && stopAfterHandle) {
            retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            retVal = kCFRunLoopRunTimedOut;
        } else if (__CFRunLoopIsStopped(rl)) {
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            retVal = kCFRunLoopRunFinished;
        }
    } while (0 == retVal);
    
    // 第十步，释放定时器
    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }
    
    return retVal;
}
```

这个方法有点有点长，300行代码=。=

这300行的流程其实就是上面归纳的10步：

> 首先进入runLoop对应的Mode并开始循环，然后在休眠之前做了三件事：DoBlocks、DoSource0、检测source1端口是否有消息，如果有则跳过稍后的休眠。
然后runLoop就进入了休眠状态，直到有端口事件唤醒runLoop，被唤醒后则处理响应的端口事件然后再次开始循环。直到runLoop超时或者runLoop被停止后在结束runLoop。

不过好在代码很全，在这里我们能出到很多问题。

### source0，source1

首先这个源事件分为两种，一种是不基于端口的source0，一直是基于端口的source1。


> Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

> Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程，其原理在下面会讲到。

> ————[引自深入理解RunLoop
](http://blog.ibireme.com/2015/05/18/runloop/)

- - -

> source0呢主要处理App内部事件、App自己负责管理（触发），如UIEvent、CFSocket

> source1呢主要有Runloop和内核管理，Mach port驱动，如CFMahPort、CFMessagePort

> ————引自孙源runLoop线下分享会视频 




### NSTimer事件是借助runLoop实现的。
这点老司机早在CoreAnimation系列中第三篇介绍三个Timer的时候老司机就有提到过，在初始化Timer的时候要将Timer提交到runLoop中，并且要指定mode，才可以工作。今天我们可以深入讲一下。

> 这个事件是怎么执行的？并且为什么有的时候会延迟？为什么子线程中创建的Timer并不执行？

> 首先，在进入循环开始以后，就要处理source0事件，处理后检测一下source1端口是否有消息，如果一个Timer的时间间隔刚好到了则此处有可能会得到一个消息，则runLoop直接跳转至端口激活处从而去处理Timer事件。

> 第二，为什么会延迟？我们知道，两次端口事件是在两个runLoop循环中分别执行的。比如Timer的时间间隔为1秒，在第一次Timer回调结束后，在很短时间内立即进入runLoop的下一次循环，这次并不是Timer回调并且是一个计算量非常大的任务，计算时间超过了1秒，那么runLoop的第二个循环就要执行很久，无法进入下一个循环等待有可能即将到来的Timer第二次回调的信号，所以Timer第二次回调就会推迟了。

> 第三，为什么在子线程中创建的Timer并且提交到当前runLoop中并不会运行？这还是要从runLoop的获取函数中看，当调用currentRunLoop的时候会取当前线程对应的runLoop，而首次是取不到的，则会创建一个新的runLoop。但是！这个runLoop并没有run。就是没有开启=。=

- - -
### 同一时间内，runLoop只能运行同一种mode。那commonMode是怎么实现的？

从runLoop的结构我们可以知道，一个runLoop会包含多种runLoopMode，runLoop是不停的在这些mode之间进行切换去完成对应Mode中的相关任务。

![runLoop中多个mode](http://upload-images.jianshu.io/upload_images/1835430-1a9ffd190ef7c73b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先为什么说runLoop只能在各种Mode之间切换，同一时间只能存在一个呢？
因为上面那个方法必须要传一个runLoopMode，然后这个方法贯穿始终，都在用。


我们看到，上面的方法中首先就要传入一个指定的mode才能执行对应mode中的事件。那么所谓的CommonMode是如何实现的呢？

我们看到runLoop中执行任务有调到CFRunLoopDoBlocks这么一个函数，那么这个函数是什么样的呢？

```
static Boolean __CFRunLoopDoBlocks(CFRunLoopRef rl, CFRunLoopModeRef rlm) { // Call with rl and rlm locked
    if (!rl->_blocks_head) return false;
    if (!rlm || !rlm->_name) return false;
    ...省略一些非重点...
    while (item) {
        struct _block_item *curr = item;
        item = item->_next;
	Boolean doit = false;
	if (CFStringGetTypeID() == CFGetTypeID(curr->_mode)) {
	    doit = CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
        } else {
	    doit = CFSetContainsValue((CFSetRef)curr->_mode, curMode) || (CFSetContainsValue((CFSetRef)curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode));
	}
	if (!doit) prev = curr;
	if (doit) {
	    if (prev) prev->_next = item;
	    if (curr == head) head = item;
	    if (curr == tail) tail = prev;
	    void (^block)(void) = curr->_block;
            CFRelease(curr->_mode);
            free(curr);
	    if (doit) {
                __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
	        did = true;
	    }
    ...省略一些非重点...
    return did;
}
```

我们看到`doit`这个bool变量完全决定了当前block是否执行。默认他是No的，而他被置为true的条件就是`CFEqual(curr->_mode, curMode) || (CFEqual(curr->_mode, kCFRunLoopCommonModes) && CFSetContainsValue(commonModes, curMode))`。就是当前mode与制定mode相等或者当前mode为commonMode（此处为一个字符串）且commonMode（此处为一个集合，若有不懂，请看runLoop结构）这个集合中包含指定mode。

这是因为这个判断的存在才允许commondMode可以在任意Mode下执行。
当然这是提交到runLoop里的代码块才会走到`__CFRunLoopDoBlocks`这个方法。

相同的，我们通过上述代码也可以知道，runLoop通过端口唤醒的事件需要通过__CFRunLoopDoSource1和__CFRunLoopDoTimers两个方法来调用。__CFRunLoopDoSource1方法没什么说的，直接调用源事件runLoopSourceRef即可。重点我们看一下Timer的实现，核心代码如下：

```
static Boolean __CFRunLoopDoTimers(CFRunLoopRef rl, CFRunLoopModeRef rlm, uint64_t limitTSR) {	/* DOES CALLOUT */
    Boolean timerHandled = false;
    CFMutableArrayRef timers = NULL;
    //遍历runLoopMode维护的Timers数组，取其中有效的timer并加入新临时数组
    for (CFIndex idx = 0, cnt = rlm->_timers ? CFArrayGetCount(rlm->_timers) : 0; idx < cnt; idx++) {
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(rlm->_timers, idx);
        
        if (__CFIsValid(rlt) && !__CFRunLoopTimerIsFiring(rlt)) {
            if (rlt->_fireTSR <= limitTSR) {
                if (!timers) timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
                CFArrayAppendValue(timers, rlt);
            }
        }
    }
    //遍历临时数组，每个有效Timer调用__CFRunLoopDoTimer
    for (CFIndex idx = 0, cnt = timers ? CFArrayGetCount(timers) : 0; idx < cnt; idx++) {
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(timers, idx);
        Boolean did = __CFRunLoopDoTimer(rl, rlm, rlt);
        timerHandled = timerHandled || did;
    }
    if (timers) CFRelease(timers);
    return timerHandled;
}
```
我们可以看到，此处Timer是否会回调完全取决于对应Mode的_Timers数组。那么当我们将Timer加入到commonModes中的时候一定是同时将Timer加入到了commonModes所包含的其他Mode中了，我们看下代码：

```
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {    
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {//commonModes分支
    	//取到commonModes所代表的Mode的集合
        CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        if (NULL == rl->_commonModeItems) {
        	//将commonModeItems中加入当前定时器
            rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
        }
        CFSetAddValue(rl->_commonModeItems, rlt);
        if (NULL != set) {
            CFTypeRef context[2] = {rl, rlt};
            /* add new item to all common-modes */
            //最主要还是还是这句，这句的作用是集合中的所有对象均调用__CFRunLoopAddItemToCommonModes这个方法。
            CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
            CFRelease(set);
        }
    } else {//非commonModes的分支
        CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
        if (NULL != rlm) {
            if (NULL == rlm->_timers) {
                CFArrayCallBacks cb = kCFTypeArrayCallBacks;
                cb.equal = NULL;
                rlm->_timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &cb);
            }
        }
        if (NULL != rlm && !CFSetContainsValue(rlt->_rlModes, rlm->_name)) {
            __CFRunLoopTimerLock(rlt);
            if (NULL == rlt->_runLoop) {
                rlt->_runLoop = rl;
            } else if (rl != rlt->_runLoop) {
                __CFRunLoopTimerUnlock(rlt);
                __CFRunLoopModeUnlock(rlm);
                __CFRunLoopUnlock(rl);
                return;
            }
            CFSetAddValue(rlt->_rlModes, rlm->_name);
            __CFRunLoopTimerUnlock(rlt);
            __CFRunLoopTimerFireTSRLock();
            __CFRepositionTimerInMode(rlm, rlt, false);
            __CFRunLoopTimerFireTSRUnlock();
            if (!_CFExecutableLinkedOnOrAfter(CFSystemVersionLion)) {
                // Normally we don't do this on behalf of clients, but for
                // backwards compatibility due to the change in timer handling...
                if (rl != CFRunLoopGetCurrent()) CFRunLoopWakeUp(rl);
            }
        }
        if (NULL != rlm) {
            __CFRunLoopModeUnlock(rlm);
        }
    }
    __CFRunLoopUnlock(rl);
}

static void __CFRunLoopAddItemToCommonModes(const void *value, void *ctx) {
    CFStringRef modeName = (CFStringRef)value;
    CFRunLoopRef rl = (CFRunLoopRef)(((CFTypeRef *)ctx)[0]);
    CFTypeRef item = (CFTypeRef)(((CFTypeRef *)ctx)[1]);
    if (CFGetTypeID(item) == __kCFRunLoopSourceTypeID) {
	CFRunLoopAddSource(rl, (CFRunLoopSourceRef)item, modeName);
    } else if (CFGetTypeID(item) == __kCFRunLoopObserverTypeID) {
	CFRunLoopAddObserver(rl, (CFRunLoopObserverRef)item, modeName);
    } else if (CFGetTypeID(item) == __kCFRunLoopTimerTypeID) {
	CFRunLoopAddTimer(rl, (CFRunLoopTimerRef)item, modeName);
    }
}


```
我们可以看到，当加入到commonModes中时，实际上系统是`找出commonModes代表的所有Mode`，如defaultMode和trackingMode，让后分别将其加入了这些mode中。
同样的方法还有`CFRunLoopAddSource`/`CFRunLoopAddObserver`都是同样的道理。




所以说当scrollView或其子类进行滚动的时候，UIKIT会自动将当前runLoopMode切换为UITrackingRunLoopMode，所以你加在defaultMode中的计时器当然不会走了。



---
### runLoop是如何休眠有如何被唤醒的？

从第7步开始，我们看到runLoop进入了休眠状态。然而所谓的休眠状态指示将当前runLoop标记为休眠之后，**进入了一个while死循环**。然后在循环内就不断的去读取端口消息。如果说从端口中**读取到一个唤醒信息的话，break掉while循环从而进入唤醒状态**。


关于runLoop的几种mode老司机之前也有讲过，在CoreAnimation中的第三篇中有讲到，这里就只罗列一下。

- NSDefaultRunLoopMode
- NSConnectionReplyMode
- NSModalPanelRunLoopMode
- UITrackingRunLoopMode
- NSRunLoopCommonModes

- - -
### 可以唤醒runLoop的都有哪些事件？
从源码中我们可以看出，所谓的runLoop进入休眠状态不过是一个while循环，如下：

```
do {
            if (kCFUseCollectableAllocator) {
                objc_clear_stack(0);
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
 } while (1);
```

相应的我们还得看一个函数，`__CFRunLoopServiceMachPort`：

```
static Boolean __CFRunLoopServiceMachPort(mach_port_name_t port, mach_msg_header_t**buffer, size_t buffer_size, mach_port_t *livePort, mach_msg_timeout_t timeout) {
    Boolean originalBuffer = true;
    kern_return_t ret = KERN_SUCCESS;
    for (;;) {		/* In that sleep of death what nightmares may come ... */
        mach_msg_header_t *msg = (mach_msg_header_t *)*buffer;
        msg->msgh_bits = 0;
        msg->msgh_local_port = port;
        msg->msgh_remote_port = MACH_PORT_NULL;
        msg->msgh_size = buffer_size;
        msg->msgh_id = 0;
        if (TIMEOUT_INFINITY == timeout) { CFRUNLOOP_SLEEP(); } else { CFRUNLOOP_POLL(); }
        ret = mach_msg(msg, MACH_RCV_MSG|MACH_RCV_LARGE|((TIMEOUT_INFINITY != timeout) ? MACH_RCV_TIMEOUT : 0)|MACH_RCV_TRAILER_TYPE(MACH_MSG_TRAILER_FORMAT_0)|MACH_RCV_TRAILER_ELEMENTS(MACH_RCV_TRAILER_AV), 0, msg->msgh_size, port, timeout, MACH_PORT_NULL);
        CFRUNLOOP_WAKEUP(ret);
        if (MACH_MSG_SUCCESS == ret) {
            *livePort = msg ? msg->msgh_local_port : MACH_PORT_NULL;
            return true;
        }
        if (MACH_RCV_TIMED_OUT == ret) {
            if (!originalBuffer) free(msg);
            *buffer = NULL;
            *livePort = MACH_PORT_NULL;
            return false;
        }
        if (MACH_RCV_TOO_LARGE != ret) break;
        buffer_size = round_msg(msg->msgh_size + MAX_TRAILER_SIZE);
        if (originalBuffer) *buffer = NULL;
        originalBuffer = false;
        *buffer = realloc(*buffer, buffer_size);
    }
    HALT;
    return false;
}
```

我们先看后面这个函数，在这里仅有`两种情况会对livePort进行赋值`，一种是**成功获取到消息后**，会根据情况赋值为msg->msgh_local_port或者MACH_PORT_NULL，而另一种**获取消息超时**的情况会赋值为MACH_PORT_NULL。首先请先记住这两个结论。

然后我们把目光聚焦到while循环中，在调用`__CFRunLoopServiceMachPort`后如果livePort变成了`modeQueuePort`(livePort初值为MACH_PORT_NULL)，则代表为当前队列的检测端口，那么在`_dispatch_runloop_root_queue_perform_4CF`的条件下再次进入二级循环，知道Timer被激活了才跳出二级循环继续循环一级循环。（这一步的目的不好意思老司机真没看懂）。

那么如果livePort不为modeQueuePort时我们的runLoop被唤醒。这代表__CFRunLoopServiceMachPort给出的livePort只有两种可能：`一种情况为MACH_PORT_NULL，另一种为真正获取的消息的端口`。

所以我们可以看到后面runLoop处理端口时间的方法如下的判断：

```
        if (MACH_PORT_NULL == livePort) {//什么都不做，有肯能是超时之类的或者是信息过大
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {//只有外界调用CFRunLoopWakeUp才会进入此分支，这是外部主动唤醒runLoop的接口
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {//这里不是从runLoop休眠后唤醒到这里的，而是在runLoop10步中的第五步跳转过来的，是处理计时器事件
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            ...省略处理计时器事件的代码...
        }
#endif
        else if (livePort == dispatchPort) {//这里是处理GCD提交到mainQueue的block的端口事件
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            ...省略处理GCD的代码...
        } else {//之前所有情况都不是，那么唤醒runLoop的就只可能是source1的源事件了。
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            ...省略处理source1源事件的代码...
            }
        }
```

`runLoop的唤醒过程，及唤醒过后的时间处理就是上面的流程`，大家可以看看每个分支后的注释。同时runLoopRun的核心代码也就解读完毕了。

剩下的几个run方法事实上都是对这个核心方法的封装了老司机不都说了：

- CFRunLoopRunSpecific
- CFRunLoopRun
- CFRunLoopRunInMode


至此，整个runLoop中的核心流程老司机也算带着大家分析了一遍~


![喘一口气](http://upload-images.jianshu.io/upload_images/1835430-c01c74630903d263.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- - -
# runLoop都能做什么

说了这么多，那么runLoop都能做些什么呢?

以下内容整理自[深入理解RunLoop
](http://blog.ibireme.com/2015/05/18/runloop/)、孙源runLoop线下分享会视频：

## AutoReleasePool：


>App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

>第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

>第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

>在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

## CAAnimation
我们知道CAAniamtion为我们提供的是`补间动画`，开发者只要给出始末状态后中间状态有系统自动生成。那么动画是怎么出现的呢，是开发者给出始末状态后，`系统计算出每一个中间态的各项参数`，`然后启一个定时器不断去回调并改变属性`。

## 事件响应

>苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

>当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考这里。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

>_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

## 手势识别

>当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

>苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

>当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

## 定时器
不多说了这个就。

## PerformSelecter

>当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

>当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

基本也就差不多了。
- - -
老司机写这篇博客呢，`也是应盆友的要求写的`，基本上也是针`对前人博客的总结`以及`自己对源码的解读`。**毕竟有了源码以后runLoop也就没有那么神秘了**。只是希望大家明白**runLoop并不是什么多么可怕的东西**，只要我们一点一点去看，他**也是人写的代码啊**=。=不过老司机懒到连伪代码都没给你们写直接上的源码。`原谅一个懒癌晚期的人吧`。

还是要感谢两位大神**郭耀源**和**孙源**两位大神之前的博客和视频讲解让我很受用。大神名里都带源字，我要不要改成`老源`=。=

- --
参考资料：

- [深入理解RunLoop
](http://blog.ibireme.com/2015/05/18/runloop/)
- [孙源runLoop线下分享会视频](https://pan.baidu.com/s/1dFIfLAD)
- [关于RunLoop部分源码的注释](http://blog.csdn.net/ssirreplaceable/article/details/53793456)
- [CFRunLoop的源码](http://opensource.apple.com/tarballs/CF/CF-855.17.tar.gz)

另外你如果想看孙源的视频，老司机已经给了下载链接，然后从**最开始到1小时10分钟**的时候都是干货，后面35分钟偏讨论，时间紧的童靴可以跳过，但是后面也有很好的思路分享，听听也是不错的。

如果你想看runLoop源码，因为源码里面有4000多行，老司机读源码的时候讲有用的方法的行数都记了下来，你可以对应的找一下：

|方法名|行数|
|:---:|:---:|
| __CFRunLoopFindMode   	|#754|
| __CFRunLoopCreate     	|#1321|
| _CFRunLoopGet0				|#1358|
|__CFRunLoopAddItemToCommonModes|#1536|
| __CFRunLoopDoObservers	|#1668|
| __CFRunLoopDoSources0  	|#1764|
| __CFRunLoopDoSource1		|#1829|
|__CFRepositionTimerInMode	|#1999|
| __CFRunLoopDoTimer		|#2024|
| __CFRunLoopDoTimers		|#2152|
|__CFRunLoopServiceMachPort|#2196|
| __CFRunLoopRun				|#2308|
| CFRunLoopRunSpecific		|#2601|
| CFRunLoopRun				|#2628|
| CFRunLoopRunInMode		|#2636|
|CFRunLoopWakeUp				|#2645|
|CFRunLoopAddSource			|#2791|
|CFRunLoopAddObserver		|#2978|
|CFRunLoopAddTimer			|#3081|



打完收功！
![打完收功](http://upload-images.jianshu.io/upload_images/1835430-96b2100c0f20d96a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

老司机写这篇博客真的事废了很多心血的，好就给赞关注吧~么么哒😘~

无耻的广告时间：

DWCoreTextLabel支持`cocoaPods`了~
**pod search DWCoreTextLabel**

![DWCoreTextLabel](http://upload-images.jianshu.io/upload_images/1835430-bc004dc959f7b675.gif?imageMogr2/auto-orient/strip)


插入图片、绘制图片、添加事件统统一句话实现~


![一句话实现](http://upload-images.jianshu.io/upload_images/1835430-456beaed3f710b02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


尽可能保持系统Label属性让你可以无缝过渡使用~


![无缝过渡](http://upload-images.jianshu.io/upload_images/1835430-e7706ffe827b1ea1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


恩，说了这么多，老司机放一下地址：[DWCoreTextLabel](https://github.com/CodeWicky/DWCoreTextLabel)，宝宝们给个star吧~爱你哟~
