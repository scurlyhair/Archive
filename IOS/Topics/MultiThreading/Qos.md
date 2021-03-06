# Quality of service

程序的执行依赖于有限的硬件资源（CPU、内存、网络等），使用更多的资源也意味着更多的电量消耗。

为了保证程序正常高效地运作，系统就需要为每个任务分配一定的资源调度能力。跟用户交互相关的任务比如 UI 操作，动画操作等，需要优先执行，以保证用户有流畅的体验。

作为开发者，我们可以根据需要为任务添加权重，来帮助系统更好地进行资源分配。

Qos（Quality of service）就是 Apple 在 iOS8 引入的一种基于资源分配的权重策略。它可以被应用于 *NSOperation, NSThread, GCD, 和 pthreads (POSIX threads)* 等多线s程管理模式中。

## Qos 级别

Qos 的权重级别、应用场景以及考量标准如下表：

| Qos 级别 | 任务类别 | 考量标准 | 任务时间 |
| --- | --- | --- | --- |
| .userInteractive | 跟用户交互相关的任务，例如需要在主线程操作的任务、刷新 UI、动画展示等。如果这些任务不能及时被执行，用户就会感觉到卡顿。|  系统优先考虑响应和性能 | 几乎立刻执行完毕 |
| .userInitiated | 用户执行操作之后需要立刻得到结果的任务，例如文档保存，点击事件的响应等。| 系统优先考虑响应和性能 | 几秒钟或者更少 |
| .utility | 需要花一些时间才能执行完毕的任务，不需要直接得到结果，例如下载文件、加载数据等，一般而言这种任务会展示给用户一个执行进度条。| 系统会在响应时间、性能和电量消耗之间取得一个平衡 | 几秒钟乃至几分钟 |
| .background | 一些不被用户所见的后台任务，例如数据的索引、同步、备份等 | 系统会优先考虑电量消耗 | 几分钟甚至更长时间 |

> 注意：
> 
> 1. 非用户交互操作和响应的任务，绝大部分都应该使用 .utility 甚至更低的权重级别。
> 2. 当 iPhone 开启低电量模式的时候，一些由系统自主决定的任务以及 .background 的任务会被暂停，包括网络请求。

另外，还有两个特别的权重级别我们一般不会用到，但还是需要做一些了解：

| Qos 级别 | 描述 |
| --- | --- |
| .default | 介于 .initiated 和 .utility 之间，Apple 不鼓励开发者为任务指定此权重，当任务的 qos 缺失时，系统会以 default 的标准做处理。另外， GCD 的 global queue 的权重级别就是 .default |
| .unspecified | This represents the absence of QoS information and cues the system that an environmental QoS should be inferred. Threads can have an unspecified QoS if they use legacy APIs that may opt the thread out of QoS. <br/>这个权重代表 Qos 信息未指定，系统会根据执行环境来进行推断其权重。另外，当任务使用 Qos 的不支持的旧版接口切导致线程退出 Qos 策略时，当前线程的权重也会被标识为 .unspecified |

我们可以通过一些耗时的操作来进行验证：

