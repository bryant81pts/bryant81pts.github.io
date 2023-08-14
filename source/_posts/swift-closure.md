---
title: Swift中闭包的本质
date: 2021-01-03 14:38:42
tags:
- 零碎知识
- iOS
---
- Swift中闭包相关的概念

1. 一个函数和它所捕获的变量\常量环境组合起来，称作闭包.

2. 一般指定义在函数内部的函数.

3. 一般它捕获的是外层函数的局部变量\常量.

- 利用汇编理解闭包本质

```swift
typealias Fn = (Int) -> Int

func getFn() -> Fn {
    var num = 0
    func plus(_ i: Int) -> Int {
        num += i
        return num
    }
    return plus
}

var fn = getFn()

print(fn(1)) //1
print(fn(2)) //3
print(fn(3)) //6
print(fn(4)) //10

```

以上的示例代码中闭包`plus`内部使用了外部变量`num` 并进行一系列操作，而后将`getFn()`赋值给变量`fn`，再通过传入不同的参数执行`fn`， 得到的结果为1， 3，6，10说明`num`是被`plus`“捕获”了，但是`plus`究竟是如何捕获`num`的还是要借助断点+汇编代码去一探究竟。

首先先看`plus没有访问num`变量时的情况，我们将断点打在`return plus`处

```swift
func getFn() -> Fn {
    var num = 0
    func plus(_ i: Int) -> Int {
        return i
    }
    return plus //断点处
}
```

```assembly
TestSwift`getFn():
    0x100003e90 <+0>:  pushq  %rbp
    0x100003e91 <+1>:  movq   %rsp, %rbp
    0x100003e94 <+4>:  movq   $0x0, -0x8(%rbp)
->  0x100003e9c <+12>: leaq   0xd(%rip), %rax           ; plus #1 (Swift.Int) -> Swift.Int in TestSwift.getFn() -> (Swift.Int) -> Swift.Int at main.swift:14
    0x100003ea3 <+19>: xorl   %ecx, %ecx
    0x100003ea5 <+21>: movl   %ecx, %edx
    0x100003ea7 <+23>: popq   %rbp
    0x100003ea8 <+24>: retq   

```

由断电的命令`leaq`可以得知，其实`getFn()`是把0xd+rip的地址 值扔进了`rax`中。

因为返回值一般会存放在`rax`寄存器中，因为`getFn`函数返回的是`plus`闭包，所以`rax`中是存放着`plus`的地址，`rax`的值可以通过`lldb`命令在`xcode`的控制台中获取。

```assembly
(lldb) si
(lldb) register read rax
     rax = 0x0000000100003eb0  TestSwift`plus #1 (Swift.Int) -> Swift.Int in TestSwift.getFn() -> (Swift.Int) -> Swift.Int at main.swift:14
(lldb) 
```

`rip`的地址`0x100003ea3`加上`0xd`正好是`0x0000000100003eb0`

接下来看看`plus`访问了`num`的情况, 断点还是打在`retrun plus`。

```swift
func getFn() -> Fn {
    var num = 0
    func plus(_ i: Int) -> Int {
        num += i
        return num 
    }
    return plus
}
```

这时候的汇编代码就复杂了许多。

```assembly
TestSwift`getFn():
    0x100003cf0 <+0>:  pushq  %rbp
    0x100003cf1 <+1>:  movq   %rsp, %rbp
    0x100003cf4 <+4>:  subq   $0x20, %rsp
    0x100003cf8 <+8>:  movq   $0x0, -0x8(%rbp)
    0x100003d00 <+16>: leaq   0x321(%rip), %rdi
    0x100003d07 <+23>: movl   $0x18, %esi
    0x100003d0c <+28>: movl   $0x7, %edx
    0x100003d11 <+33>: callq  0x100003ee6               ; symbol stub for: swift_allocObject
    0x100003d16 <+38>: movq   %rax, %rcx
    0x100003d19 <+41>: addq   $0x10, %rcx
    0x100003d1d <+45>: movq   %rcx, -0x8(%rbp)
    0x100003d21 <+49>: movq   $0x0, 0x10(%rax)
->  0x100003d29 <+57>: movq   %rax, %rdi
    0x100003d2c <+60>: movq   %rax, -0x10(%rbp)
    0x100003d30 <+64>: callq  0x100003f0a               ; symbol stub for: swift_retain
    0x100003d35 <+69>: movq   -0x10(%rbp), %rdi
    0x100003d39 <+73>: movq   %rax, -0x18(%rbp)
    0x100003d3d <+77>: callq  0x100003f04               ; symbol stub for: swift_release
    0x100003d42 <+82>: leaq   0x177(%rip), %rax         ; partial apply forwarder for plus #1 (Swift.Int) -> Swift.Int in TestSwift.getFn() -> (Swift.Int) -> Swift.Int at <compiler-generated>
    0x100003d49 <+89>: movq   -0x10(%rbp), %rdx
    0x100003d4d <+93>: addq   $0x20, %rsp
    0x100003d51 <+97>: popq   %rbp
    0x100003d52 <+98>: retq   

```

里面有个关键点是使用了`swift_allocObject`这个函数，使用了这个函数以为这开辟了堆空间，所以可以直接联想这个新开辟的堆空间是跟`num`有关的，或许是拿来存放`num`的。这也解释了为什么`num`没有在`getFn()`作用域结束时没有销毁，所以`num`的值才能累加起来。

现在就来看看`swift_allocObject`分配的堆空间内是否存放着`num`，需要在分配完毕时打汇编断点并打印`rax`的值， 此时`rax`的值就是堆空间返回的地址。

```assembly
   0x100003d11 <+33>: callq  0x100003ee6               ; symbol stub for: swift_allocObject
-> 0x100003d16 <+38>: movq   %rax, %rcx
```
读取到`rax`的值

```assembly
(lldb) register read rax
     rax = 0x00000001018067d0
(lldb) 
```

此时将代码断点`return plus`的断点去掉，并新加`return num`的断点，目的是观察`num`值改变时，堆空间有没有变化，如果有变化就能证明`num`确实是在堆空间内。 

```assembly
(lldb) register read rax
     rax = 0x00000001018067d0
(lldb) x/5xg 0x00000001018067d0
0x1018067d0: 0x0000000100004028 0x0000000200000003
0x1018067e0: 0x0000000000000001 0x00007fff7f0bc658
0x1018067f0: 0x0000000000000000
(lldb) 
```

`x/数量+格式+字节大小 地址`读取内存中的值。

因为此时调用`fn(1)`, `num`初始化的值为`1`,  在`0x1018067e0`地址中的`8`个字节也为`1`(`0x0000000000000001`)，于是继续程序的执行，看下次到达断点时那8个字节会不会因为`fn(2)`的调用而变成`3`。

```assembly
(lldb) x/5xg 0x00000001018067d0
0x1018067d0: 0x0000000100004028 0x0000000200000003
0x1018067e0: 0x0000000000000001 0x00007fff7f0bc658
0x1018067f0: 0x0000000000000000
1
(lldb) x/5xg 0x00000001018067d0
0x1018067d0: 0x0000000100004028 0x0000000200000003
0x1018067e0: 0x0000000000000003 0x00007fff7f0bc658
0x1018067f0: 0x00007fff88997f98
(lldb) 
```

`0x1018067e0`地址的`8`个字节确实变成了`3`，也证明了`num`确实存在新开辟的堆空间中。



## 总结

可以把闭包想象成是一个类的实例对象

内存在堆空间

捕获的局部变量\常量就是对象的成员

组成闭包的函数就是类内部定义的方法
