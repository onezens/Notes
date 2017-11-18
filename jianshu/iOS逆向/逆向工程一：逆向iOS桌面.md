当iPhone启动的时候，我们所看到的界面以及进行交互的界面是系统的springboard进程，bundle id为com.apple.springboard。通过ssh登录iPhone`ssh root@192.168.2.24`, 查看进程信息`ps -e | grep SpringBoard`。

### 一. 创建theos工程
1. 创建tweak工程 `/opt/theos/bin/nic.pl ` 
```
NIC 2.0 - New Instance Creator
------------------------------
  [1.] iphone/activator_event
  [2.] iphone/application_modern
  [3.] iphone/cydget
  [4.] iphone/flipswitch_switch
  [5.] iphone/framework
  [6.] iphone/ios7_notification_center_widget
  [7.] iphone/library
  [8.] iphone/notification_center_widget
  [9.] iphone/preference_bundle_modern
  [10.] iphone/tool
  [11.] iphone/tweak
  [12.] iphone/xpc_service
Choose a Template (required): 
```
2. 选择11创建tweak工程

```
//项目名称    
Project Name (required): demo 
//项目的包名，即bundle id
Package Name [com.yourcompany.demo]: cc.onezen.demo
//开发者名称
Author/Maintainer Name [wz]: wz 
//需要注入的进程的 bundle id
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]: com.apple.springboard
//deb包安装完成后需要重启的进程名字
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]: SpringBoard    
Instantiating iphone/tweak in demo/...
Done.
```

### 二. theos工程文件
1. Makefile 默认生成信息如下
```
#theos通用头文件
include $(THEOS)/makefiles/common.mk
#项目名 -> Project Name
TWEAK_NAME = demo
#tweak包含的源文件，指定多个文件时用空格隔开
demo_FILES = Tweak.xm

#tweak工程的头文件，一般有application.mk、tweak.mk和tool.mk几类
include $(THEOS_MAKE_PATH)/tweak.mk

#安装之后，需要做的事情，这里是杀掉SpringBoard进程，SpringBoard是系统进程杀掉会重启
after-install::
	install.exec "killall -9 com.apple.springboard"
```

2. tweak.xm文件：`xm`中的`x`代表这个文件支持Logos语法，如果后缀名是单独一个`x`，说明源文件支持Logos和C语法；如果后缀名是`xm`，说明源文件支持Logos和C/C++语法。

    * %hook 指定需要hook的class，必须以%end结尾
    * %log 该指令在%hook内部使用，将函数的类名、参数等信息写入syslog
    * %orig该指令在%hook内部使用，执行被hook的函数的原始代码

```
/* How to Hook with Logos
Hooks are written with syntax similar to that of an Objective-C @implementation.
You don't need to #include <substrate.h>, it will be done automatically, as will
the generation of a class list and an automatic constructor.

%hook ClassName

// Hooking a class method
+ (id)sharedInstance {
	return %orig;
}

// Hooking an instance method with an argument.
- (void)messageName:(int)argument {
	%log; // Write a message about this call, including its class, name and arguments, to the system log.

	%orig; // Call through to the original function with its original arguments.
	%orig(nil); // Call through to the original function with a custom argument.

	// If you use %orig(), you MUST supply all arguments (except for self and _cmd, the automatically generated ones.)
}

// Hooking an instance method with no arguments.
- (id)noArguments {
	%log;
	id awesome = %orig;
	[awesome doSomethingElse];

	return awesome;
}

// Always make sure you clean up after yourself; Not doing so could have grave consequences!
%end
*/
```
3. control 文件记录了deb包所需的基本信息，会被打包进deb包里
```
Package: cc.onezen.demo
Name: demo
Depends: mobilesubstrate
Version: 0.0.1
Architecture: iphoneos-arm
Description: An awesome MobileSubstrate tweak!
Maintainer: wz
Author: wz
Section: Tweaks
```

### 三. 手动安装包
根据上面介绍的默认生成的Makefile，是手动安装deb包的配置。
1. 修改Tweak里的代码
```
%hook SpringBoard

- (void)_menuButtonDown:(struct __IOHIDEvent *)arg1 {
	%orig;
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Hello" message:@"wz hook the iphone test" delegate:nil cancelButtonTitle:@"owesome" otherButtonTitles:nil, nil];
    [alert show];
    [alert release];
}
%end
```

