---
title: iOS中,Framework和.a的打包及使用
date: 2017.03.07 14:25
tags: 
- Framework
---

最近在做一个小项目,需要给客户一个demo测试,有一部分核心代码暂时不想让客户知道,就想到了打包成framework或.a库。库有两种：

- 静态库：.a和.framework
- 动态库：.tbd和.framework

静态库和动态库的区别

- .a文件肯定是静态库，. tbd肯定是动态库，.framework可能是静态库也可能是动态库
- 静态库在链接时，会被完整的复制到可执行文件中，如果多个App都使用了同一个静态库，那么每个App都会拷贝一份，缺点是浪费内存。类似于定义一个基本变量，使用该基本变量是是新复制了一份数据，而不是原来定义的；
- 动态库不会复制，只有一份，程序运行时动态加载到内存中，系统只会加载一次，多个程序共用一份，节约了内存。类似于使用变量的内存地址一样，使用的是同一个变量；
- 但是项目中如果使用了自己定义的动态库，苹果是不允许上架的，在iOS8.0以后苹果开放了动态加载. tbd的接口，用于挂载. tbd动态库

使用静态库的好处

- 模块化，分工合作
- 避免少量改动经常导致大量的重复编译连接
- 也可以重用，注意不是共享使用

使用动态库的好处

- 使用动态库，可以将最终可执行文件体积缩小
- 使用动态库，多个应用程序共享内存中得同一份库文件，节省资源
- 使用动态库，可以不重新编译连接可执行程序的前提下，更新动态库文件达到更新应用程序的目的。

静态库的使用场景

- 保护自己的核心代码，自己不想别人看到的部分
- 将MRC的项目打包成静态库，可以在ARC下直接使用，不需要在转换

iOS设备的CPU架构
模拟器

- 4S-5:i386
- 5s-7P：x86_64

真机

- armv6:iPhone - iPhone3G
- armv7:iPhone 3Gs，4,4S,iPad，iPad2
- armv7s: iPhone 5、iPhone 5c <静态库只要支持了armv7,就可以在armv7s的架构上运行>
- arm64:iPhone 5s、iPhone 6、iPhone 6 Plus、iPhone 6s、iPhone 6s Plus、iPad Air、iPad Air2、iPad mini2、iPad mini3

> 没有armv64

下面言归正传，做点正事
### .a静态库
1.创建一个新的工程，选择下面这个模板：

