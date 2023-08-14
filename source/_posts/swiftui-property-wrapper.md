---
title: SwiftUI中的常用的属性包装器
date: 2021-02-28 14:40:03
tags:
- SwiftUI
- iOS
---
整理一些SwiftUI中常用的属性包装器(Property Wrappers).

- `@Binding`用于标记其他View持有的值类型数据(例如父View传入子View的参数)，改变本地binding的值会同时改变原始的值。

  ```swift
  struct PlayerView: View {
      var episode: Episode
      @State private var isPlaying: Bool = false
  
      var body: some View {
          VStack {
              Text(episode.title)
              Text(episode.showTitle)
            // isPlaying作为参数传给子View
              PlayButton(isPlaying: $isPlaying)
          }
      }
  }
  
  struct PlayButton: View {
      @Binding var isPlaying: Bool
  
      var body: some View {
          Button(action: {
            // 改变local的isPlaying也同时会改变PlayerView中的isPlaying
              self.isPlaying.toggle()
          }) {
              Image(systemName: isPlaying ? "pause.circle" : "play.circle")
          }
      }
  }
  ```

- `@Enviroment`用于读取系统的配置，例如系统当前是否为黑夜模式。

  ```swift
  @Environment(\.colorScheme) var colorScheme: ColorScheme
  //如果colorScheme变了，SwiftUI会更新使用到这些值的View, 适用于如果用户手动改变系统主题偏好设置。
  if colorScheme == .dark { 
      DarkContent()
  } else {
      LightContent()
  }
  
  ```

- `@EnviromentObject`用于标记我们注入`enviroment`中的共享对象，例如View A使用了`@EnviromentObject`标记了一个对象属性Object_A，那么View A中的所有View都可以在这个'环境'中访问到这个Object_A, 而不需要以参数的形式传递。

- `@ObservedObject` 用于标记外部创建的类的实例对象，并且该类遵循**ObservableObject**这个协议。`@ObservedObject`与`@StateObject`有点类似，除了`@ObservedObject`用于外部创建的对象，因为SwiftUI有可能会意外销毁这些标记的对象，例如页面重绘，而`@StateObject`就不会。

  ```swift
  class Order: ObservableObject {
      // ObservableObject 的类中通常会配合 @Published使用
    // 当items发生变化时会通知到ContentView
      @Published var items = [String]()
  }
  
  struct ContentView: View {
      @ObservedObject var order: Order
  
      var body: some View {
        if order.items.count > 10 {
          // items变化时会按需更新使用到的View 
        }
      }
  }
  ```

- `@StateObject`通常用于View中创建的类的实例对象，当View更新的时候，该实例对象不会被销毁重新创建。

  ```swift
  class User: ObservableObject {
      var username = "@twostraws"
  }
  struct ContentView: View {
    // user在View重绘的时候不会重新生成
      @StateObject var user = User()
  
      var body: some View {
          Text("Username: \(user.username)")
      }
  }
  ```

- `@Published`用于标记**ObservableObject**中需要观察的属性，SwiftUI会自动监控这些属性，并且在值改变时更新与其关联的View。

- SwiftUI会管理所有用`@State`标记的属性，当`@State`标记的属性发生变化时，关联的View会重绘。