2. 编译：`make`
3. 打包：`make package`
4. 拷贝到iPhone `scp packages/cc.onezen.demo_0.0.1-1+debug_iphoneos-arm.deb root@192.168.2.24:/tmp`
5.  切换到iphone上安装： `dpkg -i /tmp/cc.onezen.demo_0.0.1-1+debug_iphoneos-arm.deb `
6. 重启springboard `killall -9 SpringBoard`。这时候会出现白苹果，几秒后会重新到桌面
7. 点击home键，效果图

![1.png](http://upload-images.jianshu.io/upload_images/1216462-49afc60f7ec5a330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四. 自动安装包
1. 修改Makefile文件
```
#debug=0是release包
DEBUG = 0
#iphone的ip地址
THEOS_DEVICE_IP = 192.168.2.24
#当前包支持的cpu架构
ARCHS = armv7 arm64 
#支持的ios版本
TARGET = iphone:latest:8.0  
include $(THEOS)/makefiles/common.mk

TWEAK_NAME = demo
demo_FILES = Tweak.xm
#需要导入的库
demo_FRAMEWORKS = UIKit 
demo_PRIVATE_FRAMEWORKS = AppSupport
include $(THEOS_MAKE_PATH)/Tweak.mk

#clean 是指调用make clean的时执行的操作
after-install::
	install.exec "killall -9 SpringBoard"
clean::
	rm -rf ./packages/*
	rm -rf ./.theos/*

```
2. 编译`make`
3. 安装`make package install`
4. 输入两次密码后到桌面
```
➜  demo make package install
> Making all for tweak demo…
make[2]: Nothing to be done for `internal-library-compile'.
> Making stage for tweak demo…
dpkg-deb: warning: deprecated compression type 'lzma'; use xz instead
dpkg-deb: warning: ignoring 1 warning about the control file(s)
dpkg-deb: building package 'cc.onezen.demo' in './packages/cc.onezen.demo_0.0.1-2_iphoneos-arm.deb'.
==> Installing…
root@192.168.2.24's password: 
(Reading database ... 4547 files and directories currently installed.)
Preparing to unpack /tmp/_theos_install.deb ...
Unpacking cc.onezen.demo (0.0.1-2) over (0.0.1-1+debug) ...
Setting up cc.onezen.demo (0.0.1-2) ...
install.exec "killall -9 SpringBoard"
root@192.168.2.24's password: 
```

### 五. deb包
1. 删除原先的包 `dpkg -r cc.onezen.demo` 然后手动重启 `killall -9 SpringBoard`
2. 查看包的结构：`dpkg -c cc.onezen.demo_0.0.1-1+debug_iphoneos-arm.deb `
    ```
    drwxr-xr-x wz/staff          0 2017-09-21 18:19 ./
    drwxr-xr-x wz/staff          0 2017-09-21 18:19 ./Library/
    drwxr-xr-x wz/staff          0 2017-09-21 18:19 ./Library/MobileSubstrate/
    drwxr-xr-x wz/staff          0 2017-09-21 18:19 ./Library/MobileSubstrate/DynamicLibraries/
    -rwxr-xr-x wz/staff     116368 2017-09-21 18:19 ./Library/MobileSubstrate/DynamicLibraries/demo.dylib
    -rw-r--r-- wz/staff         57 2017-09-21 18:19 ./Library/MobileSubstrate/DynamicLibraries/demo.plist
    ```
  3. dpkg 打包的时候会把theos根目录下的layout文件夹，映射到安装的设备的根目录下。
  4. 脚本文件
      ```
      preinst
      在Deb包文件解包之前，运行的脚本。许多“preinst”脚本的任务是停止作用于待升级软件包的服务，直到软件包安装或升级完成。

      postinst
      该脚本的主要任务是完成安装包时的配置工作。许多“postinst”脚本负责执行有关命令为新安装或升级的软件重启服务。

      prerm
     该脚本负责停止与软件包相关联的daemon服务。它在删除软件包关联文件之前执行。

      postrm
      该脚本负责修改软件包链接或文件关联，或删除由它创建的文件。
      ```

六. 相关命令

1. 查看签名信息和或者bundle id ： `codesign -dvvv WeChat`

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
