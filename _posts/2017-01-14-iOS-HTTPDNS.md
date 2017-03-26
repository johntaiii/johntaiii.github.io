---
title: iOS App HTTPDns直连

description: 一直有用户反映，不管通过通过手机端、还是PC端访问我们的产品都会不定时出现域名劫持的问题。为了解决这个问题，我们只能绕过传统的运营商域名解析，通过IP直接访问服务。本文对App中集成HttpDNS作简要介绍。

header: iOS App HTTPDns直连
---

一直有用户反映，不管通过通过手机端、还是PC端访问我们的产品都会不定时出现域名劫持的问题。为了解决这个问题，我们只能绕过传统的运营商域名解析，通过IP直接访问服务。本文对App中集成HttpDNS作简要介绍。

####  一、HTTPDNS介绍：

httpDNS是阿里提供的面向移动端的域名解析产品，提供了面向移动端的SDK，客户端可以通过传入域名的方式调用，SDK会直接返回解析出的IP地址。

    NSString *ip = [[HttpDnsService sharedInstance] getIpByHostAsync:Domain];
    

#### 二、需求描述：

对项目中: 1.**原生图片的请求**, 2.**H5网页中请求**，分别做IP直连处理。


#### 三、实现方案：


*  实现原理：

通过注册NSURLProtocol，拦截所有请求，过滤出相应的图片请求及H5网页请求，将请求的url中的域名替换为IP后，重新发起请求，获取到响应数据后，回调给URL Loading System。

* 实现过程：


##### 1.拦截请求


由于原生图片和H5网页中的请求需要分开处理以便于实现通过降级开关分别控制，所以注册了两个NSURLProtocol分别处理这两项业务，具体策略为：
YH_ImageProtocol在拦截到请求后，按照URL后缀（是否包含：.jpg/.jpeg/.png/.gif）过滤出图片的URL。
YH_WebProtocol在拦截到请求后，排除图片的URL，则认为是需要拦截的请求。


##### 2. 手动发起请求

拦截到请求后，需要根据协议分别做处理：如果是HTTP请求，使用`NSURLSession`重新发起请求，获取到响应的数据后，回调给`URL Loaidng System`。如果是`HTTPS`请求，由于当前请求URL的域名被替换成了IP地址，请求URL中的host也会被替换成HTTPDNS解析出来的IP，导致服务器获取到的域名为解析后的IP，无法找到匹配的证书，只能返回默认的证书或者不返回，所以会出现SSL/TLS握手不成功的错误。为了解决这个问题，我们需要hook HTTPS访问前SSL连接过程，根据网络请求头部域中的HOST信息，设置SSL连接PeerHost的值，之后根据服务器返回的证书执行验证过程。所以在拦截网络请求后，使用`CFHTTPMessageRef`创建`NSInputStream`实例进行Socket通信，并设置其`kCFStreamSSLPeerName`的值：


```
// 创建CFHTTPMessage对象的输入流
CFReadStreamRef readStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault,cfrequest);
inputStream = (__bridge_transfer NSInputStream *) readStream;
    
// 设置SNI host信息
NSString *host = [curRequest.allHTTPHeaderFields objectForKey:@"host"];
    if (!host) {
        host = curRequest.URL.host;
    }
    [inputStream setProperty:NSStreamSocketSecurityLevelNegotiatedSSL forKey:NSStreamSocketSecurityLevelKey];
    NSDictionary *sslProperties = [[NSDictionary alloc] initWithObjectsAndKeys:
                                   host, (__bridge id) kCFStreamSSLPeerName,
                                   nil];
    [inputStream setProperty:sslProperties forKey:(__bridge_transfer NSString *) kCFStreamPropertySSLSettings];
    [inputStream setDelegate:self];
```

##### 3.重定向

当返回的StatusCode在300、400之间，且header中location字段中取出合法的URL时，用该URL初始化新的请求，在protocol内部重新执行一遍之前的流程。

#### 四、碰到的问题及解决方法：

*  **GZIP**

之前在测试过程中发现，用Webview加载官网时，页面显示乱码。经排查，确认是返回的`content-type`为gzip，因为未解压导致页面无法识别。为此，我们在收到响应后，先判断content类型，如果为gzip，先进行解压再回调给相应的client。


*  **CSS文件中通过相对路径的方式引用的静态资源无法加载**

该问题发生的具体原因是：在WebView中的请求被拦截，域名改为IP直连后，CSS文件中通过相对路径引用的静态资源（包括iconfont和少量图片）的url直接沿用了CSS文件URL中的IP地址作为域名，跳过了域名解析的步骤，且header中的HOST字段未设置为相应的域名。最终导致无法通过SNI扩展的方式获取到SSL证书，建连失败。我们的解决方案是保存好IP地址和域名的映射关系，碰到前述问题时，能够获取到IP地址对应的域名，设置给HOST，以保证SSL握手成功。

####  五、待改进的地方：


目前的业务需求是，拦截到的H5请求，全部强制转为HTTPS方式请求。这种情况下会导致一些服务端不支持HTTPS的请求失败，尤其跳转到一些第三方网站的页面。为避免该问题，我们应该提供一种容错机制，当强制使用HTTPS的方式去打开页面时，如果SSL握手失败，可以再改为HTTP的方式去请求。

