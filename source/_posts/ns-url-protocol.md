---
title: URLProtocol导致上传进度回调丢失排查过程
date: 2020-01-02 13:04:34
tags:
- iOS
- NSURLProtol
- upload
- network
categories: 
- iOS
---

## 问题发现

最近开发时候发现了一个奇怪的问题，在使用iOS的`NSURLSessionManager`进行文件上传时候，回调方法

```objectivec
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;
```

无论如何也不会被调用。

<!-- more -->

## 问题排查

尝试了各种`[NSURLSession uploadTaskWithRequest]`的写法都已失败告终。

百思不得其解，为了确定是不是`NSURLSession`自身的bug自己写了一个Demo App尝试了一下，竟然可以正常回调。

于是感觉可能是在某处HOOK了`NSURLSession`的API导致的问题，于是赶紧在工程里面搜索一下`NSURLSeesion`的各个类相关的方法，还是没有找到。

就在问题即将要变得难搞的时候，偶然在控制台的日志里面看到有输出我发送的请求的信息，立刻就意识到可能是某个`URLProtocol`拦截了我发送的请求，于是用`URLProtocol`做关键字在工程里面搜，果然，公司的网络性能监控框架通过注册了自己的`URLProtocol`来拦截请求信息，记录请求的性能问题。

于是先注释掉监控用的`URLProtocol`注册的代码，再尝试一下，果然能正常收到回调了。看来问题就是出现在这里。

翻看了一下监控`URLProtocol`的代码，看起来用的也没有问题。于是只好去搜索下看看有没有其他的解决方案。

## 解决问题

找到了苹果的一个[示例说明文档](https://github.com/robovm/apple-ios-samples/blob/master/CustomHTTPProtocol/Read%20Me%20About%20CustomHTTPProtocol.txt)

里面有一段：

> Similarly, there is no way for your NSURLProtocol subclass to call the NSURLConnection delegate's -connection:needNewBodyStream: or -connection:didSendBodyData:totalBytesWritten:totalBytesExpectedToWrite: methods ([rdar://problem/9226155]() and [rdar://problem/9226157]()).  The latter is not a serious concern--it just means that your clients don't get upload progress--but the former is a real issue.  If you're in a situation where you might need a second copy of a request body, you will need your own logic to make that copy, including the case where the body is a stream.


大意是说我们自己实例化的`NSURLProtocol`子类没有办法调用`client`的`-connection:didSendBodyData:totalBytesWritten:totalBytesExpectedToWrite:`方法，所以没有办法把进度通知到`client`，也就不会调用上面的`- (void)URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:`方法。
要解决这个问题就要自己去copy请求的`body`。

性能监控的`NSURLProtocol`的逻辑暂时没有办法更改，那没有办法，只有在上传的时候禁止监控了。

性能监控的`PerformanceURLProtocol`的`canInitWithRequest`里面有这么一段代码：

```objectivec
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    if ([NSURLProtocol propertyForKey:@"PerformanceHTTPHandledIdentifier" inRequest:request] ) {
        return NO;
    }
}
```

根据这段代码，在上传文件之前先调用一下

```objectivec
Class HMDURLProtocol = NSClassFromString(@"PerformanceURLProtocol");
if ([PerformanceURLProtocol isSubclassOfClass:NSURLProtocol.class]) {
    [PerformanceURLProtocol setProperty:@YES forKey:@"PerformanceHTTPHandledIdentifier" inRequest:request];
}
```

防止进入`PerformanceURLProtocol`的拦截逻辑。

重新运行代码， 一切正常。
