# iOS 使用 Replaykit2 投屏



参考： wwdc [Live Screen Broadcast with ReplayKit](https://developer.apple.com/videos/play/wwdc2018/601/)

## 宿主app

负责发起屏幕共享，给 extention 传递鉴权信息。

```swift
import ReplayKit.broadcast
class ViewController: UIViewController {
	var broadcastPicker: RPSystemBroadcastPickerView?
	override func viewDidLoad() {
	super.viewDidLoad()
	broadcastPicker = RPSystemBroadcastPickerView(frame: kPickerFrame)
	broadcastPicker.preferredExtension = “com.your-app.broadcast.extension”
}
```

给 extention 传递鉴权信息的方式

1. extention 和 宿主app 同时 加入一个 app group
   * 通过 keychain service 来传递鉴权信息（推荐此种方式，安全）
   * 通过 UserDefaults Suit 来传递鉴权信息
2. extention 和 宿主app 同时 加入一个 keychain group
   * 通过 keychain service 来传递鉴权信息（推荐此种方式，安全）

## Broadcast upload extention

创建Broadcast upload extention&#x20;

![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16696426493641669642648633.png)

extention 中有个 `SampleHandler.swift`文件

```swift
// SampleHandler created by Xcode templates for Upload Extension
class SampleHandler: RPBroadcastSampleHandler {
// User has requested to start the broadcast
override func broadcastStarted(withSetupInfo setupInfo: [String : NSObject]?)
// User has requested to finish the broadcast
override func broadcastFinished()
// Handle the sample buffer here
override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer,
with sampleBufferType: RPSampleBufferType)
// Use details of application to annotate the broadcast
override func broadcastAnnotated(withApplicationInfo info: [String : NSObject])
}
```

初始化，通过keychain或者Userdefaults，获取鉴权信息

```swift
// Override init to read login credentials from shared keychain
class SampleHandler : RPBroadcastSampleHandler {
	override func init() {
		super.init()
		session = BroadcastSession.instance
		var credentials = KeychainAccess.getLoginCredentials()
		session.authentificate(credentials)
	}
}
```

broadcastStarted 进行鉴权，创建推流媒体引擎，初期鉴权错误

```swift
// Override broadcastStarted to prepare to receive media samples
override func broadcastStarted(withSetupInfo setupInfo: [String : NSObject]?) {
	// Verify user is logged in and there’s network connectivity
	if (session.userLoggedIn()) {
		session.createMediaEngine()
	} else {
		let userInfo = [NSLocalizedFailureReasonErrorKey : "Not Logged In"]
		let error = NSError(domain: "RPBroadcastErrorDomain", code: 401, userInfo: userInfo)
		finishBroadcastWithError(error)
	}
}
```

接收音视频，麦克风数据，编码与发送

```swift
// Both audio and video samples are handled by processSampleBuffer routine
override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer,
with sampleBufferType: RPSampleBufferType) {
	switch sampleBufferType {
		case RPSampleBufferType.video:
			var imageBuffer:CVImageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer)!
			var pts = CMSampleBufferGetPresentationTimeStamp(sampleBuffer) as CMTime
			VTCompressionSessionEncodeFrame(session, imageBuffer, pts,
			break
		case RPSampleBufferType.audioApp:
			kCMTimeInvalid, nil, nil, nil)
			// Handle audio sample buffer for app audio
			break
		case RPSampleBufferType.audioMic:
			// Handle audio sample buffer for mic audio
			break
	}
}
```

当前投屏app信息变更的通知

```swift
// Use application details to help users find your broadcast
override func broadcastAnnotated(withApplicationInfo applicationInfo: [AnyHashable : Any]) {
	var bundleIdentifier = applicationInfo[RPApplicationInfoBundleIdentifierKey]
	if (bundleIdentifier != nil) {
		session.addMetadataWithApplicationInfo(bundleIdentifier)
	}
}
```

处理用户停止屏幕采集, 删除鉴权信息，退出session等

```swift
override func broadcastFinished() {
	// User has requested to finish the broadcast.
}
```

阻止屏幕共享采集当前app的音视频信息

监听`UIScreenCapturedDidChangeNotification`

```swift
// Protecting content of your application from being captured
import UIKit
class func handleScreenCapturedChange() {
	let isScreenMirroring = UIScreen.screens.count > 1
	if (UIScreen.isCaptured && !isScreenMirroring) {
	}
	// stop audio playback and remove sensitive content from the screen
}
```

## extention 和 宿主app 通信

参考: [Configuring App Groups](https://developer.apple.com/documentation/xcode/configuring-app-groups)

1. Mach IPC
2. POSIX semaphores
3. shared memory
4. UNIX domain sockets
5. CFNotificationCenter

swift 中如何使用CFNotificationCenter, 参考： Swift - 正确使用CFNotificationCenterAddObserver 回调

```swift
//发送通知
sendNotificationForMessageWithIdentifier(identifier: "broadcastStarted")
func sendNotificationForMessageWithIdentifier(identifier : String) {
    let center : CFNotificationCenter = CFNotificationCenterGetDarwinNotifyCenter()
    let identifierRef : CFNotificationName = CFNotificationName(identifier as CFString)
    CFNotificationCenterPostNotification(center, identifierRef, nil, nil, true)
}

///通知回调
func callback(_ name : String) {
    print("received notification: \(name)")
}

///通知注册
func registerObserver() {
    let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())
    CFNotificationCenterAddObserver(CFNotificationCenterGetDarwinNotifyCenter(), observer, { (_, observer, name, _, _) -> Void in
        if let observer = observer, let name = name {
            // Extract pointer to `self` from void pointer:
            let mySelf = Unmanaged<OTCAppealVC>.fromOpaque(observer).takeUnretainedValue()
            // Call instance method:
            mySelf.callback(name.rawValue as String)
        }
    }, "broadcastFinished" as CFString, nil,.deliverImmediately)
}

//通知移除
deinit {
    let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())
    let cfName: CFNotificationName = CFNotificationName("broadcastFinished" as CFString)
    CFNotificationCenterRemoveObserver(CFNotificationCenterGetDarwinNotifyCenter(), observer, cfName, nil)
}

```

## 宿主app 读取 extention 写的 log 文件

1. 宿主app 和 extention 加入同一个 app group
2. extention 将 log 写入 group 共享目录
3. 宿主app 将 log 从共享目录读取该log文件，可以拷贝或移动到自身的沙盒中

```swift
struct Log {
    static var logFilePathInContainer: String? {
        guard let group = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "$(group_id)") else {
            return nil
        }
        guard var path = (group as NSURL).path else {
            return nil
        }
        path = (path as NSString).appendingPathComponent("xx.log")
        return path
    }
}

```