![图1.png](http://upload-images.jianshu.io/upload_images/1009061-628c0d788e990aa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
完成

![图2.png](http://upload-images.jianshu.io/upload_images/1009061-63ef67a2d2c8e7f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.我这里就在Test操作了(亲，你打包.a的时候，可以删除默认文件，拉入自己想加入的任何文件)

![图3.png](http://upload-images.jianshu.io/upload_images/1009061-5ef82d6247c0ac95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图4.png](http://upload-images.jianshu.io/upload_images/1009061-39589baab4a891e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面在创建一个Person类


![图5.png](http://upload-images.jianshu.io/upload_images/1009061-805abe21238dacbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图6.png](http://upload-images.jianshu.io/upload_images/1009061-94c97cc80b4ee5e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.选择需要公开的头文件

- 添加头文件

![添加头文件.gif](http://upload-images.jianshu.io/upload_images/1009061-c9f3bdc83f52c6ea.gif?imageMogr2/auto-orient/strip)

4.修改配置

- `Build Active Architecture Only`修改为NO，否则生成的静态库就只支持当前选择设备的架构。

![图7.png](http://upload-images.jianshu.io/upload_images/1009061-fe383f115720272b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `iOS Deployment Target` ，静态库需要支持版本


![图8.png](http://upload-images.jianshu.io/upload_images/1009061-39fcebfbab4f7c1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `ach-O Type`更改成`Static Library`，.a默认就是`Static Library`，这一步可以省略


![图9.png](http://upload-images.jianshu.io/upload_images/1009061-e59a8084156723bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5.编译
选择`Generic iOS Device`和`任意一个模拟器`各编译一次，完成后，在工程的`Products`文件夹下的.a文件从红色编程了黑色。编译成功，然后`Show in Finder`。


![编译.gif](http://upload-images.jianshu.io/upload_images/1009061-a9a2d8022f8f37be.gif?imageMogr2/auto-orient/strip)

可以看到真机与模拟器都生成了.a。里面都有有我选择公开的头文件
这个时候，可以用命令`lipo -info 静态库名字`来看下，支持的iOS的CPU框架

![支持架构.gif](http://upload-images.jianshu.io/upload_images/1009061-20836ba2a70728d2.gif?imageMogr2/auto-orient/strip)
- `Debug-iphoneos`里面支持armv7、arm64,属于真机，用到模拟器就会报错
- `Debug-iphonesimulator`里面支持i386、x86_64，属于模拟器，用到真机会报错

6.合并静态库-让模拟器和真机使用一个静态库
命令如下：
`lipo -create第一个.a文件的绝对路径 第二个.a文件的绝对路径 -output 最终的.a文件路径`

![合成静态库.gif](http://upload-images.jianshu.io/upload_images/1009061-386f6e515f02fd0d.gif?imageMogr2/auto-orient/strip)

这个生成的`libTest.a`，就是支持真机和模拟器的静态库了。创建一个文件夹，把.a和头文件拖进去，这个文件夹就是我们所需要的。


![完成.gif](http://upload-images.jianshu.io/upload_images/1009061-325ad1d7aff2e271.gif?imageMogr2/auto-orient/strip)

> 为了开发方便，我们可以使用生成的通用静态库，但是上线的时候只导入真机的，这样工程的体积也会小一些。

### 使用.a静态库
新建一个工程，把我们的静态库拖进去，导入头文件。


![图10.png](http://upload-images.jianshu.io/upload_images/1009061-03def0eb616a8c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### .framework静态库
1.Xcode创建一个新工程，需要选择这个`Cocoa Touch Framework`这个模板
- 创建完成后的`framework.h`和`Infn.plist`不要删除（创建framework的时候命名不要用framework命名，否则在使用这个framework的时候会报错）


![图11.png](http://upload-images.jianshu.io/upload_images/1009061-c8d65cdaa63c1ac1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.创建Person，也是输入名字和年龄，和.a一样


![图12.png](http://upload-images.jianshu.io/upload_images/1009061-ac79fc5601f1905f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图13.png](http://upload-images.jianshu.io/upload_images/1009061-43bade3196b54e40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 注意事项:
- 如果打包的文件中有设置图片的地方，如果还是通过`[UIImage imageNamed:]`的方式设置，图片可能不会显示。
- 图片最好单独打包一个`bundle`，这个时候设置图片的方式为：

```
UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
//拿到路径
NSString *path = [[NSBundle mainBundle] pathForResource:@"bundle" ofType:@"bundle"];
//设置图片
UIImage *image = [UIImage imageNamed:@"delete" inBundle:[NSBundle bundleWithPath:path] compatibleWithTraitCollection:nil];
            
 [button setImage:image forState:UIControlStateNormal];
```

3.选择要公开的头文件
这里主要是让使用者知道有哪些方法和头文件可以使用

- 第一种添加头文件的方式，把需要公开的头文件添加到public里面

![图14.png](http://upload-images.jianshu.io/upload_images/1009061-4301ebd9932ea781.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 第二种添加头文件的方式。在右侧的选择中，选择Public

![第二种方式添加头文件.gif](http://upload-images.jianshu.io/upload_images/1009061-5d22ba135ad6faf0.gif?imageMogr2/auto-orient/strip)

注意。要在这个文件中引入需要公开的头文件


![图15.png](http://upload-images.jianshu.io/upload_images/1009061-e61cdadb40ef4098.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 这里有一个注意点，公开的头文件中，#import的其他类也要公开，添加到public中，如果不想公开，就在头文件用用@class的方式，在对应的.m中用#import方式

4.修改配置

- `Build Active Architecture Only`修改为NO，否则生成的静态库就只支持当前选择设备的架构。
![图16.png](http://upload-images.jianshu.io/upload_images/1009061-16ae567efdbfc649.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



- `Mach-O`格式，因为动态库也可以是以framework形式存在，所以需要设置，否则默认打出来的是动态库。
静态库`Static Library`（默认为`Dynamic Library`）


![图17.png](http://upload-images.jianshu.io/upload_images/1009061-0213e841d46711ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 支持版本


![图18.png](http://upload-images.jianshu.io/upload_images/1009061-ba812545c4b3cb0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5.编译
选择`Generic iOS Device`和`任意一个模拟器`各编译一次，完成后，在工程的`Products`文件夹下的.a文件从红色编程了黑色。编译成功，然后`Show in Finder`。


![生成framework.gif](http://upload-images.jianshu.io/upload_images/1009061-40fec589ccfb4fae.gif?imageMogr2/auto-orient/strip)

可以看到真机与模拟器都生成了.framework。里面都有我们选择公开的头文件
这个时候，可以用命令`lipo -info framework静态库文件下二进制文件的名字`来看下，支持的iOS的CPU框架


![支持框架.gif](http://upload-images.jianshu.io/upload_images/1009061-8bc9e9cf1e2109c8.gif?imageMogr2/auto-orient/strip)

- `Debug-iphoneos`里面支持armv7、arm64,属于真机，用到模拟器就会报错
- `Debug-iphonesimulator`里面支持i386、x86_64，属于模拟器，用到真机会报错

6.合并静态库-让模拟器和真机使用一个静态库
命令如下：
`lipo -create第一个framework文件下二进制文件的绝对路径 第二个framework文件下二进制文件的绝对路径 -output 最终的二进制文件路径`



![新的framework.gif](http://upload-images.jianshu.io/upload_images/1009061-a7e85cf132286796.gif?imageMogr2/auto-orient/strip)

将合成的二进制文件拖进任何一个framework，替换掉原来的，然后把这个新的framework拖进项目就可以使用了



### 使用framework静态库
新建一个工程，把我们的静态库拖进去，导入头文件。然后调用Person中的方法。

![图19.png](http://upload-images.jianshu.io/upload_images/1009061-18c6eee32ac18338.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 如果静态库中有Category类，就要在使用静态库项目的配置中添加`-ObjC`或者`-all_load`
> 如果创建的framework类中使用了.tbd,则项目中也要导入.tbd动态库

注释：部分名词摘自网络。虽然说，学到的都是我们的，但是也不要忘记他人。

> 如果有错误的地方欢迎指出，一起学习，一起进步。
> 如果有不懂的地方，请留言，或者加QQ810646506
>如果喜欢给个赞哈


