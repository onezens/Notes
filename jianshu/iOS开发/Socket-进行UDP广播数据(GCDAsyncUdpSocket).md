在做一个App的时候，有一个需求是服务端在局域网当中发送广播数据，然后连接到此局域网当中的客户端接收到广播数据后，对广播数据做出相应地回复。在这个需求当中广播的唯一的好处就是客户端连接到局域网当中不需要知道服务端的IP地址，客户端通过收到服务端的广播消息之后，从广播报文当中获取到服务端的IP地址。唯一的不足是广播是一个耗能的操作，要控制好广播时间和广播的频率。

客户端和服务端之间通信我们当然要用socket通信了，建立一个socket通信管道，然后绑定端口，然后监听端口就行了。从GitHub上下载框架[GCDAsyncUdpSocket](https://github.com/robbiehanson/CocoaAsyncSocket/tree/master/Source/GCD)来实现UDP的无连接广播功能。废话不多说了，代码撸起来...

**1.通过 touch begin 方法来启动一个socket服务**

```
-(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{
    
    GCDAsyncUdpSocket *socket = [[GCDAsyncUdpSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)];
    
    NSError *error = nil;
    
    //绑定本地端口
    [socket bindToPort:54321 error:&error];
    
    if (error) {
        NSLog(@"1:%@",error);
        return;
    }
    
    //启用广播
    [socket enableBroadcast:YES error:&error];
    
    if (error) {
        NSLog(@"2:%@",error);
        return;
    }
    
    //开始接收数据(不然会收不到数据)
    [socket beginReceiving:&error];
    
    if (error) {
        NSLog(@"3:%@",error);
        return;
    }

    self.socket = socket;
    
    //重复发送广播
    NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(broadcast) userInfo:nil repeats:YES];
    
    
    [timer fire];
    
    
}

- (void)broadcast{
    
    NSString *msg = @"Hello world!";
    
    [self.socket sendData:[msg dataUsingEncoding:NSUTF8StringEncoding] toHost:@"192.168.0.255" port:54321 withTimeout:10 tag:100];
    
}
```

上面的代码主要是创建了一个socket实例，这个实例绑定了一个端口并且处于监听的状态，在类扩展里面对socket实例进行一个强指针引用，防止被释放。在广播的时候IP地址192.168.0.255是一个广播地址，这个地址可以通过自己手机的IP地址和子网掩码相与进行计算出来，实在不知道可以放一个所有地址的广播地址255.255.255.255 。

**2. 设置代理后，实现代理方法。代理方法当中有一个方法是监听到发送到当前socket的数据。**

```
#pragma mark - socket delegate

- (void)udpSocket:(GCDAsyncUdpSocket *)sock didReceiveData:(NSData *)data fromAddress:(NSData *)address withFilterContext:(id)filterContext{
    
    NSString *msg = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    
    NSLog(@"%@",msg);
}
```
因为只是一个广播数据，所以不需要在写一个socket客户端来获取局域网当中广播数据，自己就可以接收广播数据。在自己个需求中我们也可以通过代理中传过来的address 或者 filterContext 来过滤数据，但是在此文中不需要。

**3. 运行结果**
```
2015-12-20 23:50:48.462 广播数据测试[5869:339268] Hello world!
2015-12-20 23:50:48.462 广播数据测试[5869:339386] Hello world!
2015-12-20 23:50:48.462 广播数据测试[5869:339261] Hello world!
2015-12-20 23:50:48.462 广播数据测试[5869:339385] Hello world!
2015-12-20 23:50:48.463 广播数据测试[5869:339388] Hello world!
2015-12-20 23:50:48.463 广播数据测试[5869:339268] Hello world!
2015-12-20 23:50:48.464 广播数据测试[5869:339390] Hello world!
2015-12-20 23:50:48.465 广播数据测试[5869:339386] Hello world!
2015-12-20 23:50:48.465 广播数据测试[5869:339391] Hello world!
2015-12-20 23:50:48.466 广播数据测试[5869:339261] Hello world!
2015-12-20 23:50:48.466 广播数据测试[5869:339393] Hello world!
```

**4 . 注意事项**

（1）监听的端口(绑定端口)和发送到的目的端口要一致。

（2）如果要进行广播数据，必须只能使用[socket bindToPort:54321 error:&error] 方法来绑定端口，不能绑定IP地址并且启用广播，否则不能广播数据。

（3）该实例没有被销毁前，只能被创建一次，因为端口正在被使用，可以用一个懒加载优化下。

（4）注意NSTimer 的销毁防止内存泄露。