```swift

import Foundation

func test() {
    let queue = DispatchQueue(label: "concurrent_queue", attributes: [.concurrent])

    let work1 = DispatchWorkItem(qos: .background) {
        doSomeWork(withName: "work 1")
    }
    let work2 = DispatchWorkItem(qos: .utility) {
        doSomeWork(withName: "work 2")
    }
    let work3 = DispatchWorkItem(qos: .userInitiated) {
        doSomeWork(withName: "work 3")
    }
    let work4 = DispatchWorkItem(qos: .userInteractive) {
        doSomeWork(withName: "work 4")
    }
    queue.async(execute: work1)
    queue.async(execute: work2)
    queue.async(execute: work3)
    queue.async(execute: work4)
}

func doSomeWork(withName name: String) {
    print(name, "start", Thread.current, CFAbsoluteTimeGetCurrent())
    var array: [Int] = []
    for i in 0..<2000 {
        array.append(i)
    }
    print(name, "done", Thread.current, CFAbsoluteTimeGetCurrent())
}

test()

/**
 work 1 start <NSThread: 0x6000007e0100>{number = 4, name = (null)} 618469487.961388
 work 2 start <NSThread: 0x6000007f4600>{number = 5, name = (null)} 618469487.96142
 work 3 start <NSThread: 0x6000007e0a80>{number = 6, name = (null)} 618469487.961492
 work 4 start <NSThread: 0x6000007c5f00>{number = 7, name = (null)} 618469487.96153
 
 work 4 done <NSThread: 0x6000007c5f00>{number = 7, name = (null)} 618469489.726575
 work 3 done <NSThread: 0x6000007e0a80>{number = 6, name = (null)} 618469489.911544
 work 2 done <NSThread: 0x6000007f4600>{number = 5, name = (null)} 618469490.061548
 work 1 done <NSThread: 0x6000007e0100>{number = 4, name = (null)} 618469490.262329
 */
```

可以看到在执行相同耗时任务的并发队列中，即使权重较低的任务先开始执行，但权重高的任务会因为可以获得更多的系统资源而优先执行完毕。

但这并不能说明权重级别高的任务一定优先完成，比如把测试函数中的 doSomeWork 做以下修改，完成顺序就不确定了。

```swift
func doSomeWork(withName name: String) {
    print(name, "start", Thread.current, CFAbsoluteTimeGetCurrent())
    var array: [Int] = []
    for i in 0..<2 {
        array.append(i)
    }
    print(name, "done", Thread.current, CFAbsoluteTimeGetCurrent())
}
```

## Qos 级别的推断和提升

队列和任务的 qos 级别并非一成不变的，它会随着根据需要进行改变。比如当添加到队列中的任务的 qos 和队列的 qos 不一致时，系统会根据一些规则来进行权重的推断。

**NSOperationQueue**  的 qos 推断和提升规则：

| 情形 | 结果 |
| --- | --- |
| 指定了 qos 的任务被添加到未指定 qos 的队列 | 队列和其中的其它任务不受影响 |
| 指定了 qos 的任务被添加到指定了 qos 的队列 | 1. 如果新入队的任务 qos 更高，则队列的 qos 也会被提升。<br/> 2. 队列中的其它较低权重的任务的 qos 也会被提升。<br/> 3. 未来新加入的较低权重的任务 qos 也会被提升。|
| 通过修改队列的 `qualityOfService` 属性来提升 qos | 1. 队列中的低权重任务都会被提升。<br/> 2. 未来加入队列的低权重任务的 qos 会被提升。|
| 通过修改队列（例如 OperationQueue）的 `qualityOfService` 属性来降低 qos | 1. 队列中的任务不会受到影响。<br/> 2. 在未来加入队列的任务，如果指定了 qos 则维持他们的权重级别，如果没有指定，则会被推断为较低的 qos |

**NSOperation** 的 qos 推断和提升规则：

| 情形 | 结果 |
| --- | --- |
| 任务未指定 qos | 1. 根据任务的 parent operation, queue, [NSProcessInfo performActivityWithOptions:reason:usingBlock:] block, 或 thread 的 qos 来推断。<br/> 2. 如果任务在主线程创建，则会被推断为 .userInitiated |
| 指定了 qos 的任务被添加到更高级别 qos 的队列中 | 任务的 qos 会被提升 |
| 包含任务的队列的 qos 被提升 | 如果队列中的任务的权重较低，则其 qos 会被提升 |
| 一个任务被指定为另一个任务的子任务 | 如果父任务的权重较低，则其 qos 会被提升 |
| 通过修改任务的 `qualityOfService` 属性来提升 qos | 1. 其子任务的 qos 也会被提升。<br/> 2. 队列中排在此任务之前的任务 qos 也会被提升。|
| 通过修改任务的 `qualityOfService` 属性来降低 qos | 1. 其子任务的 qos 不会受影响。<br/> 2. 队列的 qos 不会受影响 |

