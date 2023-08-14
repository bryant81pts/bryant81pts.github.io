---
title: LiveActivity.md
date: 2022-12-20 14:26:15
tags:
- iOS
---
Live Activities是iOS 16比较亮眼的新功能之一，它允许App在锁屏界面或者iPhone 14 Pro系列的Dynamic Island上展示“当前正在进行的活动“的动态信息，比如用了饿了么点了外卖后，锁屏界面就会显示你的外卖进度：

<img src="https://s2.loli.net/2023/01/02/xHOEuTg7pKDiSqX.png" width="400px" />

或者用直播8关注球赛，直接在锁屏界面摸鱼看文字直播：

<img src="https://s2.loli.net/2023/01/02/rFJmL9Gq31hSsac.png" width="400px" />
直接省去了进入App才能查看进度状态的步骤。


### Live Activities基本概念

每个Live Activity的状态可以最多持续更新8个小时，前提是App或者用户没有去手动停止。 一旦超过了8小时的限制，系统会自动杀掉对应的Activiity，此时Live Activity会自动从Dynamic Island消失，但是在锁屏界面会最多保持12个小时。

Live Activity需要用到WidgetKit来实现相关功能，Home Screen Widgets和 Live Activity之间也可以共用代码，但是Live Activity并不是Widget，Widget用timeline来更新界面数据等，而更新Live Activity的更新需要在app内借助ActivityKit框架。

### 数据

Live Activity包含两种类型的数据：静态数据和动态数据。以球赛为例，静态数据就是两队的队名、logo等；动态数据就是即时比分、文字直播信息等。需要注意的是每次更新的数据大小不能超过4kb的限制。

AcitivityKit需要我们通过实现ActivityAttributes protocol来描述数据类型，可以想象成data model。

```swift
import ActivityKit

struct NBAGame: ActivityAttributes {
    public typealias GameState = ContentState

    public struct ContentState: Codable, Hashable {
        var score1: Int
        var score2: nt
    }

    var team1: String
    var team2: String
}
```

### 启动Live Activity

app必须要在前台启动Live Activity,  并且在info.plist里把NSSupportLiveActivities设置为YES。
```swift
static func request(
    attributes: Attributes,
    content: ActivityContent<Activity<Attributes>.ContentState>,
    pushType: PushType? = nil
) throws -> Activity<Attributes>

if ActivityAuthorizationInfo().areActivitiesEnabled {
    do {
        // request函数有3个参数
        // attributes 描述静态数据
        // contentState 描述随着时间变化的动态数据
        // pushType 默认为nil，表示通过update方法来更新数据，如果需要通过推送更新则需要传入token
        activity = try Activity.request(
            attributes: NBAGame(
                team1: "Lakers",
                team2: "Celtics"
            ),
            contentState: .init(
                score1: 0,
                score2: 0
            )
        )
    } catch (let error) {
    }
}
```

### 更新Live Activity

虽然启动Live Activify需要在前台进行，但是更新或者停止移除等操作可以通过Background Tasks执行。

```swift
// 更新
func update(_ content: ActivityContent<Activity<Attributes>.ContentState>) async

Task {
    await activity.update(using: .init(score1: 48, score2: 50))
}
```

```swift
// 停止
// ActivityUIDismissalPolicy提供了ui延迟移除的选项，最多可以延迟4小时。
func end(
    _ content: ActivityContent<Activity<Attributes>.ContentState>?,
    dismissalPolicy: ActivityUIDismissalPolicy = .default
) async

Task {
    await activity.end(
        using: .init(score1: 120, score2: 90),
        dismissalPolicy: .immediate
    )
}
```

### 展示Live Activity

```swift
@main
struct NBAGameActivityWidget: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: NBAGame.self) { context in
            // 配置锁屏的小组件
            SomeActivityView()...
        } dynamicIsland: { context in
            // 配置Dynamic Island
            DynamicIsland {
                
            } compactLeading: {
                
            } compactTrailing: {
                
            } minimal: {
                
            }
        }
    }
}
```
如果app已有Widget并想增加Live Activity的话需要在现有的WdigetBundle里添加

```swift
@main
struct PizzaDeliveryWidgets: WidgetBundle {
    var body: some Widget {
        FavoritePizzaWidget()

        if #available(iOS 16.1, *) {
            NBAGameActivityWidget()
        }
    }
}
```
