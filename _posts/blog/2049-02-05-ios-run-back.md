---
layout: post
title: app 后台运行
description: Background Execution
category: blog
tag: ios
---

## 后台执行

原文链接 [Background Execution](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/BackgroundExecution/BackgroundExecution.html#//apple_ref/doc/uid/TP40007072-CH4-SW1)

当用户当前不使用你的app时，app此时就处于后台状态。对于大多数app来说，后台状态只是app到挂起状态的一个暂时状态，即它是一个过渡状态。让app处在挂起状态能节约手机电池的电量，也能让让出更多地系统资源给当前正在被使用的app.

大多数的app可以很轻松的到挂起状态，但是也有一些特殊需求的app需要在后台持续运行。比如计步助手、音频播放app、有下载功能的app。当你需要让app在后台状态下保持运行的时候，ios可以帮助你高效的完成，并且不会消耗过多的系统资源和用户电池电量。ios提供了以下三种情况： 
- Apps that start a short task in the foreground can ask for time to finish that task when the app moves to the background.在app处在前台时启动一个短任务来申请一段时间完成这个任务。这段时间是app处于后台的时间。 
- Apps that initiate downloads in the foreground can hand off management of those downloads to the system, thereby allowing the app to be suspended or terminated while the download continues. app在前台启动下载并且把下载管理权交给系统，从而实现app被挂起或终止时而下载任务继续。 
- Apps that need to run in the background to support specific types of tasks can declare their support for one or more background execution modes. 需要在后台运行，以支持特定类型的任务的应用可以申报一个或多个背景执行模式的支持。

通常情况下，都是避免在后台做任何的工作除非这样做能提高用户的体验。app转到后台状态时，要么使用户打开了其它app，要么是用户锁住了屏幕，此时用户没有使用此app。在这两种情况下，用户都是在告诉系统他此时不需要app再做任何有意义的工作。如果你执意让app这样的情况下运行，只会耗尽手机电池，并可能导致用户强制退出你的程序。所以你要特别注意你在后台所做的。

## Executing Finite-Length Tasks （执行有限长度的任务）

app进入后台尽快的会进入一个安静的状态，所以他会被系统挂起。如果你的应用需要执行一个中小型的任务从而需要一点额外的时间去完成这个任务时，你可以调用UIApplication对象的`beginBackgroundTaskWithName:expirationHandler: `或`beginBackgroundTaskWithExpirationHandler:`方法申请一些额外的时间。调用这两个方法中的任何一个方法就可以暂时延迟app的挂起，给它一些额外的时间来完成任务。在你的任务完成时，你必须调用 `endBackgroundTask: `方法让系统知道应用完成了任务，应用可以挂起了。

所有调用` beginBackgroundTaskWithName:expirationHandler:`或者`beginBackgroundTaskWithExpirationHandler:` 方法都会生成一个和相应任务相关联的一个token. 当你的应用完成任务时它必须调用`endBackgroundTask:` 方法相应token让系统知道任务完成。调用`endBackgroundTask:` 失败将会导致你的应用终止。如果你在启动任务的时候提供了一个过期处理，系统就会调用这个处理，这是系统给你最后一次机会来避免应用终止。

你不用等到应用转到后台的时候指派后台任务，一个更有效的方法就是你在开始后台任务前调用`beginBackgroundTaskWithName:expirationHandler: `or `beginBackgroundTaskWithExpirationHandler:` 方法，在任务完成时调用 `endBackgroundTask: `。你甚至可以在应用程序的前台执行这种模式。

eg: 
Listing 3-1 shows how to start a long-running task when your app transitions to the background. In this example, the request to start a background task includes an expiration handler just in case the task takes too long. The task itself is then submitted to a dispatch queue for asynchronous execution so that the applicationDidEnterBackground: method can return normally. The use of blocks simplifies the code needed to maintain references to any important variables, such as the background task identifier. The bgTask variable is a member variable of the class that stores a pointer to the current background task identifier and is initialized prior to its use in this method.

```
- (void)applicationDidEnterBackground:(UIApplication *)application
{
    bgTask = [application beginBackgroundTaskWithName:@"MyTask" expirationHandler:^{
        // Clean up any unfinished task business by marking where you
        // stopped or ending the task outright.
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];

    // Start the long-running task and return immediately.
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{

        // Do the work associated with the task, preferably in chunks.

        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    });
}
```

Note: Always provide an expiration handler when starting a task, but if you want to know how much time your app has left to run, get the value of the backgroundTimeRemaining property of UIApplication. 
注释：通常在任务开始的时候提供了过期处理的handle，但是如果你想看一下你的应用还剩多少时长进入挂起状态，你可以打印UIApplication的backgroundTimeRemaining属性。

In your own expiration handlers, you can include additional code needed to close out your task. However, any code you include must not take too long to execute because, by the time your expiration handler is called, your app is already very close to its time limit. For this reason, perform only minimal cleanup of your state information and end the task. 
在你的任务过期处理中，你可以包含额外的代码来关闭你的任务。但是，你所包含的代码不能花费太多的时间去执行，因为当你的过期处理handle被调用是，你的应用就被限制为很少的执行时间。由于此原因，你在过期handle中只执行你用时较短的清理状态信息和结束任务的代码。

### 示例

#### 默认选项创建的app

我们首先检测一下我们默认创建APP选项情况下的状态变化。即我们直接在`ViewController`启动一个timer打印一些信息，然后点击app让其进入后台，看看日志打印情况。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    
    if (_executeTimer == nil) {
        _executeTimer = [NSTimer scheduledTimerWithTimeInterval:1.0f
                                                          target:self
                                                        selector:@selector(info)
                                                        userInfo:nil
                                                         repeats:YES];
    }
}

