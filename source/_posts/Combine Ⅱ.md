---
title: Combine 浅尝 Ⅱ
date: 2022-05-18 14:33:31
tags:
---
在**Combine Ⅰ**中，我们知道了**What is Combine**和**Why use Combine**，接下来会通过解释**Combine**中各个重要角色的职能去演示**How to Combine**。

## Publisher

顾名思义**发布者**，**Combine**中包括**Publisher**在内的一系列角色都是通过**protocol**去定义的，当初在Swift面世的时候apple就宣布Swift是世界上第一门面向协议编程的语言(POP)，这里也是这一思想的具体体现。

```swift
public protocol Publisher {
    associatedtype Output
    associatedtype Failure : Error
    func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
}
```

**Publisher**定义的比较简单，只有两个关联类型和一个**receive**方法，主要职责也比较简单：

1. 产生发布事件和数据
2. 准备被**Subscriber**订阅

**Output**为**Publisher**发布的值的类型，**Failure**为错误类型，对于事件流来说，**Publisher**可以发布三种类型的事件：

1. **Output** : 事件流产生了新的值
2. **Failure**:  事件流产生了错误
3. 完成： 事件流结束

对于第一种事件仅仅只是发出数据，而**Failure**和完成则是会在**Subscribers.Completion**中描述，其中**Completion**是一个enum， 其中有**finished**和**failure**分别表示着事件流的**结束**和**出错**。

```swift
    /// A signal that a publisher doesn’t produce additional elements, either due to normal completion or an error.
    @frozen public enum Completion<Failure> where Failure : Error {
        case finished
        case failure(Failure)
    }
```

## Operator

**Publisher**仅仅负责发送事件和数据，如果你想改变App内的某些"状态"，那么还需要**Operator**。

以**append**为例

```swift
    public func append(_ elements: Self.Output...) -> Publishers.Concatenate<Self, Publishers.Sequence<[Self.Output], Self.Failure>>
```

他接受的参数类型必须要与**上游Publisher**的**Output**类型一致，并生成一个新的**Publisher**传递下去。它的功能也是顾名思义，将特定数据与上游Publisher的Output数据进行一个拼接。

```swift
let dataElements = (0...10) // 0 1 2 3 4 5 6 7 8 9 10
cancellable = dataElements.publisher
    .append(0, 1, 255) // 需要拼接的数据
    .sink { print("\($0)", terminator: " ") }
         // Prints: "0 1 2 3 4 5 6 7 8 9 10 0 1 255"
```

**Combine**提供很多方便的**Operator**， 像是使用频率很高的**map**、**flatmap**等等，它们命名都比较简单并且也很容易知道其对应的职责。

在响应式编程中，绝大部分逻辑都发生在数据处理和变形中。每个**Operator**的模式都一样，接收上游Publisher的数据，进行变形、处理，然后再产生一个新的Publisher传给下游。

**Subscriber**

事件的发生，数据的处理都已经有了，接下来需要去“**接收**“、”**消费**“上游的数据，这时候就需要**Subscriber**订阅者。

和**Publisher**类似， **Subscriber**也是一个抽象的协议，它定义了某个类型想要成为订阅者应该要满足的条件。

```swift
public protocol Subscriber : CustomCombineIdentifierConvertible {
    associatedtype Input
    associatedtype Failure : Error
    func receive(subscription: Subscription)
    func receive(_ input: Self.Input) -> Subscribers.Demand
    func receive(completion: Subscribers.Completion<Self.Failure>)
}
```

其中的**Input**和**Failure**分别表示订阅者能够接收的事件流数据类型和错误类型，想要订阅某个**Publisher**，**Subscriber**中的**Input**和**Failure**必须要跟**Publisher**中的**Output**和**Failure**一致。

**Combine**中也定义了很多常用的**Subscriber**，这里用**sink**来举例。

```swift
public func sink(receiveCompletion: @escaping ((Subscribers.Completion<Self.Failure>) -> Void), receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
```

**sink**方法中你可以提供两个**closure**，**receiveCompletion**用来接收**finish**或者**failure**事件，而**receiveValue**就是接受上游的**Output**值。

**Subscriber**的作用相当于一座桥梁，将响应函数式代码终结并桥接到指令式代码中(Closure中)。

## 总结

从**Publisher**->**Operator**->**Subscriber**，从事件发布->变形->订阅接收，随着事件的发生，App的状态也随之改变，这些就是响应式框架**Combine**的基本架构。