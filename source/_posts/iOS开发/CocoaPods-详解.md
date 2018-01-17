---
title: CocoaPods-详解
date: 2016.02.05 19:43
tags: iOS开发
---
>What is cocoaPods?

>CocoaPods is a dependency manager for Swift and Objective-C Cocoa projects. It has over ten thousand libraries and can help you scale your projects elegantly. Interested in the news about Swift Package Manager? 

CocoaPods 是用来管理Swift和OC项目的，它已经有1万多个第三方库，可以优雅的帮助你扩充你的项目，并且热衷于Swift方面的报的管理。PS：翻译有点蛋疼...

CocoaPods主要是用来管理第三方库的一个工具，主要运行在Ruby环境下，通过命令行还以很方便的来管理我们项目工程所需要的第三方框架。当一个框架是MRC当时我们的项目是ARC的话，CocoaPods很方便的解决了这个ARC和MRC的转换。当一个框架依赖于其它的框架或者库的时候，CocoaPods都很方便的为我们解决了。GitHub上的有些框架只支持CocoaPods，对于我们iOS开发者来说简直是一个利器。

## CocoaPods安装
安装官网的包经常翻不过去墙，还好有淘宝！

```
 #升级Ruby
 $ sudo gem update --system

 # 删除源(这个系统自带的不好用)
 $ sudo gem sources -r https://rubygems.org/

 # 添加源(使用淘宝的镜像,记住要用https)
 $ sudo gem sources -a https://ruby.taobao.org/

 # 查看是否使用的是淘宝镜像
 $ gem sources -l

 # 安装
 # macOS Sierra 10.12.5 过期：$ sudo gem install cocoapods
 $ sudo gem install -n /usr/local/bin cocoapods --pre

 # 查看版本，检测是否安装成功
 $ pod --version

 # 下载pods信息
 $ pod setup
```

##安装过程中常见的问题

```
[!] Pod::Executable clone
'https://github.com/CocoaPods/Specs.git' master

xcrun: error: active developer path
("/Users/xiakejie/工具/Xcode 2.app/Contents/Developer")
does not exist, use xcode-select to change

解决上面这个问题, 使用这个命令: sudo xcode-select -switch
/Applications/Xcode.app/Contents/Developer

```

## CocoaPods配置

初次使用CocoaPods的时候，首先打开终端，`cd`到你的项目的和`项目名称.xcodeproj`同级的目录下，然后在终端直接敲命令`pod init`，这个命令会初始化一个`Podfile`的文件，这个文件非常重要，里面保存着你项目中所需要的所有的第三方库的信息(名称和版本)

初始化的Podfile里面的信息：

```
  # Uncomment this line to define a global platform for your project
  # 取消注释下面这行，来指定一个你的项目支持的平台以及最低版本
  # platform :ios, '8.0'
  # Uncomment this line if you're using Swift
  # 当你使用Swift的时候取消注释下面这行
  # use_frameworks!

  target 'pod-test' do

  end

```

标准的Podfile里面的信息：

```
  platform :ios, '8.0'
  use_frameworks!

  target 'MyApp' do
    pod 'AFNetworking', '~> 2.6'
    pod 'ORStackView', '~> 3.0'
    pod 'SwiftyJSON', '~> 2.3'
  end

```

简写的Podfile里面的信息：

```
  platform :ios, '8.0'
  use_frameworks!

  pod 'AFNetworking'
  pod 'SDWebImage'
  pod 'SVProgressHUD'

```

Podfile中版本信息说明：

```
 pod 'AFNetworking', '>= 3.2.X' 会根据本地pod库，导入不低于 3.2.X 版本的第三方库

 pod 'AFNetworking', '~> 3.2.X' 会根据本地的pod库，导入介于 3.2.X~3.3.0 之前版本的第三方库

 pod 'AFNetworking', '3.2.X' 锁定第三方库，库的版本不会根据本地的pod库的变化而变化

```

## CocoaPods 使用


```
  # 安装项目所需要的第三方库
  $ pod install
  
  # 终端提示完成(complete)的时候，打开项目白色的那个项目工程文件,使用终端命令打开为
  $ open 你的项目名.xcworkspace
  
  # OC里面导入第三方库 
  #import <Reachability/Reachability.h>
  
  # 也可以使用(推荐用上面的导入方法)
  #import "Reachability/Reachability.h"

```

## CocoaPods 常用命令详解

```
$ pod install : 更新本地pod库，并且下载Podfile里的框架，创建一个Xcode的Pods库

```

```
$ pod init : 在一个Xcode项目里面创建一个Podfile文件
     
```

```
$ pod update : 更新本地pod库，并且下载Podfile里的第三方库
    
```

```
$ pod search SDWebImage: 根据关键字在本地pod库搜索第三方库

```

```
$ pod install --verbose --no-repo-update : 根据Podfile文件，在本地pod库当中搜索所需要的第三方库，不会从网络上获取，不会更新本地pod库

类似：
  $ pod update --verbose --no-repo-update
  $ pod install --no-repo-update
  $ pod update --no-repo-update

```

```  
  $ pod repo update ：更新本地pod库
```