- (void)info
{
    NSLog(@"app running..");
}
```

日志打印：

```
2018-02-05 15:34:50.936389+0800 Long_Task[6658:4831578] -[AppDelegate application:didFinishLaunchingWithOptions:],25
2018-02-05 15:34:50.956714+0800 Long_Task[6658:4831578] refreshPreferences: HangTracerEnabled: 0
2018-02-05 15:34:50.956782+0800 Long_Task[6658:4831578] refreshPreferences: HangTracerDuration: 500
2018-02-05 15:34:50.956806+0800 Long_Task[6658:4831578] refreshPreferences: ActivationLoggingEnabled: 0 ActivationLoggingTaskedOffByDA:0
2018-02-05 15:34:50.956896+0800 Long_Task[6658:4831578] -[AppDelegate applicationDidBecomeActive:],63
2018-02-05 15:34:51.941874+0800 Long_Task[6658:4831578] app running..
2018-02-05 15:34:52.942148+0800 Long_Task[6658:4831578] app running..
2018-02-05 15:34:53.942072+0800 Long_Task[6658:4831578] app running..
2018-02-05 15:34:54.941343+0800 Long_Task[6658:4831578] app running..
2018-02-05 15:34:55.353138+0800 Long_Task[6658:4831578] -[AppDelegate applicationWillResignActive:],33
2018-02-05 15:34:55.942276+0800 Long_Task[6658:4831578] app running..
2018-02-05 15:34:56.433431+0800 Long_Task[6658:4831578] -[AppDelegate applicationDidEnterBackground:],40
Message from debugger: Terminated due to signal 9
```
由此可见，我们直接用Xcode创建一个app，然后点击home键后，app直接进入到挂起状态。系统也不会杀死这个app进程，而是当app再次回到活跃状态时，接着执行挂起的代码。

#### app在后台执行有限长度的任务

* 在appDelegate中首先申请一个backgroundTask
* 在利用一个timer打印app在后台剩余的时长(即还有多久app进入挂起状态）


```
#import "AppDelegate.h"

@interface AppDelegate ()

// about background task
@property (nonatomic,unsafe_unretained) UIBackgroundTaskIdentifier backgroundTaskIdentifer;
@property (nonatomic,strong)NSTimer *homeQuitTimer;
@property (nonatomic,assign) NSTimeInterval enterBackTime;

@end

@implementation AppDelegate


- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    NSLog(@"%s,%i",__func__,__LINE__);
    return YES;
}


