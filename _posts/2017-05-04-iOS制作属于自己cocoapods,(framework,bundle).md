---
title: iOS,制作属于自己cocoapods,(framework,bundle)
date: 2017-05-04 17:06
tags: 
- cocoapods
- framework
---

[TOC]

缘由:
还是那个小活,需求方想用cocoapods来集成framework,由于之前,我从没有自己做过属于自己的cocoapods,所以也是一脸懵逼,各种查资料.

制作cocoapods步骤:

- 代码提交到github平台
- 创建.podspec
- 编辑.podspec
- 项目打tag
- 验证.podspec
- 注册 cocoapods trunk帐号
- 发布.podspec到cocoapods

#### 1.代码提交到github平台
1.在github上创建一个新的仓库<图中的1、2一定要选择，2可以是其他的`License`>
![创建仓库.png](http://upload-images.jianshu.io/upload_images/1009061-32865ff524998870.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.clone代码到本地
![复制地址.png](http://upload-images.jianshu.io/upload_images/1009061-37c072a90d9b0e30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![克隆.png](http://upload-images.jianshu.io/upload_images/1009061-0ad63537bfe1990b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.相加自己的项目，然后提交到github上
​    `git add .`
​    `git commit -m "描述"`
​    `git push origin master`

#### 2.创建.podspec
在项目目录下，执行命令创建.podspec,一下3中方式都可以创建：

- $ pod spec create CFMobAdSDK.podspec
- $ touch CFMobAdSDK.podspec
- $ vim CFMobAdSDK.podspec

#### 3.编辑.podspec
> 记住：不要用文本编辑打开编辑，不要用文本编辑打开编辑，不要用文本编辑打开编辑，
> 可以用[atom](https://www.atom.io)编辑
> 可以用`vim命`令编辑,`vim`状态下,按`i`进入编辑状态,按`esc`退出编辑状态,然后输入`:wq`保存退出编辑

```
:Pod::Spec.new do |s|
    s.name         = "CFMobAdSDK"
    s.version      = "1.0.6"
    s.ios.deployment_target = '7.0'
    s.summary      = "广告sdk,一个简单的广告SDK."
    s.homepage     = "https://github.com/lixianshen/CFMobAdSDK"
    s.license              = { :type => "MIT", :file => "LICENSE" }
    s.author             = { "Simple" => "810646506@qq.com" }
    s.source       = { :git => "https://github.com/lixianshen/CFMobAdSDK", :tag => s.version }
    #s.source_files  = "CFMobAdSDK/*"
    s.resources          = "CFMobAdSDK/CFMobAdSDK.bundle"
    s.frameworks = 'Foundation', 'UIKit', 'CoreLocation', 'AdSupport'
    s.vendored_frameworks = 'CFMobAdSDK.framework'
    s.requires_arc = true
end
```
下面介绍一下.podspec中部分代码的含义:

- s.name：名称，`pod search` 搜索的关键词,一定要和.podspec的名称一样,否则报错
- `s.version`：版本号
- `s.ios.deployment_target`:支持的`pod`最低版本
- `s.summary`: 简介
- `s.homepage`:项目主页地址
- `s.license`:开源协议(创建`github`库的时候选择的)
- `s.author`:作者信息(这里随便谢谢也可以通过)
- `s.social_media_url`:社交网址
- `s.source`:项目的地址
- `s.source_files`:需要包含的源文件
- `s.resource`:资源文件,单个
- `s.resources`: 资源文件(含`bundle`)
- `s.requires_arc`: 是否支持`ARC`
- `s.dependency`：依赖库，不能依赖未发布的库.如AFNetWorking
- `s.vendored_frameworks`:包含的`framework`,也就是我们自己制作的pod
- `s.description`:描述,字数要比`s.summary`长
- `s.screenshots`:截图
- `s.exclude_files`:隐藏的文件
- `s.public_header_files`:公开的头文件
- `s.framework`:所需的`framework`,单个
- `s.frameworks`:所需的`framework`,多个用逗号隔开
- s.vendored_libraries:包含的的.a
注意事项:

- 多个s.dependency可以这样写,(可以加上版本号):
 s.dependency  = 'AFNetworking', '~> 2.3'
 s.dependency  = 'SDWebImage'
 s.dependency  = 'AFNetworking'

- s.license可以用下面2中写法:
 s.license = "MIT" 会有一个警告
 s.license = { :type => "MIT", :file => "LICENSE" }

- s.source_files:写法及含义:
```
"CFMobAdSDK/*
"CFMobAdSDK/*.{h,m}"
"CFMobAdSDK/**/*.h"
```
>“*” 表示匹配所有文件
>“*.{h,m}” 表示匹配所有以.h和.m结尾的文件
>“**” 表示匹配所有子目录

- s.source 常见写法

    ```
    /// git commit -m =>"13287dd",讲pod版本与git仓库中的某一次提交绑定
    s.source = { :git => "https://github.com/lixianshen/CFMobAdSDK", :commit => "13287dd" }
    
    /// 将这个Pod版本与Git仓库中某个版本的comit绑定 
    s.source = { :git => "https://github.com/lixianshen/CFMobAdSDK", :tag => 1.0.0 }
    
    /// 将这个Pod版本与Git仓库中相同版本的comit绑定
    s.source = { :git => "https://github.com/lixianshen/CFMobAdSDK", :tag => s.version }
    ```

### 4.上传编辑好的.podspec
### 5.tag标记,并且上传
    ```
    /// 第一次需要在前面加一个v
    git tag "v1.0.0"
    git push --tags
    ```
### 6.验证.podspec
方式一

    // 加上--verbose验证失败会显示详细的报错信息
    pod spec lint CFMobAdSDK.podspec --verbose

方式二
​    

    pod spec lint

 验证开始
​    
    -> CFMobAdSDK

成功:
​     ![验证成功.png](http://upload-images.jianshu.io/upload_images/1009061-860a7b06e425f6a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
​    
 验证失败的一些情况:
​    1.下面这种情况,只要在终端运行命令:`echo "2.3" > .swift-version`
​    ![echo "2.3" > .swift-version.png](http://upload-images.jianshu.io/upload_images/1009061-358864d7811e561b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
​    
   2.- ERROR | [iOS] file patterns: The `source_files` pattern did not match any file.
​    解决方法: 文件路径不对, 也就是设置 s.source_files 字段时, 发生了错误
​    3.- ERROR | [iOS] file patterns: The `vendored_frameworks` pattern did not match any file.
​    解决方法: framework路径不对, 也就是设置 s.vendored_frameworks 字段时, 发生了错误
​    
### 7.注册cocoapods trunk

- trunk需要`cocoaPods`的版本在0.33版本，用`pod --version`,如果版本低,先升级`sudo gen install cocoapods`和`pod setup`

- 注册的三种方式:
  
  - pod trunk register eloy@example.com `Eloy Durán` --description=`Personal Laptop`
  - pod trunk register eloy@example.com --description=`Work Laptop`
  - pod trunk register eloy@example.com
  
 > 这个时候,你填写的邮箱会收到一封邮件,把链接复制,在浏览器打开就可以了,如果没有打开这个链接,下面的步骤是不能进行的

- 查看注册信息:
```
pod trunk me
```
```
  - Name:     lixianshen
  - Email:    810646506@qq.com
  - Since:    May 1st, 01:51
  - Pods:
  - CFMobAdSDK
  - Sessions:
  - May 1st, 01:51 - September 7th, 08:30. IP: 125.121.226.128 Description:
    Simple
  - May 2nd, 18:35 - September 7th, 18:43. IP: 125.118.107.149
  - May 2nd, 20:55 - September 7th, 21:05. IP: 125.118.107.149
  - May 4th, 02:19 - September 9th, 02:20. IP: 125.118.107.149
```

### 8.发布自己的.podspec到cocoapods

- pod trunk push CFMobAdSDK.podspec 
-  如果有警告用:pod trunk push CFMobAdSDK.podspec --allow-warnings

    1.先验证是否正确
    
    ![验证是否正确.png](http://upload-images.jianshu.io/upload_images/1009061-535b6d1d59eafaf2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    2.发布成功
    ![发布成功.png](http://upload-images.jianshu.io/upload_images/1009061-7bbeea4c12c01603.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    3.检查自己发布的pod
    > 检查之前先删除缓存中的json文本 `~/Library/Caches/CocoaPods/search_index.json`
    > ![5581BF55-1A5F-41BA-88C7-34F90B0FA421.png](http://upload-images.jianshu.io/upload_images/1009061-e1caeb7c9aeb78ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    > 如果不删除,会搜索不到自己发布的,如果让你朋友也要查到也要删除现有的缓存
    
    4.搜索记录
    ![搜索记录.png](http://upload-images.jianshu.io/upload_images/1009061-6ec741b102cec6bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
    如果想删除已发的版本,需要使用下面的命令:
    ```
    pod trunk delete CFMobAdSDK 版本号
    ```
    例如
    ```
    pod trunk delete CFMobAdSDK 1.0.1
    ```
    到这基本结束了:下面是几个和本内容相关的比较好的链接:
    
    - trunk 命令详解:
    [trunk](https://guides.cocoapods.org/terminal/commands.html#group_trunk)
    
    - 制作cocoapods的官方网站
    [Making Cocoapods](https://guides.cocoapods.org/making/index.html)
    
    - Framework和.a的制作
    [Framework+a](http://www.jianshu.com/p/cb17d6bae5a0)
    
### 结束语:
如果发现问题,或者有不懂的地方,请留言


