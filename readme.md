---
title: 闲时队列
date: 2019-08-01 20:32:11
tags:
- iOS
- 组件设计
---

## 问题&思路

通常App启动，是需要尽量快的进入主界面，所以App启动的几秒内，属于重要的时间段，一些低优先级的任务要尽量延后。某些对性能要求较高的场景，比如列表滑动。目前这类处理，都是丢到globalQueue中处理。比如，启动时会下载h5资源，影响首屏网络加载等。

当我们有一些低优先级任务时，需要避开这些场景，仅通过延时，或者各自用自己的策略处理，可能不会特别恰当，并且也不好管理，考虑设计一个队列，当进入到这些场景时，就暂停任务，在进入一些不繁忙场景时，处理。

## 检测App是否繁忙的手段

### 使用方手动设置

比如启动设置繁忙，首屏展示之后设置为不繁忙

### 探测

下面的探测方法，取自戴铭老师在极客时间中的分享。

#### CPU使用率

App 运行起来后，会创建多个线程，每个线程都会使用CPU，各个线程使用CPU之和，就是App的CPU使用率，下面是线程数据结构，`cpu_usage`中记录了该线程的CPU占用，接下来，只需要定时遍历相加即可

````c++
struct thread_basic_info {
  time_value_t    user_time;     // 用户运行时长
  time_value_t    system_time;   // 系统运行时长
  integer_t       cpu_usage;     // CPU 使用率
  policy_t        policy;        // 调度策略
  integer_t       run_state;     // 运行状态
  integer_t       flags;         // 各种标记
  integer_t       suspend_count; // 暂停线程的计数
  integer_t       sleep_time;    // 休眠的时间
};

````

````objective-c
+ (integer_t)cpuUsage {
    thread_act_array_t threads; //int 组成的数组比如 thread[1] = 5635
    mach_msg_type_number_t threadCount = 0; //mach_msg_type_number_t 是 int 类型
    const task_t thisTask = mach_task_self();
    // 根据当前 task 获取所有线程
    kern_return_t kr = task_threads(thisTask, &threads, &threadCount);
    
    if (kr != KERN_SUCCESS) {
        return 0;
    }
    
    integer_t cpuUsage = 0;
    // 遍历所有线程
    for (int i = 0; i < threadCount; i++) {
        
        thread_info_data_t threadInfo;
        thread_basic_info_t threadBaseInfo;
        mach_msg_type_number_t threadInfoCount = THREAD_INFO_MAX;
        
        if (thread_info((thread_act_t)threads[i], THREAD_BASIC_INFO, (thread_info_t)threadInfo, &threadInfoCount) == KERN_SUCCESS) {
            // 获取 CPU 使用率
            threadBaseInfo = (thread_basic_info_t)threadInfo;
            if (!(threadBaseInfo->flags & TH_FLAGS_IDLE)) {
                cpuUsage += threadBaseInfo->cpu_usage;
            }
        }
    }
    assert(vm_deallocate(mach_task_self(), (vm_address_t)threads, threadCount * sizeof(thread_t)) == KERN_SUCCESS);
    return cpuUsage;
}

````

#### 内存使用量

在 2018 WWDC Session 416 [iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)中，提到phys_footprint是实际使用的物理内存。这个信息存在 task_info.h 的 task_vm_info结构中。

````c++
struct task_vm_info {
  mach_vm_size_t  virtual_size;       // 虚拟内存大小
  integer_t region_count;             // 内存区域的数量
  integer_t page_size;
  mach_vm_size_t  resident_size;      // 驻留内存大小
  mach_vm_size_t  resident_size_peak; // 驻留内存峰值
  ...
  /* added for rev1 */
  mach_vm_size_t  phys_footprint;     // 物理内存
  ...
````

直接取值返回即可

```c++
uint64_t memoryUsage() {
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    kern_return_t result = task_info(mach_task_self(), TASK_VM_INFO, (task_info_t) &vmInfo, &count);
    if (result != KERN_SUCCESS)
        return 0;
    return vmInfo.phys_footprint;
}

```

#### FPS监控

FPS 是指图像连续在显示设备上出现的频率。FPS 低，表示 App 不够流畅，还需要进行优化。

FPS 的监控也可以比较简单的实现：通过注册 CADisplayLink 得到屏幕的同步刷新率，记录每次刷新时间，然后就可以得到 FPS。

```objective-c
- (void)start {
    self.dLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(fpsCount:)];
    [self.dLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

// 方法执行帧率和屏幕刷新率保持一致
- (void)fpsCount:(CADisplayLink *)displayLink {
    if (lastTimeStamp == 0) {
        lastTimeStamp = self.dLink.timestamp;
    } else {
        total++;
        // 开始渲染时间与上次渲染时间差值
        NSTimeInterval useTime = self.dLink.timestamp - lastTimeStamp;
        if (useTime < 1) return;
        lastTimeStamp = self.dLink.timestamp;
        // fps 计算
        fps = total / useTime; 
        total = 0;
    }
}

```

### runloop 和 Notification

#### 通知队列

```objective-c
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1,
    NSPostASAP = 2,
    NSPostNow = 3
};
```

通知队列中有个可选项，`NSPostWhenIdle`

> The notification is posted when the run loop is idle.

在runloop空闲时发送。可以收到通知后进行处理。

优点：

* 系统自主触发，时机有保证
* 后续内部机制有修改也不会有影响

缺点：

只有开始，没有结束，每次收到通知，只能处理一个任务

#### runloop监听

````objective-c
CFRunLoopRef runLoop = CFRunLoopGetMain();
CFStringRef             mode = kCFRunLoopDefaultMode;
CFRunLoopActivity activities = kCFRunLoopBeforeWaiting | kCFRunLoopAfterWaiting;
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, activities, YES, INT_MAX, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
    if (activity == kCFRunLoopBeforeWaiting) {
        NSLog(@"idle");
    } else if (activity == kCFRunLoopAfterWaiting) {
        NSLog(@"runing");
    }
});
CFRunLoopAddObserver(runLoop, observer, mode);
````

## 代码设计

对外API的设计

