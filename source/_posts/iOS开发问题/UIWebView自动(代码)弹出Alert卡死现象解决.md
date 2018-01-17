---
title: UIWebView自动(代码)弹出Alert卡死现象解决
date: 2017.09.15 22:33
tags: iOS开发问题
---
### 现象描述
UIWebView代码调用弹出Alert的时候，界面上没有弹出任何Alert。并且在APP内，点击屏幕没有反应，卡死的现象。  


### 产生卡死现象的代码
1. OC调用JS ,在HTML页面加载完成后`- (void)webViewDidFinishLoad:(UIWebView *)webView`调用
	 
	```
	JavaScript:

	function showAlert() {
		alert('秒杀5折起，超性价比抢购');
	}


	OC:

	[self.webView stringByEvaluatingJavaScriptFromString:@"showAlert()"];

	```

2. OC直接执行Alert  

	```
	[self.webView stringByEvaluatingJavaScriptFromString:@"alert('秒杀5折起，超性价比抢购');"];

	```
	
### 解决办法

1. JS Alert方法延时
	 
	```
	JavaScript:

	function showAlert() {
		setTimeout(function(){
			alert('秒杀5折起，超性价比抢购')
		},100)
	}


	OC:

	[self.webView stringByEvaluatingJavaScriptFromString:@"showAlert()"];

	```

2. OC直接执行Alert延时

	```
	[self.webView stringByEvaluatingJavaScriptFromString:@"setTimeout(function(){alert('秒杀5折起，超性价比抢购，更有群“师”乱舞大趴体')},100);"];

	```

3. OC调用alert直接执行: 在主线程，不用等showAlert完成，所以不会死锁，该方法js不用延时
    ```
    [self performSelectorOnMainThread:@selector(showAlert) withObject:nil waitUntilDone:NO];

    - (void)showAlert{
        [self.webView stringByEvaluatingJavaScriptFromString:@"showAlert()"];
    }
    ```


![performSelectorOnMainThread用法](http://upload-images.jianshu.io/upload_images/1216462-b4d5af60accadbc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 现象分析

UIWebView在代码调用alert的时候，执行JS方法和弹出alert，都在主线程，产生死锁的现象(多个主队列同步任务执行)。所以通过js延迟弹出alert，让其他的任务完成后，弹出方可避免。