- (void)applicationWillResignActive:(UIApplication *)application {
    // Sent when the application is about to move from active to inactive state. This can occur for certain types of temporary interruptions (such as an incoming phone call or SMS message) or when the user quits the application and it begins the transition to the background state.
    // Use this method to pause ongoing tasks, disable timers, and invalidate graphics rendering callbacks. Games should use this method to pause the game.
    NSLog(@"%s,%i",__func__,__LINE__);
}


- (void)applicationDidEnterBackground:(UIApplication *)application {
    // Use this method to release shared resources, save user data, invalidate timers, and store enough application state information to restore your application to its current state in case it is terminated later.
    // If your application supports background execution, this method is called instead of applicationWillTerminate: when the user quits.
    NSLog(@"%s,%i",__func__,__LINE__);
    
    
    
    self.backgroundTaskIdentifer = [[UIApplication sharedApplication] beginBackgroundTaskWithExpirationHandler:nil];
    /* test app in background time */
    if (_homeQuitTimer == nil) {
        _homeQuitTimer = [NSTimer scheduledTimerWithTimeInterval:1.0f
                                                          target:self
                                                        selector:@selector(timerMethod:)
                                                        userInfo:nil
                                                         repeats:YES];
    }
    
}


- (void)applicationWillEnterForeground:(UIApplication *)application {
    // Called as part of the transition from the background to the active state; here you can undo many of the changes made on entering the background.
    NSLog(@"%s,%i",__func__,__LINE__);
}


- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Restart any tasks that were paused (or not yet started) while the application was inactive. If the application was previously in the background, optionally refresh the user interface.
    NSLog(@"%s,%i",__func__,__LINE__);
}


- (void)applicationWillTerminate:(UIApplication *)application {
    // Called when the application is about to terminate. Save data if appropriate. See also applicationDidEnterBackground:.
    NSLog(@"%s,%i",__func__,__LINE__);
}

/** calculator rest of the time  */
- (void)timerMethod:(NSTimer *)paramSender
{
    NSTimeInterval backgroundTimeRemaining = [[UIApplication sharedApplication] backgroundTimeRemaining];
    if (backgroundTimeRemaining == DBL_MAX) {
        NSLog(@"Background Time Remaining = Undetermined..");
    }else{
        NSLog(@"Background Time Remaining = %.02f Seconds",backgroundTimeRemaining);
    }
    
}

