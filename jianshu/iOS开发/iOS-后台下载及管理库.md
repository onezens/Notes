说起下载第一个想起的就是ASI。一年前接手的新项目是核心功能是视频相关业务，在修改和解决视频下载相关的问题的时候让我体会到了ASI的下载的强大。后来新需求需要视频后台下载，使用NSURLSession的时候，更加深刻的体会到了ASI的强大好用。  

后来替换下载的时候的原因：
1. ASI开启后台下载功能，在iOS10的设备上，只能下载三分钟，然后就处于休眠状态
2. AFN下载也是三分钟
3. 测试后台下载的时候，不要用模拟器，使用用真机。模拟器APP处于后台时不会休眠。



### NSURLSession的特点简介
通过NSURLSession创建的后台下载任务，保证了APP在后台或者退出的状态下，依然能进行下载任务，下载完成后通过唤醒APP，来将下载完成的数据保存到特定的位置。

1. 在APP处于后台、锁屏状态下依然能后下载
2. 最强大的是：APP在手动退出以及闪退后的状态下依然能够进行下载任务

### NSURLSession

**创建下载session**

```
NSString *bundleId = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleIdentifier"];
        NSString *identifier = [NSString stringWithFormat:@"%@.BackgroundSession", bundleId];
        NSURLSessionConfiguration* sessionConfig = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:identifier];
        session = [NSURLSession sessionWithConfiguration:sessionConfig
                                                delegate:self
                                           delegateQueue:[NSOperationQueue mainQueue]];

```

1. 在创建下载session的时候，需要一个下载标识，该标识需要在整个个系统内保证唯一，所以使用APP的bundle id。
2. sessionConfig.allowsCellularAccess  控制是否可以通过蜂窝网络下载

当APP手动退出或者闪退后，重新启动时获取正在下载的tasks

```
NSMutableDictionary *dictM = [self.downloadSession valueForKey:@"tasks"];
[dictM enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
           
}];

```

**AppDelegate后台下载回调**

当APP处于后台下载状态时，需要处理下载完成后的数据的回调，这里就涉及了一个AppDelegate中的一个特别重要的回调

```
-(void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler{
    NSLog(@"%s", __func__);
}

```
该代理方法使用场景分析(详细分析过程见下面)：

1. 不实现该代理方法，手动进入App调用相关代理方法，部分情况下异常
2. 实现该方法，不执行completionHandler，某一下载任务完成后唤醒App，继续其它下载任务，异常
3. 实现该方法，执行completionHandler，某一下载任务完成后唤醒App，继续其它下载任务，一切正常


**相关操作**

1. 创建下载任务:其实通过session创建的任务是NSURLSessionDownloadTask的子类__NSCFBackgroundDownloadTask，是苹果的私有API。

    ```
    NSURL *downloadURL = [NSURL URLWithString:downloadURLString];
       NSURLRequest *request = [NSURLRequest requestWithURL:downloadURL];
       NSURLSessionDownloadTask *downloadTask = [self.downloadSession downloadTaskWithRequest:request];
       [downloadTask resume];
    ```


2. 暂停下载:downloadTask有多种办法去暂停，但是我们选择有resume的下载方法，可以更加方便我们管理和多次暂停继续。

    ```
    [downloadTask cancelByProducingResumeData:^(NSData * resumeData) {
    }];
   ```

3. 继续下载:通过resumeData继续下载。

    ```
    NSURLSessionDownloadTask *downloadTask = [self.downloadSession downloadTaskWithResumeData:data];
    [downloadTask resume];
    
    ```

4. 取消或者删除下载

    ```
    [downloadTask cancel];
    ```


**相关代理方法说明**

1. 下载任务开始后，下载文件的进度回调方法。`bytesWritten` 某一断点续传过程中已经下载的数据大小， `totalBytesWritten` 已经下载的文件的大小；`totalBytesExpectedToWrite`当前需要下载的文件的大小

    ```
    - (void)URLSession:(NSURLSession *)session
          downloadTask:(NSURLSessionDownloadTask *)downloadTask
          didWriteData:(int64_t)bytesWritten
     totalBytesWritten:(int64_t)totalBytesWritten
    totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite{
    
    }
    ```

