---
title: KMP数据持久化之Key-Value
date: 2023-05-14 14:11:34
tags:
- KMP
---
常用的数据持久化方式有3种

1. Key-Value
2. DB
3. File system

## Key-Value

iOS中的`UserDefaults`和Android中的`SharedPreferences`通常用于存储用户的偏好设置，这两种就是各自平台典型的Key-Value存储的实现。

既然上述两种Key-Value存储方式是平台限定的，在KMP中当然可以用`expect/actual`的方式，定义一个统一的接口，具体实现由各自平台特性去完成，这也意味着要写两份代码，并且还有很多的样板代码。

Kotlin提供了个叫做`Multiplatform Setting`的库可以减少上述的工作量。

#### 设置Multiplatform Setting

在shared模块中的build.gradle.kts文件中为commonMain添加依赖
```kotlin
implementation("com.russhwolf:multiplatform-settings:${rootProject.extra["settingsVersion"]}")
```
接着在androidApp模块的build.gradle.kts文件中也添加相同的依赖。

假设项目中用了依赖注入库Koin，则需要在commonMain中设置Koin的文件中用添加一个expect常量

```kotlin
expect val myModule: Module
```

同样，在koin的初始化方法`initKoin`中的`startKoin`也需要添加对应的Module。
```kotlin
startKoin {
  modules(
    xxxModule,
    xxxModule,
    myModule, 
  )
}
```
接下来就到各个平台中提供对应的实现(`actual`)。

#### Android

```kotlin
actual val myModule  = module {
    // single表示单例
     single<Settings> {
        AndroidSettings(get())
     }
}
```

这里的AndroidSettings是安卓平台对`Settings`的实现，并需要一个`SharedPreferences`的实例来构造。Koin在遇到`get()`函数的时候会根据提供的声明上下文(AndroidSettings构造方法)去寻找匹配的参数(SharedPreferences实例)。

接下来在Anroid平台初始化Koin的函数`initKoin`中添加`SharedPreferences`的获取依赖函数即可。

```kotlin
//设置SharedPreferences需要一个Contenxt实例， 这里是在声明使用app实例来作为context
single<Context> { this@TestApp }

single<SharedPreferences> {
     //使用gert()函数获取Context的实例并创建一个SharedPreferences的实例
    get<Context>().getSharedPreferences(
        "TestApp", 
        Context.MODE_PRIVATE
      )
}
```

#### iOS 

在iOS平台Koin初始化文件中同样添加
```kotlin
actual val myModule = module { }
```
在Koin的初始化方法`initialize`通过添加参数进行注入UserDefaults的实例
```kotlin
fun initialize(
  userDefaults: UserDefaults,
): KoinApplication = initKoin(
  appModule = module {
    single<Settings> {
      AppleSettings(userDefaults)
    }
  }
)
```
与Android不同的是，Settings在iOS平台的实现为`AppleSettings`，它需要UserDefaults的实例来初始化。

#### 使用

`Settings`提供了许多通俗易懂的API, 使用起来几乎无门槛。

<img src="https://s2.loli.net/2023/01/02/bDH8Qlo5amPdp6V.png" width="400px" />

```kotlin
val userIdKey = "USERID_KEY"
val saved = settings.getString(userIdKey)
settings.putString(userIdKey, "xxxx")
```

