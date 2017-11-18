iOS的国际化功能可以使APP很方便的在不同国家的不同语言之间进行切换，大大的方便了APP走向国际。国际化的时候主要分为三个方面的国际化：InfoPlist、Xib/Storyboard以及代码的国际化。  

最新脚本地址： https://github.com/onezens/AutoLocalization

##脚本升级记录
 1. 2017.06.02 自动化脚本在原先作者的基础上进行修改，现在同一个文件，支持xib和storyboard的一次性国际化
 2. 2017.8.10 遍历项目根目录下所有xib、storyboard文件并且国际化

##项目开启并且添加国际化语言
首先选中项目，然后在PROJECT中添加需要添加的不同的语言，添加完毕之后会跳出一个选中项目中所有的xib和storyboard对话框，点击finish即可。
![](http://upload-images.jianshu.io/upload_images/1216462-d403ec74bceac61d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##InfoPlist国际化

首先在项目中创建一个新的strings文件，默认InfoPlist的命名为`InfoPlist.strings`
![](http://upload-images.jianshu.io/upload_images/1216462-581b7c901f7ba739.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后开启该文件的国际化，选中该文件在右侧的工具视图里点击`Localize...`按钮开启
![](http://upload-images.jianshu.io/upload_images/1216462-2decd8d4f9958fca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后选择需要国际化的语言即可
![](http://upload-images.jianshu.io/upload_images/1216462-74ffd690df7888b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最后在不同语言的国际化文件添加对应语言，例如中英文显示不同的应用名称
```
英文文件(InfoPlist.strings(Chinese(Simplified)))：
"CFBundleDisplayName" = "Weibo";
中文文件(InfoPlist.strings(Chinese(Simplified)))：
"CFBundleDisplayName" = "微博";

```
通过进入系统设置，修改语言后查看对应的效果，**注意格式一定不能出错（"key" = "value";）！**

##代码国际化
代码国际化的时候默认创建国际化文件`Localizable.strings`，创建文件以及选择国际化语言的过程和InfoPlist创建的相同，所以下面直接演示示例。

例如：在代码中设置一个控制器的title
```
//代码
self.title = NSLocalizedString(@"首页", @"首页");

//国际化文件设置
英文："首页" = "Home";

```
**要点：NSLocalizedString传递的第一个参数是当前国际化文字的key，第二个参数是注释comment，这个参数可有可为nil；当国际化文件某一语言中没有对应的指定国际化信息的时候，默认会以key的值来显示，所以在上面示例中没有中文国际化的代码。**

##Xib和Storyboard国际化
Xib和Storyboard的国际化相对上面两种来说就麻烦了许多，因为添加控件之后每一个控件对应的内容(title、text等)不会动态的生成，需要每次开启相应XibhuoStoryboard的国际化来在国际化文件中生成对应属性的国际化内容。在查找相关资料的时候发现了一种使用脚本动态生成国际化的内容。

在xib中添加一个按钮
```
"TqI-wI-tvP.normalTitle" = "Button";
```
`TqI-wI-tvP `是添加的按钮的id，可以从按钮属性中查看
![](http://upload-images.jianshu.io/upload_images/1216462-878435c8f4009867.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`normalTitle`是该按钮对应的属性，可设置相应的国际化内容

##Xib和Storyboard自动国际化
> 自动化脚本在原作者的基本上，增加了2个新功能

步骤：

* 将存放脚本文件的文件夹，导入脚本文件到项目的根目录
* 选择项目 -> targes -> Build Phases -> + -> New Run Script Phase
* 添加两个运行脚本，其中一个添加脚本python (**注意路径，该路径实在项目根目录的RunScript文件夹下面**)
```
${SRCROOT}/${TARGET_NAME}/RunScript/AutoGenStrings.py ${SRCROOT}/${TARGET_NAME}
```
* Build Setting -> Deployment -> Deployment Location / Deployment Postprocessing -> Yes
* 分别创建一个xib和storyboard，然后开启国际化后，添加控件，Command + B 后查看你的国际化文件
![](http://upload-images.jianshu.io/upload_images/1216462-50f0ca5f46c6fed7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  

![添加脚本](http://upload-images.jianshu.io/upload_images/1216462-9f20ceca2e58bebb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  
**效果图**

![自动国际化过程](http://upload-images.jianshu.io/upload_images/1216462-c6978ae9c7814094.gif?imageMogr2/auto-orient/strip)


**注意**
文件路径中不能拥有空格，否则Xcode脚本会找不到文件的错误

**自动国际化脚本和demo：**[https://github.com/onezens/AutoLocalization](https://github.com/onezens/AutoLocalization)

## 参考
[[iOS国际化——通过脚本使storyboard翻译自增 - 东方木兮 - 博客园](https://www.baidu.com/link?url=O2GGX2iboUD1HN7Q7zY9max1yk7G-2vhoborILAwtShvMQ2tZKGKUNZmh9Xv7WcVDr2crtrOEBFr6TI5eNusRK&wd=&eqid=aa370aa000004f3e0000000359ae6dfa)](http://www.cnblogs.com/levilinxi/p/4296712.html)

## 扩展：App内切换语言
[http://www.jianshu.com/p/627da5a9992c](http://www.jianshu.com/p/627da5a9992c)