运行中的 **Operation** qos 的调整

- 修改任务的 `qualityOfService` 属性。通过这种方式修改 qos 也会影响到运行该任务的线程的 qos。
- 向正在运行的任务队列中添加 qos 更高的任务。这种方式将会提升运行中的任务的 qos。
- 使用 `addDependency: ` 函数向正在运行的任务添加 qos 更高的子任务。
- 使用 `waitUntilFinished ` 或者 `waitUntilAllOperationsAreFinished` 函数。这种方式将会提升运行中的任务的 qos 以匹配其调用者的 qos。

**GCD 队列的 qos 一旦被指定，就不能再做修改。加入其中的任务都会以此队列的 qos 来执行。**

我们可以将上面的例子做一些修改以做验证：

```swift
import Foundation

func test(queueQos: DispatchQoS) {
    let queue = DispatchQueue(label: "concurrent_queue", qos: queueQos, attributes: [.concurrent])

    let work1 = DispatchWorkItem() {
        doSomeWork(withName: "work 1", queue: queue)
    }
    let work2 = DispatchWorkItem() {
        doSomeWork(withName: "work 2", queue: queue)
    }
    let work3 = DispatchWorkItem() {
        doSomeWork(withName: "work 3", queue: queue)
    }
    let work4 = DispatchWorkItem() {
        doSomeWork(withName: "work 4", queue: queue)
    }
    queue.async(execute: work1)
    queue.async(execute: work2)
    queue.async(execute: work3)
    queue.async(execute: work4)
}

func doSomeWork(withName name: String, queue: DispatchQueue) {
    print(name, "start", Thread.current, CFAbsoluteTimeGetCurrent())
    var array: [Int] = []
    for i in 0..<2000 {
        array.append(i)
    }
    print(name, "done", Thread.current, CFAbsoluteTimeGetCurrent(), queue.qos.qosClass)
}

test(queueQos: .background)
test(queueQos: .userInteractive)

/**
 work 1 start <NSThread: 0x6000000304c0>{number = 4, name = (null)} 618579051.493967
 work 2 start <NSThread: 0x60000003d2c0>{number = 7, name = (null)} 618579051.493986
 work 3 start <NSThread: 0x60000003e900>{number = 3, name = (null)} 618579051.494019
 work 4 start <NSThread: 0x60000000a500>{number = 8, name = (null)} 618579051.494059
 work 1 start <NSThread: 0x600000031980>{number = 9, name = (null)} 618579051.49427
 work 2 start <NSThread: 0x600000028640>{number = 10, name = (null)} 618579051.494315
 work 4 start <NSThread: 0x60000000a540>{number = 12, name = (null)} 618579051.494455
 work 3 start <NSThread: 0x60000003d280>{number = 11, name = (null)} 618579051.494438
 
 work 4 done <NSThread: 0x60000000a540>{number = 12, name = (null)} 618579054.494546 userInteractive
 work 3 done <NSThread: 0x60000003d280>{number = 11, name = (null)} 618579054.511157 userInteractive
 work 2 done <NSThread: 0x600000028640>{number = 10, name = (null)} 618579054.512461 userInteractive
 work 1 done <NSThread: 0x600000031980>{number = 9, name = (null)} 618579054.514096 userInteractive
 work 2 done <NSThread: 0x60000003d2c0>{number = 7, name = (null)} 618579055.361108 background
 work 4 done <NSThread: 0x60000000a500>{number = 8, name = (null)} 618579055.361791 background
 work 1 done <NSThread: 0x6000000304c0>{number = 4, name = (null)} 618579055.364302 background
 work 3 done <NSThread: 0x60000003e900>{number = 3, name = (null)} 618579055.365978 background
 */
```




参考链接：

- [Prioritize Work with Quality of Service Classes
](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html#//apple_ref/doc/uid/TP40015243-CH39-SW1)