2. 下载任务继续开始下载时的回调方法
    
    ```
     - (void)URLSession:(NSURLSession *)session
          downloadTask:(NSURLSessionDownloadTask *)downloadTask
     didResumeAtOffset:(int64_t)fileOffset
    expectedTotalBytes:(int64_t)expectedTotalBytes {

     }
    ```

3. 当资源发生重定向时回调的方法。NSURLSession内部自己处理定向回调
 
    ```
    - (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
    willPerformHTTPRedirection:(NSHTTPURLResponse *)response
            newRequest:(NSURLRequest *)request
     completionHandler:(void (^)(NSURLRequest * _Nullable))completionHandler{
    
    }
    ```

    >讲个小故事：去年元旦左右，我们是使用ASI下载视频的，甘肃的一个用户反馈，视频死活不能下载。但是我们在公司网络，国外VPN，4G，家里的网络测试下载没有问题，并且进行n次的测试，没有复现该问题，后来只能只作罢。然后过年回家，腊月30到家的，和家人吃年夜饭过后，看春晚，实在没意思，突然想起这个问题，然后就测试代码，我去，还真的是个必现的bug，死活下载不了。卡断点，调试一会后，发现下载资源URL被302重定向了，然后我们项目中的以前集成ASI，并不支持重定向，最后在失败的回调用，判断了下，如果是重定向，拿到新的URL重新去下载，想想也是坑啊。  下载的资源经过cdn加速后，猜测cdn在不同的region做了不同的处理。然后北京没事，其它的地域有可能重定向了。当时改完代后有点小开心， 因为怕同事说我猿气太重，所以在公司提交的代码...

4. 当下载任务完成后的代理回调方法，回调的参数`location`是下载完成后的文件，在沙盒当中存在的路径

    ```
    - (void)URLSession:(NSURLSession *)session
          downloadTask:(NSURLSessionDownloadTask *)downloadTask
    didFinishDownloadingToURL:(NSURL *)location{
      
    }
    
    ```

5. 假如在后台下载完成的回调，会触发该回调方法。

    ```
    - (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session{
    }
    ```

6. 下载失败后的回调。暂停，停止，失败都会触发。区别是：正常状态下的暂停会回调resumeData，如果resumeData不为空的话我们需要保存该数据，方便下次继续。

    ```
    - (void)URLSession:(NSURLSession *)session
                  task:(NSURLSessionTask *)task
    didCompleteWithError:(NSError *)error{
    } 
    ```
    
### 后台下载相关回调
AppDelegate 回调方法3种情况分析：


| | 实现代理| 执行completionHandler | 现象 |
|------|--------|--------|--------|
| 1 | 否   |      | 部分情况下异常|
| 2 | 是   |   否 | 部分情况下异常|
| 3 | 是   |  是   |  一切正常 |


**1. 不实现代理方法**
当不实现AppDelegate代理方法的时候，简单的下载是没有任何问题。NSURLSession下载完成后相关代理方法执行顺序如图：



