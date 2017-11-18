### 一. 逆向开发工具
1. 可越狱iPhone一台
2. 电脑一部
3. 诸多软件等

### 二. iPhone越狱
设备详情：

1. iPhone 5s Version: iOS 9.3.2 13F69
2. Mac电脑一台，VMware Fusion， Win7
爱思助手，pp越狱助手
3. 越狱注意点：

     * 部分iOS版本并不支持越狱，比如：iOS8.4.1, iOS9.1不支持越狱。所以选择/购买机器时，查询对应机型的版本是否支持越狱 
     * 注意版本编号，有些版本有beta版，也是不支持越狱，注意对应的系统版本和版本编号的统一。13F69 是iOS9.3.2的一个版本编号
     * 选购机器最好选择5s以上的机型，因为32位(armv7, armv7s等)的机型已经停止越狱的支持
   * 最初越狱的时候，一直用的爱思，但是用虚拟机一直越狱失败。换了pp越狱助手，虚拟机越狱成功。(pp和爱思都是用的盘古越狱)
   * iOS9.3.2是不完全越狱，即：重启手机，越狱失效。需要重新越狱。

 
### 三. 逆向环境搭建
使用pp越狱成功后，会自动安装Cydia，通过Cydia可以安装一些越狱的时候使用的工具

通过Cydia安装软件：

 1. OpenSSH 默认登录名和密码：root alpine 
     * 登录iPhone `ssh root@ip `   ip为iPhone的ip地址
    * scp拷贝文件：
	  
	    ``` 		
	    scp filename root@ip:/tmp    ;将本地文件拷贝到远程tmp文件夹下
	    scp root@ip:/tmp/1.txt  /tmp  ;将远程tmp文件夹下的1.txt 拷贝到本地tmp文件夹下
	   ```

    * 更改root密码：  
	
		```
		passwd  ;更改当前登录的用户密码
		passwd root  ;更改root用户密码
		```
