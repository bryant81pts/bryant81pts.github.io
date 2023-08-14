---
title: Combine 浅尝 Ⅰ
date: 2022-05-17 14:30:27
tags:
- iOS
- Combine
---
## old tricks

对于一个iOS App，99%会有如下的异步的编程场景：

- 请求API返回JSON
- 网络加载图片并下载持久化

传统的Cocoa和UIKit框架则提供了异步API三巨头：

- **Block\Closure**: 调用耗时方法提供Block\Closure接受回调，耗时方法在非主线程中执行，完成时执行Block\Closure通知调用者任务完成。例子：**URLSession**的网络请求方法**dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask**中的**completionHandler**
- **Protocol-Delegate**: 定义一个protocol和其中的方法来描述异步行为中可能发生的结果，使用时创建遵循该protocol并实现其中的方法的delegate，异步行为完成后会检查是否有符合条件的delegate，并会尝试执行protocol中的方法来通知行为的完成。例子：**UITableView**的**UITableViewDataSource**和**UITableViewDelegate**， 用于描述UITableView该如何展示。同时**URLSession**中除了**Block\Closure**回调方式也有基于**Protocol-Delegate**的回调方式。
- **Notification**: 可以通过**NotificationCenter**发送一个**Notification**， 这个**Notification**会被注册的观察者所接收到并且执行相应的代码。这也是**Cocoa**中常见的观察者模式。例子：键盘将要展示的 **UIKeyboardWillShowNotification**。

这三种方式我认为并没有优劣之分，只有各自擅长的使用场景之分，如果事件比较简单不需要关心过程，则使用**Block-Closure**，如果需要关心更多的过程与细节，则使用**Protocol-Delegate**比较合理。如果触发时机不规律则会使用**Notification**。

## problems

如果只有一两个异步操作和状态时，对这些状态的管理不会特别困难。但在实际开发中，往往需要混合调用多个不同类型的异步 API，而有时候这些不同的异步 API 会操作和设置同一个状态。因为无法确定这些异步 API 的调用顺序，这会让状态的维护成本变得很高，进而导致逻辑复杂，导致Bug。

解决这些有关联并且逻辑复杂的问题时，通常会寻找这些问题的共同点，在上一层进行抽象总结对各个问题的通用方案。无论是**Block\Closure**还是**Protocol-Delegate**或者是**Notification**，其实都是提前写好逻辑，去相应未来发生的事件。**Combine**框架提供了另外一种方式去模糊不同异步API的区别，让事件发生这一核心概念暴露出来，在异步操作中某个事件发生时，把这个事件和与其相关的数据**publish**出来。而对这个事件感兴趣的可以**subscribe**这个事件，来进行后续操作。

这种方式接受事件的行为与**Notification**有那么一点类似，但是最大的区别就是后续处理数据的方式，**Combine**框架中事件和与其相关的数据**publish**后，我们可以在事件流中按需处理\改变这些事件和数据。事件中的数据不符合需要的格式，我们可以进行一系列变形操作。事件中的某些数据并不需要，我们可以将其过滤。A事件的数据需要与B事件的数据合并，我们可以先把A事件数据“暂存”起来，等待B事件数据后再进行合并。随后在末端会有个**subscriber**来接受事件和数据的最终形态。

## new tricks

正如**Combine**其名，通过**组合**事件处理操作来自定义处理异步操作。响应式编程并不是什么新概念，OC的**ReactiveCocoa**和Swift的**RxSwift**，虽然他们的实现略有差异，但是核心思想都是一致的，而**Combine**这个“正规军”的出现，则从系统层级对响应式编程提供了完备的支持。**Combine**中有三个重要的角色**Publisher**、**Operator**和**Subscriber**, 下一篇将会对这三个角色进行详细说明。

## 