@end
```

日志输出：
```
2018-02-05 15:49:55.423283+0800 Long_Task[6671:4838645] -[AppDelegate application:didFinishLaunchingWithOptions:],25
2018-02-05 15:49:55.438470+0800 Long_Task[6671:4838645] refreshPreferences: HangTracerEnabled: 0
2018-02-05 15:49:55.438541+0800 Long_Task[6671:4838645] refreshPreferences: HangTracerDuration: 500
2018-02-05 15:49:55.438566+0800 Long_Task[6671:4838645] refreshPreferences: ActivationLoggingEnabled: 0 ActivationLoggingTaskedOffByDA:0
2018-02-05 15:49:55.438664+0800 Long_Task[6671:4838645] -[AppDelegate applicationDidBecomeActive:],65
2018-02-05 15:49:58.345475+0800 Long_Task[6671:4838645] -[AppDelegate applicationWillResignActive:],33
2018-02-05 15:49:59.409160+0800 Long_Task[6671:4838645] -[AppDelegate applicationDidEnterBackground:],40
2018-02-05 15:50:00.416618+0800 Long_Task[6671:4838645] Background Time Remaining = 179.00 Seconds
2018-02-05 15:50:01.501142+0800 Long_Task[6671:4838645] Background Time Remaining = 177.92 Seconds
2018-02-05 15:50:02.435090+0800 Long_Task[6671:4838645] Background Time Remaining = 176.98 Seconds
2018-02-05 15:50:03.443717+0800 Long_Task[6671:4838645] Background Time Remaining = 175.98 Seconds
2018-02-05 15:50:04.504423+0800 Long_Task[6671:4838645] Background Time Remaining = 174.91 Seconds
2018-02-05 15:50:05.443031+0800 Long_Task[6671:4838645] Background Time Remaining = 173.98 Seconds
2018-02-05 15:50:06.441122+0800 Long_Task[6671:4838645] Background Time Remaining = 172.98 Seconds
```

时长大概是180s 。 当用户再次进入app时，时长会变成`DBL_MAX` 。


## Downloading Content in the Background(在后台下载内容)

当下载文件时，应用应该利用NSURLSession 对象来启动下载，这样系统就可以控制下载而不去管应用处于何种状态，即使应用挂起或终止。当你配置一个NSURLSession 对象为了后台传输，通常情况下系统会在一个单独的线程里管理这些传输并且向你的应用报告传输状态。如果传输正在进行时你的应用被终止了，系统会继续在后台传输并且会在传输完成或者有一个或多个任务需要app关注时启动应用。//这它妈咋翻译：If your app is terminated while transfers are ongoing, the system continues the transfers in the background and launches your app (as appropriate) when the transfers finish or when one or more tasks need your app’s attention.

为了支持后台传输，你必须适当的配置你的NSURLSession对象。为了配置这个session对象，你必须首先创建一个NSURLSessionConfiguration对象并且设置一些属性。然后通过这个*configuration对象来创建NSURLSession对象。

创建configuration对象支持后台下载的步骤如下： 
1. 创建configuration对象，利用NSURLSessionConfiguration 的backgroundSessionConfigurationWithIdentifier: 方法。 
2. 设置configurtion对象的sessionSendsLaunchEvents 属性为YES 
3. 如果你的应用在前台的时候就开始传输，那么建议设置configuration对象的 discretionary属性为YES 
4. 配置其他相关的configuration对象属性 
5. 利用configuration对象，创建NSURLSession对象

一旦你配置完这些，你的NSURLSession对象就会把上传和下载任务在相应的时间交给系统处理。如果任务完成的时候，你的应用依然在前台运行，session对象会通知它的代理。如果任务没有完成的时候系统终止了你的应用，系统会在后台继续自动管理这些任务，如果用户终止了你的应用，系统会取消所有正在等待的任务。

当所有和后台session相关联的任务完成的时候，系统重新启动终止的应用（assuming that the sessionSendsLaunchEvents property was set to YES and that the user did not force quit the app）并会调用app的代理方法* application:handleEventsForBackgroundURLSession:completionHandler:*（The system may also relaunch the app to handle authentication challenges or other task-related events that require your app’s attention.），在你实现这个代理中，利用提供的标识创建一个和以前配置一样的新的NSURLSessionConfiguration和NSURLSession对象， 
系统会重新连接新的session对象到以前的任务，并向session对象的代理报告其状态。

## Implementing Long-Running Tasks(实现长运行任务)

对于那些需要更长的额外时间运行的应用，你必须申请特殊的权限来让系统允许应用一直在后台运行而不被挂起。在ios中，只有以下几种类型的应用允许在后台运行： 
- 在后台为用户播放音频的应用，例如音乐播放器。 
- 录制音频的应用，即使在后台状态下。 
- 随时能让用户了解自己的位置的应用，比如导航应用。 
- 支持互联网协议的语音应用。support Voice over Internet Protocol (VoIP) 
- 需要下载并定期处理新内容的应用。 
- 从外部附件收到定期更新的应用。 
实现这些服务的应用必须声明它所支持的服务和它使用的系统框架来实现这些服务的相关方面。声明这些服务让系统知道你使用了哪些服务，但在某些情况下，它实际上是防止被暂停的应用的系统框架。

### Declaring Your App’s Supported Background Tasks(声明所支持的后台任务)

对于某些类型的后台执行的支持，必须首先在应用中事先通过使用它们的声明。在Xcode5即以后的版本，声明后台运行所支持的类型在project setting。Enabling the Background Modes option adds the UIBackgroundModes key to your app’s Info.plist file. Selecting one or more checkboxes adds the corresponding background mode values to that key. Table 3-1 lists the background modes you can specify and the values that Xcode assigns to the UIBackgroundModes key in your app’s Info.plist file.

Table 3-1 Background modes for apps 

![image](http://img.blog.csdn.net/20160331135037051)

通过以上声明，应用就可以在进入后台的时候依然运行了。

### Tracking the User’s Location(跟踪用户位置)

有多种方法能在后台情况下跟踪用户位置，这些方法中的大多数都不是真的让你的应用一直在后台运行： 
- The significant-change location service (Recommended) 
- Foreground-only location services 
- Background location services 

第一种方法是高度推荐的方法，它不需要高精度的位置数据。对于这项服务，只有当用户的位置显著变化时位置才更新；这种策略是一些社会化app为用户提位置相关的信息。如果在应用挂起的状态下有一个位置更新了，那么系统会在后台唤醒这个app并处理更新。如果app开始这个服务，但后来app被终止了，系统会在一个新的位置更新的时候重新启动app。这个服务是在ios4.0即其以后才有的。只有当设备上有蜂窝无线的时候，此项服务才有效。

第二种和第三种方法都是使用了表中的Core Location 服务来获取位置数据。它们唯一的区别是foreground-only location service 将会停止提供数据如果app曾经挂起过。这项服务适用于在前天运行时只需要位置数据的应用。

第三种方式 You enable location support from the Background modes section of the Capabilities tab in your Xcode project. (You can also enable this support by including the UIBackgroundModes key with the location value in your app’s Info.plist file.) Enabling this mode does not prevent the system from suspending the app, but it does tell the system that it should wake up the app whenever there is new location data to deliver. Thus, this key effectively lets the app run in the background to process location updates whenever they occur. 使用这种模式不会阻止系统挂起应用程序，但是它会告诉系统应该唤醒应用当有新的位置信息来的时候。

Important: You are encouraged to use the standard services sparingly or use the significant location change service instead. Location services require the active use of an iOS device’s onboard radio hardware. Running this hardware continuously can consume a significant amount of power. If your app does not need to provide precise and continuous location information to the user, it is best to minimize the use of location services.

### Playing and Recording Background Audio

如果一款应用播放或录制音频，那就可以在后台给它注册这么一个任务。你可以通过xcode的属性中选择让该应用在后台情况下支持音频服务。你也可以通过在Info.plist中设置UIBackgroundModes的值为audio.如果应用设置为支持此项服务，那么必须给它设置音频内容让它播放，而不是让它什么都不做。

背景是音频的应用的典型例子有： 
- 音乐播放应用 
- 音频录制应用 
- 支持AirPlay的多音频或视频播放应用 
- VoIP应用 
当为app注册了背景是音频后，系统的多媒体框架就是自动的阻止应用进入后台挂起。只要应用一直播放音频或视频，应用就会一直在后台运行下去。但是，如果录制或播放停止了，应用就会立马挂起。

你可以利用任何系统的音频框架来播放后台的音频内容，并且当你选择一种框架后，在使用的过程中不能改变。（如果是视频播放，你可以使用Media Play或者是AV Foundation框架播放）。因为你的app在播放多媒体文件的时候是不会挂起的，你的应用在后台运行的时候回调是正常的。在回调中，你应该做的就是为播放提供数据。例如，一个音乐播放应用需要从服务器上下载音乐数据并且需要播放音乐样本。应用不应该执行无关播放的任何多余任务。

因为不知一个应用支持音频，有系统决定在某一时间有哪个应用被允许播放或录制音频。在前台运行的应用一般优先级比较高。这种情况是有可能的，对于多个在后台运行的应用被允许播放音频，这种决定权依赖于每个应用的audio session的具体配置。你应该经常配置你的audio session，并且小心的利用系统框架去处理其它音频应用的打断通知。具体如何配置后台执行的audio session对象，应该看Audio Session Programming Guide.

#### 示例

在示例代码中，播放的仅仅是本地音乐。

* 首先在 Capabilities 中设置后台模式为 `Audio, AirPlay, and Picture in Picture`
* 在主控制器中用`AVPalyer`循环播放一段视频
* 在appDelegate中，在'applicationWillResignActive:'中设置AVAudioSession的类型。如果不设置这一步，则无法实现后台播放。

示例代码：

ViewController.m

```
#import "ViewController.h"
#import <AVFoundation/AVFoundation.h>

