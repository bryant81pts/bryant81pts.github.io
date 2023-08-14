---
title: iOS 14相册相关变更要点
date: 2020-10-07 14:36:20
tags:
- iOS
---
## 功能预览

在iOS 14之前，一旦允许访问相册，App是可以通过Photos框架 APIs**读取**/**写入**你的手机相册。

<img src="https://i.loli.net/2020/10/07/8ZTHxosPkJfepE5.png" width="400px" />

而在iOS 14中，则新增了一种模式叫做**Limited photo library access**，你可以将其视为PhotoKit API的一层过滤器，App只能访问用户选定的媒体资料而不是整个媒体库。



<img src="https://i.loli.net/2020/10/07/3O1CQLw6ogvHmKE.png" width="400px" />

当用户修改了他们的选择时，App将会自动收到通知，这样就可以更新UI来反应用户的操作。

这次新加的模式将会影响所有使用PhotoKit的iOS 14应用，已经上架的应用也不例外。

## 授权弹窗

当用户首次打开App内相册时通常都会遇到这个弹窗。

<img src="https://i.loli.net/2020/10/08/SDCgMTtRmwBXNl3.png" width="400px" />

- **Select Photos**: 有限访问权限，点击后系统会要求用户选择提供给App的媒体资源。

- **Allow Access to All Photos**: 完全访问权限。

- **Don't Allow**: 不允许访问权限。

  

  

## 有限访问

点击**Select Photos**后，会弹出一个系统级的媒体资源选择器提供给用户选择。

<img src="https://i.loli.net/2020/10/08/mtNqS7YE5nk8Jdw.png" width="400px" />

<img src="https://i.loli.net/2020/10/08/nahWog2v63fFX4S.png" width="400px" />

## 修改已选择项

- 可以在设置中找到对应App，修改已选中的媒体资源。

<img src="https://i.loli.net/2020/10/08/y7XtdkFirUTR9hv.png" width="400px" />

<img src="https://i.loli.net/2020/10/08/3wKZHbCdJtpOBRN.png" width="400px" />

<img src="https://i.loli.net/2020/10/08/5o2SxnftZdQTUyK.png" width="400px" />

<img src="https://i.loli.net/2020/10/08/5axYfUg8T7D3QbH.png" width="400px" />

- 冷启动首次在App中使用相册功能时，也可以修改已选的媒体资源。

  <img src="https://i.loli.net/2020/10/08/t3USKMEGszqmTcJ.png" width="400px" />

  ## Photos框架API变更

  - 在**PHAuthorizationStatus**(授权状态)枚举中增加了**limited**枚举值。

    ```objective-c
    public enum PHAuthorizationStatus : Int {
    
        
        @available(iOS 8, *)
        case notDetermined = 0 // User has not yet made a choice with regards to this application
    
        @available(iOS 8, *)
        case restricted = 1 // This application is not authorized to access photo data.
    
        // The user cannot change this application’s status, possibly due to active restrictions
        //   such as parental controls being in place.
        @available(iOS 8, *)
        case denied = 2 // User has explicitly denied this application access to photos data.
    
        @available(iOS 8, *)
        case authorized = 3 // User has authorized this application to access photos data.
    
        @available(iOS 14, *)
        case limited = 4 // User has authorized this application for limited photo library access. Add PHPhotoLibraryPreventAutomaticLimitedAccessAlert = YES to the application's Info.plist to prevent the automatic alert to update the users limited library selection. Use -[PHPhotoLibrary(PhotosUISupport) presentLimitedLibraryPickerFromViewController:] from PhotosUI/PHPhotoLibrary+PhotosUISupport.h to manually present the limited library picker.
    }
    ```

    

  - 增加**PHAccessLevel**枚举值，分别代表两种相册访问方式。

    ```objective-c
    public enum PHAccessLevel : Int {
    
        
        @available(iOS 8, *)
        case addOnly = 1
    
        @available(iOS 8, *)
        case readWrite = 2
    }
    ```

  - 增加两个替换性API，分别用于查询相册授权状态API和请求授权API，增加了**accessLevel**的传入参数，值得注意的是，旧的API已经被打上**deprecated**标签。

    ```objective-c
        /// Replaces \c +authorizationStatus to support add-only/read-write access level status
        @available(iOS 14, *)
        open class func authorizationStatus(for accessLevel: PHAccessLevel) -> PHAuthorizationStatus
    
        @available(iOS 14, *)
        open class func requestAuthorization(for accessLevel: PHAccessLevel, handler: @escaping (PHAuthorizationStatus) -> Void)
    
        
        /// Deprecated and replaced by authorizationStatusForAccessLevel:, will return \c PHAuthorizationStatusAuthorized if the user has chosen limited photo library access
        @available(iOS, introduced: 8, deprecated: 100000)
        open class func authorizationStatus() -> PHAuthorizationStatus
    
        @available(iOS, introduced: 8, deprecated: 100000)
        open class func requestAuthorization(_ handler: @escaping (PHAuthorizationStatus) -> Void)
    ```

Apple在iOS 14进行相册权限的改动是为了让开发者重新审视自己的App是否真的需要用户的完全相册权限，旨在保护用户的隐私。Apple认为只有以下类型的App才可能需要用户相册的完全权限。

<img src="https://i.loli.net/2020/10/08/YUR2iwAdabqlnLO.png" width="400px" />

## PHPicker

Apple建议在readOnly的情景下使用新推出的**PHPicker**，所以在**PHAccessLevel**中并没有**readOnly**的枚举值，iOS 14之前**PHPicker**的前任是**UIImagePickerController**。

<img src="https://i.loli.net/2020/10/08/IBhOdSQoRzvrGEe.png" width="400px" />

相比于自定义相册，**PHPicker**有几个优势:

- 独立进程，不影响App性能。
- 默认多选。
- 默认支持搜索。
- 没有授权弹窗。
- App不需要直接访问媒体库来显示选择器。
- 只提供用户选择的照片和视频。

劣势就是无法定制化，如果需要定制UI还是需要使用Photos框架自己写一个图片选择器。

## 隐藏选择图片弹窗

在上面说过，用户选择了**limited**模式后，App第一次冷启动时还是会显示一个弹窗。

<img src="https://i.loli.net/2020/10/08/t3USKMEGszqmTcJ.png" width="400px" />

Apple建议关闭自动弹窗，建议在App其他页面提供修改已选媒体资源的入口，例如设置页。

若要关闭此自动弹窗，需要改动**info.plist**，增加一个新的key: **PHPhotoLibraryPreventAutomaticLimitedAccessAlert**，将其设置为**YES**即可。



参考WWDC Session:

[Handle the Limited Photos Library in your app](https://developer.apple.com/videos/play/wwdc2020/10641/)

[Meet the new Photos picker](https://developer.apple.com/videos/play/wwdc2020/10652)