![代理调用顺序](http://upload-images.jianshu.io/upload_images/1216462-2b1c8817bba99f20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


只有手动将后台的App进入前台后会调用成功的回调，处理相关的数据。假如有3个下载任务则会回调3次`URLSession:downloadTask:didFinishDownloadingToURL:`方法。

缺点和问题：

* 如果下载完成后，不进入前台或者手动杀死进程，则丢失下载数据
* 等待下载的任务无法继续下载
* 下载完成后，在后台无法使用本地通知


**2. 实现代理方法，不执行completionHandler**

AppDelegate 方法`-(void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler` completionHandler 作用：

首先看苹果SDK的解释：
>// Applications using an NSURLSession with a background configuration may be launched or resumed in the background in order to handle the
// completion of tasks in that session, or to handle authentication. This method will be called with the identifier of the session needing
// attention. Once a session has been created from a configuration object with that identifier, the session's delegate will begin receiving
// callbacks. If such a session has already been created (if the app is being resumed, for instance), then the delegate will start receiving
// callbacks without any action by the application. You should call the completionHandler as soon as you're finished handling the callbacks.

最后两句是说明completionHandler的，大概意思是：回调callbacks，只要session创建将会开始接收到。回调callbacks没有对你的应用程序进行任何的处理，一旦完成处理callbacks，你应该调用completionHandler。

英语不好感觉说的云里雾里的。callbacks应该是暂停继续任务时受到的代理方法。说的是我们完成下载任务后应该调用completionHandler


completionHandler作用测试代理：

```
#pragma mark - test code
- (void)applicationWillResignActive:(UIApplication *)application{
    
    [self testTimer];
    NSLog(@"%s",__func__);
}

- (void)applicationDidBecomeActive:(UIApplication *)application {
    [self.timer invalidate];
    self.timer = nil;
    self.duration = 0;
    NSLog(@"%s", __func__);
}


- (void)testTimer {
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(timerRun) userInfo:nil repeats:true];
    [self.timer fire];
}

- (void)timerRun {
    _duration += 1;
    NSLog(@"%zd", _duration);
}

```

completionHandler具体作用：
这个回调的作用有点牛皮。通过上面代理可以测试出completionHandler，在第一个后台下载任务完成时回调，这时后台App已经被唤醒，定时器开始输出计时秒数。然后其它的下载任务完成时不会再次回调该方法。所有下载任务完成时，没有处理completionHandler计时器继续运行。 调用执行时completionHandler计时器停止运行，App继续处于休眠状态。

虽然不知道completionHandler做了哪些处理，但是通过测试现象得出大概的作用。他用来控制后台的App被唤醒后继续处于休眠状态，节约系统资源。

所以不执行completionHandler App如果不重新启动，处于后台时会一直在运行状态。下载任务正常。

**3. 实现代理方法，执行completionHandler**
上面我们分析了completionHandler大概作用。所以所有后台任务下载完成后调用completionHandler，是App处于正常的状态。相关代理调用顺序：


![代理调用顺序](http://upload-images.jianshu.io/upload_images/1216462-a6f28eddfc10c81b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

completionHandler调用时机：
所有的下载任务下载完成后调用。感兴趣的可以看[YCDowloadSession]([https://github.com/onezens/YCDownloadSession](https://github.com/onezens/YCDownloadSession))下载库对completionHandler的处理逻辑。


### YCDownloadSession
YCDownloadSession是我写的一个视频后台下载的库。里面拥有我对视频下载的详细的处理过程和管理的库。

该视频下载库主要有四个核心类：YCDownloadSession，YCDownloadTask，YCDownloadItem，YCDownloadManager  

1. YCDownloadSession：对NSURLSession的进一步分装，是一个单例，所有的下载任务都是由其生成和管理。是最主要的核心类。实现了下载的代理方法，通过一个可下载的url，生成一个YCDownloadTask，并且将该task的所有数据进行实时存储。
2. YCDownloadTask 将YCDownloadSession里的代理方法进一步封装和扩展，保存session生成和所需要的一些下载信息和数据。
3. YCDownloadItem 存放需要下载的视频的信息
4. YCDownloadManager 管理下载视频操作，生成一个YCDownloadItem，并且实时保存相关信息(下载状态，文件大小，已下载文件大小，以及其它的需要和UI交互的数据)，然后调用YCDownloadSession去下载该视频。

**图解**


![YCDownloadSession结构图解](http://upload-images.jianshu.io/upload_images/1216462-a5278facdcb6aca9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YCDownloadSession和YCDownloadTask是两个核心类。与YCDownloadManager和YCDownloadItem相互独立。大家和可以通过YCDownloadSession和YCDownloadTask自定义需要的下载管理类的信息类。


 **使用效果图**

1. 单文件下载测试


![单文件下载](http://upload-images.jianshu.io/upload_images/1216462-cff1634a8c2a9e10.gif?imageMogr2/auto-orient/strip)


2. 多视频下载测试

![多视频下载](http://upload-images.jianshu.io/upload_images/1216462-a9c4bda4b1a8c65b.gif?imageMogr2/auto-orient/strip)

3. 通知

![通知](http://upload-images.jianshu.io/upload_images/1216462-2a8f143a8049ae55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **用法**
1. AppDelegate设置后台下载成功回调方法

	```
	-(void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler{
	    [[YCDownloadSession downloadSession] addCompletionHandler:completionHandler];
	}
	
	```
	
	如果想要多个任务在后台同时进行，必须要进行设置上述的代理方法。YCDownloadSession内部会处理该回调方法(completionHandler的作用将会在blog里详细说明)，内部处理逻辑：

    ```
     //等task delegate方法执行完成后去判断该逻辑
     //URLSessionDidFinishEventsForBackgroundURLSession 方法在后台执行一次，所以在此判断执行completedHandler
     if (status == YCDownloadStatusFinished) {
         
         if ([self allTaskFinised]) {
             [[NSNotificationCenter defaultCenter] postNotificationName:kDownloadAllTaskFinishedNoti object:nil];
             //所有的任务执行结束之后调用completedHanlder
             if (self.completedHandler) {
                 NSLog(@"completedHandler");
                 self.completedHandler();
                 self.completedHandler = nil;
             }
         }
      
     }
    ```

2. 直接使用YCDownloadSession下载文件

	```
	self.downloadURL = @"http://dldir1.qq.com/qqfile/QQforMac/QQ_V6.0.1.dmg";
	
    - (void)start {
        [[YCDownloadSession downloadSession] startDownloadWithUrl:self.downloadURL delegate:self saveName:nil];
    }
    - (void)resume {
        [[YCDownloadSession downloadSession] resumeDownloadWithUrl:self.downloadURL delegate:self saveName:nil];
    }
    
    - (void)pause {
        [[YCDownloadSession downloadSession] pauseDownloadWithUrl:self.downloadURL];
    }
    
    - (void)stop {
        [[YCDownloadSession downloadSession] stopDownloadWithUrl:self.downloadURL];
    }
    	
    //代理
    - (void)downloadProgress:(YCDownloadTask *)task downloadedSize:(NSUInteger)downloadedSize fileSize:(NSUInteger)fileSize {
        self.progressLbl.text = [NSString stringWithFormat:@"%f",(float)downloadedSize / fileSize * 100];
    }
    
    	
    - (void)downloadStatusChanged:(YCDownloadStatus)status downloadTask:(YCDownloadTask *)task {
        if (status == YCDownloadStatusFinished) {
            self.progressLbl.text = @"download success!";
        }else if (status == YCDownloadStatusFailed){
            self.progressLbl.text = @"download failed!";
        }
    }

	```
	
3. 使用自定义的管理类(YCDownloadManager 视频类型文件专用下载管理类)下载。假如视频下载完成，自定义保存名称，那么使用fileId来标识。如果fileId为空使用下载URL的MD5的值来保存

	```
    /**
     开始/创建一个后台下载任务。downloadURLString作为整个下载任务的唯一标识。
     下载成功后用downloadURLString的MD5的值来保存
     文件后缀名取downloadURLString的后缀名，[downloadURLString pathExtension]
    
     */
    + (void)startDownloadWithUrl:(NSString *)downloadURLString fileName:(NSString *)fileName imageUrl:(NSString *)imagUrl;
    
    /**
     开始/创建一个后台下载任务。fileId作为整个下载任务的唯一标识。
     下载成功后用fileId来保存, 要确保fileId唯一
     文件后缀名取downloadURLString的后缀名，[downloadURLString pathExtension]
     
     */
    + (void)startDownloadWithUrl:(NSString *)downloadURLString fileName:(NSString *)fileName imageUrl:(NSString *)imagUrl fileId:(NSString *)fileId;

	
	```

4. 蜂窝煤是否允许下载的方法(YCDownloadSession, YCDownloadManager)

	```
	YCDownloadSession: 
	/**
	 是否允许蜂窝煤网络下载，以及网络状态变为蜂窝煤是否允许下载，必须把所有的downloadTask全部暂停，然后重新创建。否则，原先创建的
	 下载task依旧在网络切换为蜂窝煤网络时会继续下载
	 
	 @param isAllow 是否允许蜂窝煤网络下载
	 */
	- (void)allowsCellularAccess:(BOOL)isAllow;
	
	YCDownloadManager:
	/**
	 获取当前是否允许蜂窝煤访问状态
	 */
	- (BOOL)isAllowsCellularAccess;
	```

5. 设置最大同时进行下载的任务数

	```
	YCDownloadSession: 
	/**
	 设置下载任务的个数，最多支持3个下载任务同时进行。
	 NSURLSession最多支持5个任务同时进行
	 但是5个任务，在某些情况下，部分任务会出现等待的状态，所有设置最多支持3个
	 */
	@property (nonatomic, assign) NSInteger maxTaskCount;
	
	
	
	YCDownloadManager:
	/**
	 设置下载任务的个数，最多支持3个下载任务同时进行。
	 */
	+ (void)setMaxTaskCount:(NSInteger)count;
	```
	
6. 下载完成的通知
	* 本地通知(YCDownloadManager实现)：
	
		```
		/**
		 本地通知的开关，默认是false,可以根据通知名称自定义通知类型
		 */
		+ (void)localPushOn:(BOOL)isOn;
		```
	* 当前session中所有的任务下载完成的通知。 不包括失败、暂停的任务: `kDownloadAllTaskFinishedNoti`
	* 某一的任务下载完成的通知object为YCDownloadItem对象：`kDownloadTaskFinishedNoti`

7. 某一任务下载的状态发生变化的通知: `kDownloadStatusChangedNoti` 主要用于状态改变后，及时保存下载数据信息。

**GitHub连接**
[https://github.com/onezens/YCDownloadSession](https://github.com/onezens/YCDownloadSession)

**欢迎各位关注该库，如果你有任何问题请issues我，将会随时更新新功能和解决存在的问题。**


### 遇到的一些问题
这里总结下载开发[YCDownloadSession](https://github.com/onezens/YCDownloadSession)下载库中碰到的一些问题

![苹果下载相关SDK关系图](http://upload-images.jianshu.io/upload_images/1216462-2eeff820360be8a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. 下载资源重定向的问题
YCDownloadSession 内部标识一下下载task的时候，使用的下载资源的URL来标识。如果该资源被301/302重定向到一个另一个URL后，会存在两个URL。标识用的URL在代理回调的NSURLSessionTask或者NSURLSessionDownloadTask的currentRequest中取的url，这样就出现了一个问题，重定向后通过URL拿不到下载的task；originalRequest属性可以拿到重定向前的URL使用该属性解决这个问题。苹果下载相关SDK关系图可看它们之间关系。

2. 断点续传
NSURLSessionDownloadTask的断点续传是由其内部自己控制实现。在暂停某一下载任务的时候有两个方法：
    * `cancel `内部自己控制断点续传数据，拿到对应task可以继续下载。如果拿不到，不可继续。
    * `cancelByProducingResumeData:^(NSData * resumeData) {}`通过session继续下载`[downloadSession downloadTaskWithResumeData:data]`，需要自己保存处理resumeData，可以满足很多情况下的续传。

3. 部分下载资源不可断点续传
[YCDownloadSession](https://github.com/onezens/YCDownloadSession) Demo中的测试用的下载资源来自百度视频，可以正常下载。网易视频的资源和部分响应头不完整的资源在暂停下载之后拿不到resumeData而回调失败的情况。
    * 百度视频资源：`https://vd1.bdstatic.com/mda-hiqmm8s10vww26sx/mda-hiqmm8s10vww26sx.mp4\?playlist\=%5B%22hd%22%5D\&auth_key\=1506158514-0-0-6cde713ec6e6a15bd856fbb4f2564658\&bcevod_channel\=searchbox_feed`
    响应头：
    通过Mac Terminal自带的curl命令获取响应头 `curl -I https://vd1.bdstatic.com/mda-hiqmm8s10vww26sx/mda-hiqmm8s10vww26sx.mp4\?playlist\=%5B%22hd%22%5D\&auth_key\=1506158514-0-0-6cde713ec6e6a15bd856fbb4f2564658\&bcevod_channel\=searchbox_feed`
    
    ```
    HTTP/1.1 200 OK
    Server: bfe/1.0.8.13-sslpool-patch
    Date: Tue, 10 Oct 2017 07:43:02 GMT
    Content-Type: video/mp4
    Content-Length: 19727666
    Connection: keep-alive
    ETag: "125b8b749037921ccd03120fb0f90189"
    Last-Modified: Fri, 15 Sep 2017 09:42:18 GMT
    Accept-Ranges: bytes
    Content-MD5: EluLdJA3khzNAxIPsPkBiQ==
    x-bce-debug-id: MTAuMTgxLjk5LjE0OlNhdCwgMTYgU2VwIDIwMTcgMTE6MzA6MzYgQ1NUOjE4MzY0NTgzMjM=
    x-bce-request-id: b3c7857b-b71a-4cd4-ab98-8676319dc8cb
    x-bce-storage-class: STANDARD
    Ohc-Response-Time: 1 0 0 0 0 94
    Access-Control-Allow-Origin: *
    Cache-Control: max-age=2592000
    ```

   * 网易视频资源：`http://flv2.bn.netease.com/videolib3/1706/07/gDNOH8458/HD/gDNOH8458-mobile.mp4 `
    响应头：
    
    ```
    curl -I http://flv2.bn.netease.com/videolib3/1706/07/gDNOH8458/HD/gDNOH8458-mobile.mp4
    
    HTTP/1.1 200 OK
    Expires: Thu, 09 Nov 2017 07:00:42 GMT
    Date: Tue, 10 Oct 2017 07:00:42 GMT
    Server: nginx
    Content-Length: 43260722
    Cache-Control: max-age=2592000
    Content-Type: video/mp4
    Via: 1.1 zhshx117:1 (Cdn Cache Server V2.0)[11 200 2], 1.1 xxz195:3 (Cdn Cache Server V2.0)[94 200 2], 1.1 PStjdgdx3jx152:4 (Cdn Cache Server V2.0)[154 200 2]
    Connection: keep-alive
    cache: state
    cdn-user-ip: 101.96.129.122
    cdn-ip: 42.81.28.152
    cdn-source: chinanetcenter
    ```
    
    * 响应头异常视频资源：`https://www.zmzfile.com:9043/rt/route\?fileid\=152260954bdfa322725ba58df2ab1e2c2e3a6050 `
    响应头：
    
    ```
    curl -I https://www.zmzfile.com:9043/rt/route\?fileid\=152260954bdfa322725ba58df2ab1e2c2e3a6050
    
    HTTP/1.1 302 Found
    Server: nginx/1.12.1
    Date: Tue, 10 Oct 2017 07:53:23 GMT
    Content-Type: text/plain; charset=utf-8
    Connection: keep-alive
    Location: http://175.6.228.3:9021?id=152260954bdfa322725ba58df2ab1e2c2e3a6050
    Ts: 1716489
    
    继续取302资源的响应头
    curl -I http://175.6.228.3:9021\?id\=152260954bdfa322725ba58df2ab1e2c2e3a6050
    Ts: 1716489
    curl: (52) Empty reply from server
    
    Xcode 输出NSURLSessionDownloadTask的response信息：
    <NSHTTPURLResponse: 0x13cee64d0> { URL: [http://60.211.203.204:9031?id=152260954bdfa322725ba58df2ab1e2c2e3a6050](http://60.211.203.204:9031/?id=152260954bdfa322725ba58df2ab1e2c2e3a6050) } {
     status code: 200,
     headers {
      Connection = Close;
      "Content-Length" = 247272762;
      Server = p4pcacher;
    } }
    ```
    
   目前这个问题还正在探索解决的办法中。目前来看断点续传是系统内部实现的，苹果SDK没有提供一些相关的操作，如果出现类似的问题，那么只能和后台沟通，解决响应头相关的信息来解决这个问题。
