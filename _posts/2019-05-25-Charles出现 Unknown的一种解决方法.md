---
title: Charles出现Unknown的一种解决方法
date: 2019-05-25 10:25
tags: 
- 开发笔记
- iOS逆向
---



文采不好，凑活着看吧!!
文采不好，凑活着看吧!!

最近在逆向分析一个 `App`，遇到了 `unknown` 的现象，按照常规的处理就是安装证书，然后信任证书，打开 `App`，调试。然后发现。还是 `unknown`，如下图：

![Charles-1](http://blog.objccf.com/blog/2019-05-24-020040.png)

如上图提示 `SSL Proxying not enabled for this host: enable in Proxy Settings, SSL locations`，其实已经设置了，唯一不同的是 `SSL Proxying` 的端口配置不对，这里是 `8663`，而默认配置是 `443`，这里添加配置，如下图：

![image-20190524102447436](http://blog.objccf.com/blog/2019-05-24-022447.png)

刷新请求，出现如下图所示：

![image-20190524102724665](http://blog.objccf.com/blog/2019-05-24-022725.png)

基本上常规的操作，都操作了，由于我用的设备是非越狱机，不能安装 `ssl kill switch2`，所以要从代码上进行处理。这里猜测是开启了证书验证，去安装包里看看有没有 `cer` 的证书。

![image-20190524103127643](http://blog.objccf.com/blog/2019-05-24-023128.png)

这里猜测，项目里做了 `SSL  Pinning` 处理。下面去分析代码，由于被分析项目网络请求使用的是 `AFNetworking`，`AFNetworking` 中针对 `SSL` 绑定实现安全链接的是 `AFSecurityPolicy` 这个类负责的，是 `AFNetworking` 中网络通信安全策略模块。它提供三种 `SSL Pinning Mode` 基本上的处理如下：

```
/**
 ## SSL Pinning Modes

 The following constants are provided by `AFSSLPinningMode` as possible SSL pinning modes.

 enum {
 AFSSLPinningModeNone,
 AFSSLPinningModePublicKey,
 AFSSLPinningModeCertificate,
 }

 `AFSSLPinningModeNone`
 Do not used pinned certificates to validate servers.

 `AFSSLPinningModePublicKey`
 Validate host certificates against public keys of pinned certificates.

 `AFSSLPinningModeCertificate`
 Validate host certificates against pinned certificates.
*/
```

基本使用如下：

```
+ (AFHTTPSessionManager *)manager
{
    static AFHTTPSessionManager *manager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
    
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        manager =  [[AFHTTPSessionManager alloc] initWithSessionConfiguration:config];

        AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModePublicKey withPinnedCertificates:[AFSecurityPolicy certificatesInBundle:[NSBundle mainBundle]]];
        manager.securityPolicy = securityPolicy;
    });
    return manager;
}
```

下断点：`+[AFSecurityPolicy certificatesInBundle:]`，

![image-20190524110733468](http://blog.objccf.com/blog/2019-05-24-030734.png)

然后对 `+[AFSecurityPolicy policyWithPinningMode:]` 下断点，看看是否使用了 `SSL  Pinning`。

![image-20190524111132630](http://blog.objccf.com/blog/2019-05-24-031133.png)

这里可以得出，确实使用了 `SSL  Pinning`。这里的 `Mode` 是 `AFSSLPinningModePublicKey`(只比对服务器证书和本地证书的 `Public Key` 是否一致，如果一致则信任服务器证书)，下面把这个值改成 `0`(``AFSSLPinningModeNone`),然后在请求，结果如下：

![image-20190524111716433](http://blog.objccf.com/blog/2019-05-24-031716.png)



文采不好，凑活着看吧