2. usbmuxd工具
有时候使用ip连接ssh速度会很慢，或者在动态调试的时候，启动debugserver也会很慢。通过usb连接会解决这些问题。
    * 下载连接：[https://cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz](https://cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz)
    * 转发iPhone ssh 22号端口到localhost 2222
     ```
     ./tcprelay.py -t 22:2222
     Forwarding local port 2222 to remote port 22

    //ssh进行连接iPhone
    ssh root@localhost -p 2222
     ```
    * iPhone debugserver 9527 转发到到本地9527
    ```
    ./tcprelay.py -t 9527:9527

    //iPhone启动debugserver
    debugserver *:9527 -a "WeChat" 

    //mac 连接debugserver
    process connect connect://localhost:9527

    ```


3. 安装软件管理包apt-
    * 在Cydia中搜索 `APT 0.6 Transitional`并
    * 命令介绍
 	
 		```
 		apt-get update                    ;更新所有的源;

        apt-get upgrade                 ;更新所有通过apt-get安装的程序

       apt-get install  packagename         ;安装软件包

       apt-get remove  packagename      ;删除软件包，不删除依赖包，不删除配置文件;

       apt-get remove --purge packagename  ;删除该软件包，不删除依赖包，删除配置文件

       apt-get autoremove packagename       ;删除该软件包，删除依赖包，不删除配置文件

       apt-get autoremove --purge packagname    ;可以删除所有依赖包+配置文件

       apt-cache search string             ;搜索含有该string字段的软件包

       apt-cache show packagename     ;详细显示该软件包的信息

       apt-get clean                   ;清除apt-get安装的软件包备份，可以释放储存空间，不影响软件正常使用;
 		```
    * 使用apt-get 安装必要安装包
 		
 		```
 	   apt-get install  ping

        apt-get install  ps

        apt-get install  find
  
        apt-get install tcpdump

        apt-get install top

        apt-get install vim

        apt-get install  network-cmds   //-arp, ifconfig, netstat, route, traceroute
     
 		```
 		
    * 工具使用测试
 	
 		```
 		 ping www.baidu.com ;发送icmp报文
 		 ps aux / ps -e ;查看进程详细信息
 		 find / -name filename  ;在更目录下查找filename的文件
 		 grep -r 'hello*' /tmp ;在tmp文件夹下查找包含hello的文件，-r遍历查找
 		 top -l 1 | head -n 10 | grep PhysMem ;查看内存使用情况
 		 
 		 tcpdump -i en0 src host 192.168.1.137
  		 tcpdump -i en0 dst host 192.168.1.137
         tcpdump -w /tmp/ssh.cap -i en0 tcp port 22 and dst host 192.168.1.137
         tcpdump -w /tmp/ssh.cap -i en0 -t -s 0 -c 100 tcp port ! 22 and dst host 192.168.1.137
         解释:
         (1)-w /tmp/ssh.cap : 保存成cap文件，方便用ethereal(即wireshark)分析
         (2)-i en0 : 只抓经过接口en0的包
         (3)-t : 不显示时间戳
         (4)-s 0 : 抓取数据包时默认抓取长度为68字节。加上-S 0 后可以抓到完整的数据包
         (5)-c 100 : 只抓取100个数据包
         (6) tcp port ! 22  : 不抓tcp端口22的数据包
         (7) dst port ! 22 : 不抓取目标端口是22的数据包
         (8)dst host 192.168.1.137 : 抓取目标地址为192.168.1.137的包
         (9)src net 192.168.1.0/24 : 数据包的源网络地址为192.168.1.0/24
 		```


### 四. iOS文件系统结构

1. UNIX常见系统目录
```
/：磁盘根目录

/bin："binary"的简写，存放提供用户级基础功能的二进制文件，如ls、ps等

/boot：存放能使系统成功启动的所有文件。iOS中此目录为空。

/dev："device"的简写，存放BSD设备文件。每个文件代表系统的一个块设备或字符设备，一般来说，“块设备”以块为单位传输数据，如硬盘；而“字符设备”以字符为单位传输数据，如调制解调器。

/sbin："system binaries"的简写，存放提供系统级(root使用)基础功能的二进制文件，如netstat、reboot等。

/etc："Et Cetera"的简写，存放系统脚本及配置文件，如passwd、hosts等。在iOS中，/etc是一个符号链接，实际指向/private/etc。

/lib：存放系统库文件、内核模块及设备驱动等。iOS中此目录为空。

/mnt："mount"的简写，存放临时的文件系统挂载点。iOS中此目录为空。加载光盘、U盘等。

/private：存放两个目录，分别是/private/etc和/private/var。

/tmp：临时目录。在iOS中，/tmp是一个符号链接，实际指向/private/var/tmp。

/usr：包含了大多数用户工具和程序。/usr/bin包含那些/bin和/sbin中未出现的基础功能，如nm、killall等；/usr/include包含所有的标准C头文件；/usr/lib存放库文件。

/var："variable"的简写，存放一些经常更改的文件，比如日志、用户数据、临时文件等。其中/var/mobile和/var/root分别存放了mobile用户和root用户的文件，是重点关注的目录。

```

2. iOS的独有目录

```
/Applications：存放所有的系统App和来自于Cydia的App，但不包括AppStore

/Developer：如果一台设备连接Xcode后被指定为调试用机，Xcode就会在iOS中生成这个目录，其中会含有一些调试需要的工具和数据。

/Library：存放一些提供系统支持的数据。其中/Library/MobileSubstrate下存放了所有基于CydiaSubstrate（原名MobileSubstrate）的插件（如：tweak编写的插件）。

/System/Library：iOS文件系统中最重要的目录之一，存放大量系统组件。

/System/Library/Frameworks和/System/Library/PrivateFrameworks：存放iOS中的各种framework

/System/Library/CoreServices里的SpringBoard.app：iOS桌面管理器（类似于Windows里的explorer），是用户与系统交流的最重要中介。

/User：用户目录（其实就是mobile用户的home目录），实际指向/var/mobile，这个目录里存放大量用户数据，比如：

/var/mobile/Media/DCIM下存放照片；

/var/mobile/Media/Recordings下存放录音文件；

/var/mobile/Library/SMS下存放短信数据库；

/var/mobile/Library/Mail下存放邮件数据。

/var/mobile/Containers/Bundle 存放AppStore下载的APP的Bundle起始目录

/var/mobile/Containers/Data存放AppStore下载的APP的沙盒起始目录
```
3. 文件目录权限
    * `rwxr-xr-x ` 每三个分成三组，分别代表：所有者(rwx)，所属的用户组(r-x), 其它用户(r-x)
    * 每一组三位可以看做三个二进制，每位意义为：是否有读权限(r)，是否有写权限(w)，是否有执行权限(x)；即：所有者拥有权限 rwx = 111 = 7
    * 修改权限 : `chmod  777  filename` 给与filename所有的权限即：`rwxrwxrwx`

4. 获取App的目录
    * 获取Bundle目录
      ```
      ps -e | grep WeChat
      1368 ?? 5:41.43 /var/mobile/Containers/Bundle/Application/749DC69A-3A8D-4B5C-9926-1220E69FC85F/WeChat.app/WeChat
      ```

    * 获取沙盒目录
      ```
       cycript -p WeChat
       cy# directory = NSHomeDirectory()
       @"/var/mobile/Containers/Data/Application/986376B5-EF08-4CAF-81FB-CAE48D1FE4AE"
      ```

### 五. Cycript
作者： saurik 
官网： [http://www.cycript.org/](http://www.cycript.org/)

1. Cycript是一款脚本语言，可以看作是Objective-JavaScript，它可以帮助我们轻松测试和验证函数效果
    * 在越狱手机中可以通过注入方式在第三方应用中运行
    * 也可以用静态库的方式把cycript集成到自己的应用(MonkeyDev，可以给非越狱iOS第三方App写插件，但是权限受沙盒限制)

2. 越狱手机安装Cycript
    * 在Cydia上搜索Cycript进行安装
    * `apt-get install cycript`

3. 注入到第三方进程空间，`CMD+D`退出cycript
```
//确认进程名或者进程PID
root# ps -e | grep WeChat 
1368 ?? 6:17.44 /var/mobile/Containers/Bundle/Application/749DC69A-3A8D-4B5C-9926-1220E69FC85F/WeChat.app/WeChat

//方式1
root# cycript -p WeChat

//方式2
root# cycript -p 1368  
```

4. 注入实例:(代码最好在Xcode里写，然后复制)
```
//WeChat
[[[UIAlertView alloc]initWithTitle:@"Cyscript" message:@"inject wechat test" delegate:ni cancelButtonTitle:@"confirm" otherButtonTitles:nil, nil] show];

//Springboard
[[SBScreenShotter sharedInstance] saveScreenshot:YES]   截屏，闪光
[[SBScreenShotter sharedInstance] saveScreenshot:NO]   截屏，不闪光
[[SBScreenFlash mainScreenFlasher] flashColor:[UIColor magentaColor] withCompletion:nil] 屏幕闪紫色光
```

### 六. 逆向工具
1. 逆向工具大致可分为四类：
    * 检测工具。如：Reveal、tcpdump等
    * 反编译工具。如：IDA、Hopper Disassembler、classdump等
    * 调试工具。如：lldb、Cycript等
    * 开发工具。如：Xcode、theos等

2. classdump 导出没有加密过得头文件，借助于otool。      
    * 官网： [http://stevenygard.com/projects/class-dump/](http://stevenygard.com/projects/class-dump/)
    * 用法： `class-dump -s -S -H WeChat -o ./wechatHeaders` 将砸壳后的wechat文件的头文件导出到wechatHeaders目录
    * 判断是否加密 `otool -l WeChat | grep -B 2 crypt`  cryptid 0 已经解密
      ```
      --
      cmd LC_ENCRYPTION_INFO_64
      cmdsize 24
      cryptoff 16384
      cryptsize 52379648
      cryptid 0
      ```
    * 查看某文件夹下文件的个数，包括子文件夹里的： `ls -lR|grep "^-"|wc -l`
    * 查看某文件夹下文件夹的个数，包括子文件夹里的:   `ls -lR|grep "^d"|wc -l`

3. dumpdecrypted 砸壳工具
    * 下载源码： git clone [https://github.com/stefanesser/dumpdecrypted.git](https://github.com/stefanesser/dumpdecrypted.git)
    * 编译：make 生成 dumpdecrypted.dylib 文件
    * 用法：将dumpdecrypted.dylib 文件拷贝到越狱iPhone的需要砸壳的App沙盒的Documents文件夹下面
      ```
      例如微信：
      # cd /var/mobile/Containers/Data/Application/986376B5-EF08-4CAF-81FB-CAE48D1FE4AE/Documents/
      # cp /tmp/dumpdecrypted.dylib .
      # DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/749DC69A-3A8D-4B5C-9926-1220E69FC85F/WeChat.app/WeChat
      ```

4. theos 越狱工具开发包
**百科：**
[http://iphonedevwiki.net/index.php/Theos/Setup#For_Mac_OS_X](http://iphonedevwiki.net/index.php/Theos/Setup#For_Mac_OS_X)
[http://iphonedevwiki.net/index.php/Theos](http://iphonedevwiki.net/index.php/Theos)

    **安装教程**： [https://github.com/theos/theos/wiki/Installation](https://github.com/theos/theos/wiki/Installation)

    **安装步骤简介：**
    * `sudo mkdir /opt` 如果没有opt目录则创建，有cd到该目录
    * `git clone --recursive https://github.com/theos/theos.git`
    * `sudo chown -R $(id -u):$(id -g) theos`  
  
    **设置环境变量：**  
      * 在新建theos工程编译的时候执行：`export THEOS=/opt/theos`
      * 开机启动时候执行
        ```
        vi ~/.bash_profile
        写入： export THEOS=/opt/theos

        调用： source ~/.bash_profile
        测试： echo $THEOS
        ```

5. ldid 
越狱iPhone下的签名工具（更改授权entitlements），可以为thos开发的程序进程签名（支持在OS X和iOS上运行）
wiki： [http://iphonedevwiki.net/index.php/Ldid](http://iphonedevwiki.net/index.php/Ldid)

    * 安装： `brew install ldid fakeroot`
    * 查看WeChat签名信息 `codesign -dvvv WeChat`
      ```
      Executable=/Users/leaf/yy/WeChat
      Identifier=com.tencent.xin
      Format=Mach-O thin (arm64)
      CodeDirectory v=20200 size=511823 flags=0x0(none) hashes=15987+5 location=embedded
      Hash type=sha256 size=32
      CandidateCDHash sha1=d6e410a862077728c19fb53f6938e1bbbc723202
      CandidateCDHash sha256=0925ea0e119b16a34f890d85ad8e8a4781d4d984
      Hash choices=sha1,sha256
      CDHash=0925ea0e119b16a34f890d85ad8e8a4781d4d984
      Signature size=4297
      Authority=Apple iPhone OS Application Signing
      Authority=Apple iPhone Certification Authority
      Authority=Apple Root CA
      Info.plist=not bound
      TeamIdentifier=88L2Q4487U
      Sealed Resources=none
      Internal requirements count=1 size=96
      ```
    * 查看entitlement内容  `codesign -d --entitlements - WeChat` 或
`ldid -e WeChat`
    * 修改entitlement内容 `ldid -Sentitlement.xml  WeChat`

6. dpkg工具
    * 安装
      ```
      brew install --from-bottle https://raw.githubusercontent.com/Homebrew/homebrew-core/7a4dabfc1a2acd9f01a1670fde4f0094c4fb6ffa/Formula/dpkg.rb

      brew pin dpkg
      ```
    * 安装/卸载 `dpkg -i/-r cc.onezen.demo1_0.0.1-1+debug_iphoneos-arm.deb`
    * 查看安装包信息 `dpkg -s cc.onezen.demo1`


### 七. 日志功能
  **注意要想看日志信息，一定要在Makefile里设置DEBUG=1，否则除了NSLog信息可以看到外，其它的信息看不到！**
1. syslog：iOS9.3.2没有配成功
    * 安装：在Cydia搜索 `syslogd to /var/log/syslog`
    * 查看日志： `tail -f /var/log/syslog`
    * 过滤查看： `tail -f /var/log/syslog | grep WeChat`


2. socat
    * 安装：在Cydia搜索`SOcket CAT`
    * 运行：`socat - UNIX-CONNECT:/var/run/lockdown/syslog.sock`
    * 运行起来后，输出help查看用法
      ```
      help
      Commands
          quit                 exit session
          select [val]         get [set] current database
                               val must be "file" or "mem"
          file [on/off]        enable / disable file store
          memory [on/off]      enable / disable memory store
          stats                database statistics
          flush                flush database
          dbsize [val]         get [set] database size (# of records)
          watch                print new messages as they arrive
          stop                 stop watching for new messages
          raw                  use raw format for printing messages
          std                  use standard format for printing messages
          *                    show all log messages
          * key val            equality search for messages (single key/value pair)
          * op key val         search for matching messages (single key/value pair)
          * [op key val] ...   search for matching messages (multiple key/value pairs)
                         operators:  =  <  >  ! (not equal)  T (key exists)  R (regex)
                         modifiers (must follow operator):
                                 C=casefold  N=numeric  S=substring  A=prefix  Z=suffix
      ```
    * 过滤查看日志(根据bundle id)：`* Facility com.apple.springboard`
    * 过滤查看日志(根据进程id)：`* PID 123`
    * 过滤条件
      ```
      ASLMessageID, //asl消息id
      Level, //级别
      Time, //时间
      TimeNanoSec, //时间ns
      Sender, //消息的发送者
      Facility, //按照Bundle ID来过滤日志
      PID, //进程id
      Message, //消息内容
      ReadUID //读取所使用的用户id
      ```


![socats](http://upload-images.jianshu.io/upload_images/1216462-340c3e92dd0c02dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. deviceconsole(更喜欢这种日志)
 通过usb连接打印iPhone日志。克隆相关源码，然后make编译后，会在mac上自动创建deviceconsole命令。在终端输入deviceconsole，自动输出iphone上的日志。过滤输出： `deviceconsole -i -f WeChat`
    ```
    git clone https://github.com/MegaCookie/deviceconsole
    make
    ```


![deviceconsole](http://upload-images.jianshu.io/upload_images/1216462-ed5179e45eb562e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