@interface ViewController ()

@property (nonatomic,strong) AVPlayer *avPlayer;

@end

@implementation ViewController

- (AVPlayer *)avPlayer
{
    if (!_avPlayer) {
        
        AVPlayerItem *playerItem = [self getPlayItem:1];
        _avPlayer = [AVPlayer playerWithPlayerItem:playerItem];
        
         [self addNotification];
    }
    return _avPlayer;
}

- (void)addNotification
{
    //给AVPlayerItem添加播放完成通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(playbackFinished:) name:AVPlayerItemDidPlayToEndTimeNotification object:self.avPlayer.currentItem];
}

- (AVPlayerItem *)getPlayItem:(int)videoIndex
{
    NSURL *url = [[NSURL alloc] initFileURLWithPath:[[NSBundle mainBundle] pathForResource:@"IMG_2362" ofType:@"mp4"]];
    AVPlayerItem *playerItem = [AVPlayerItem playerItemWithURL:url];
    return playerItem;
}

- (void)playbackFinished:(NSNotification *)notification
{
    [self.avPlayer seekToTime:CMTimeMake(0, 1)];  // 视频播放完成以后，从新设置到起点
    NSLog(@"视频播放完成");
    [self.avPlayer play];
   
}


- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    [self.avPlayer seekToTime:CMTimeMake(0, 1.0)];
    [self.avPlayer play];
}

```

AppDelegate.m

```
- (void)applicationWillResignActive:(UIApplication *)application {
    // Sent when the application is about to move from active to inactive state. This can occur for certain types of temporary interruptions (such as an incoming phone call or SMS message) or when the user quits the application and it begins the transition to the background state.
    // Use this method to pause ongoing tasks, disable timers, and invalidate graphics rendering callbacks. Games should use this method to pause the game.
    
    //开启后台处理多媒体事件
    [[UIApplication sharedApplication] beginReceivingRemoteControlEvents];
    AVAudioSession *session = [AVAudioSession sharedInstance];
    [session setActive:YES error:nil];
    //后台播放
    [session setCategory:AVAudioSessionCategoryPlayback error:nil];
}

```


## Getting the User’s Attention While in the Background（在应用处于后台时，得到用户的注意）

Notifications是ios中的一种很好的机制，无论应用是在挂起、后台运行、前台运行的时候都可以发送通知。应用可以利用本地通知来显示提示框、播放音频、设置应用提示图标或者是这三种功能的组合。举例来说，一个闹钟应用应该使用本地通知来播放闹钟声音和显示提示框提示闹钟。当一个通知发送给用户时，用户必须决定是否处理信息提示，并把app回到前台。(If the app is already running in the foreground, local notifications are delivered quietly to the app and not to the user.)

要安排本地通知的发送，创建一个UILocationNotification的实例，设置通知对象的属性，并使用UIApplication的方法来安排它。本地通知对象包含了一些信息（sound，alert、badge）和交付时间。UIApplication的方法可以立即发送通知或者是在计划的时间内选择。

eg: 
Listing 3-2 shows an example that schedules a single alarm using a date and time that is set by the user. This example configures only one alarm at a time and cancels the previous alarm before scheduling a new one. (Your own apps can have no more than 128 local notifications active at any given time, any of which can be configured to repeat at a specified interval.) The alarm itself consists of an alert box and a sound file that is played if the app is not running or is in the background when the alarm fires. If the app is active and therefore running in the foreground, the app delegate’s application:didReceiveLocalNotification: method is called instead.

Listing 3-2 Scheduling an alarm notification

```
- (void)scheduleAlarmForDate:(NSDate*)theDate
{
    UIApplication* app = [UIApplication sharedApplication];
    NSArray*    oldNotifications = [app scheduledLocalNotifications];

    // Clear out the old notification before scheduling a new one.
    if ([oldNotifications count] > 0)
        [app cancelAllLocalNotifications];

    // Create a new notification.
    UILocalNotification* alarm = [[UILocalNotification alloc] init];
    if (alarm)
    {
        alarm.fireDate = theDate;
        alarm.timeZone = [NSTimeZone defaultTimeZone];
        alarm.repeatInterval = 0;
        alarm.soundName = @"alarmsound.caf";
        alarm.alertBody = @"Time to wake up!";

        [app scheduleLocalNotification:alarm];
    }
}
```

Sound files used with local notifications have the same requirements as those used for push notifications. Custom sound files must be located inside your app’s main bundle and support one of the following formats: Linear PCM, MA4, µ-Law, or a-Law. You can also specify the UILocalNotificationDefaultSoundName constant to play the default alert sound for the device. When the notification is sent and the sound is played, the system also triggers a vibration on devices that support it.

You can cancel scheduled notifications or get a list of notifications using the methods of the UIApplication class. For more information about these methods, see UIApplication Class Reference. For additional information about configuring local notifications, see Local and Remote Notification Programming Guide.

## Understanding When Your App Gets Launched into the Background

Apps that support background execution may be relaunched by the system to handle incoming events. If an app is terminated for any reason other than the user force quitting it, the system launches the app when one of the following events happens:

For location apps: 
The system receives a location update that meets the app’s configured criteria for delivery. 
The device entered or exited a registered region. (Regions can be geographic regions or iBeacon regions.) 
For audio apps, the audio framework needs the app to process some data. (Audio apps include those that play audio or use the microphone.) 
For Bluetooth apps: 
An app acting in the central role receives data from a connected peripheral. 
An app acting in the peripheral role receives commands from a connected central. 
For background download apps: 
A push notification arrives for an app and the payload of the notification contains the content-available key with a value of 1. 
The system wakes the app at opportunistic moments to begin downloading new content. 
For apps downloading content in the background using the NSURLSession class, all tasks associated with that session object either completed successfully or received an error. 
A download initiated by a Newsstand app finishes. 
In most cases, the system does not relaunch apps after they are force quit by the user. One exception is location apps, which in iOS 8 and later are relaunched after being force quit by the user. In other cases, though, the user must launch the app explicitly or reboot the device before the app can be launched automatically into the background by the system. When password protection is enabled on the device, the system does not launch an app in the background before the user first unlocks the device.


## Opting Out of Background Execution（选择退出后台执行）

If you do not want your app to run in the background at all, you can explicitly opt out of background by adding the UIApplicationExitsOnSuspend key (with the value YES) to your app’s Info.plist file. When an app opts out, it cycles between the not-running, inactive, and active states and never enters the background or suspended states. When the user presses the Home button to quit the app, the applicationWillTerminate: method of the app delegate is called and the app has approximately 5 seconds to clean up and exit before it is terminated and moved back to the not-running state. 
如果你不想让你的应用在后执行，你可以在Info.plist中明确第增加UIApplicationExitsOnSuspend的值为YES.当一个应用退出时，他的状态就会在 不运行-不活跃-活跃状态下更换，应用不会进入后台或者是挂起状态。当用户点击Home键退出app，AppDelegate的* applicationWillTerminate:*会被调用，应用在进入不运行状态之前大约有5秒钟的清理时间。

Opting out of background execution is strongly discouraged but may be the preferred option under certain conditions. Specifically, if coding for background execution adds significant complexity to your app, terminating the app might be a simpler solution. Also, if your app consumes a large amount of memory and cannot easily release any of it, the system might kill your app quickly anyway to make room for other apps. Thus, opting to terminate, instead of switching to the background, might yield the same results and save you development time and effort. 
选择退出后台执行是不被推荐的，但可能会在一定条件下优先选择。特别是，如果后台编码执行增加了显著的复杂性，终止应用可能是一个简单的方案。还有就是，如果你的app占用了大量的内存资源而不能轻易释放时，系统有可能终止你的程序，以让更多的资源给其它应用。这样，选择终止应用，替代切换到后台，通过这种方式不仅能产生相同的结果而且还能节省开发经历和时间。

## 代码

[github: ios_app_runInBackground](https://github.com/MaxwellQi/ios_app_runInBackground)