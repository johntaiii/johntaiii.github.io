---

title: Universal Links

description: 在iOS9之前,对于从各种从Safari、UIWebView或者WKWebView中唤醒App的需求, 我们只能使用Scheme，但是这种方式有个缺点...

header: Universal Links

---

### 一、什么是Universal Links

在iOS9之前,对于从各种从Safari、UIWebView或者WKWebView中唤醒App的需求, 我们只能使用Scheme，但是这种方式有个缺点, 就是需要提前判断系统中是否安装了能够响应此Scheme的App, 而且很多网页中的判断是有问题的, 可能会出现空白页的情况。另外,微信机（wu）智（chi）地禁用了该功能。这意味着从微信中没法直接调起我们的App。

Universal Links(通用链接)是iOS9推出的一项新功能,当用户在iOS系统中任何地方：备忘录、浏览器或者其他App中的webview点击某个普通的Http/Https链接,（例如 http://163.com/63625.html）, 如果用户已经安装了我们的App, 且App已经支持Universal Links, 那么此时会调起我们的App并打开相应的界面。如果没有安装, 则直接打开原来的H5页面, 这是一个用户无感知的过程。


-------

### 二、听上去略神奇的技术是如何实现的

iOS系统会在安装某个App时，从该App提供的Associated Domains下载一个json格式的文件（由App开发者提前上传到服务器上），这个文件包含一些Path，Associated Domains拼上这些Path就组成了该App相关的Universal Liks，并存储在iOS系统中。这样，在用户点击系统中无论某个App中的链接时，若与已存储的某个Universal Links匹配，就触发了该功能。

-----


### 三、有哪些明明白白的优点

* **唯一性:** 不像自定义的Scheme, 因为Universal Links使用标准的Http/Https, 所以不能被其它的App声明。

* **安全:** 当用户的手机上安装了我们的App,那么iOS系统会去我们的网站下载一个说明文件(这个说明文件声明了我们App可以打开哪些类型的Http链接).因为只有我们自己才能上传文件到网站上,所以网站和App之间的关联是安全的。

* **灵活:** 当用户手机上没有安装App的时候,Universal Links也不会失效。用户点击链接,会在Safari中展示网页的内容。

* **简单:** 一个URL链接,可以同时对H5和App都起作用。

* **私密** 其它App可以在不需要知道我们的App是否安装的情况下和我们的app相互通信。

-------

### 四、如何集成Universal Links

**1. 创建包含appID和路径的说明文件**

创建一个名为apple-app-site-association的json文件, 格式如下：


```{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "9JA89QQLNQ.com.apple.wwdc",
                "paths": [ "/wwdc/news/", "/videos/wwdc/2015/*"]
            },
            {
                "appID": "ABCD1234.com.apple.wwdc",
                "paths": [ "*" ]
            }
        ]
    }
}
```
appID: 组成方式是 teamId.yourapp's bundle identifier.
paths: 根据 paths 键设定一个app支持的路径列表,只有这些指定的路径的链接,才能被app所处理,举个例子:我们网站地址是www.163.com,如果path写的是"/support/*",那么当用户点击www.163.com/support/myDoucument,就可以进入app了。如果是www.163.com/other 就不会进入. 
 
**2. 上传文件到网站所在服务器**
上传该文件到域名所对应的网站的**.well-known**目录下,这一步是为了iOS系统能从https://163.com/.well-known/apple-app-site-associationxh 获取到上传的apple-app-site-association文件。

**3. 在Xcode中配置App能处理的域名** 
在Xcode工程里打开配置中的Associated Domains，并在Domains中填入需要支持Universal Links的域名 必须以 applinks: 为前缀，比如：

```
applinks: 163.com
applinks: www.163.com 
``` 

**4. 客户端工程里添加进一步处理的方法**
至此在用户点击某个链接后, 可以直接进入我们的App了。另外，我们还需要获取到用户进来的链接, 然后根据链接来打开相应的页面。

在AppDelegate里实现委托方法


```
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray *))restorationHandler
{
    if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        NSURL *webpageURL = userActivity.webpageURL;
        NSString *host = webpageURL.host;
        if ([host isEqualToString:@"163.com"]) {
            //进一步处理
        }
        else {
            [[UIApplication sharedApplication]openURL:webpageURL];
        }

    }
    return YES;

}
```
当 userActivity == NSUserActivityTypeBrowsingWeb 类型时, 表示App由Universal Links调起, 我们可以在此处添加相应的处理逻辑。

**5. 测试是否生效**

* 在微信中输入App能识别的链接，然后直接点击此链接,应该能直接跳转到App

* 长按该链接,在出现的弹出菜单中第二项是“在'XXX'中打开”

-----

## 五、注意事项

1. 服务端需要支持Https，并且不支持任何重定向。
2. paths 可以设置通配符，即只是一个星号，如果想打开APP 而不管路径是什么。
3. paths 路径大小写敏感。
4. 如果用户在Safari中打开我们的网站，此时点击了某个Universal Link，且Universal Link的域名与当前网站的域名一致，iOS系统会认为用户更倾向于在浏览中打开该链接，所以不会调起App。如果二者域名不一致，App则会被调起。


-----

*参考*
[1. Support Universal Links](https://developer.apple.com/library/prerelease/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html)

[2. iOS Universal Links(通用链接)](https://yohunl.com/ios-universal-links-tong-yong-lian-jie/)

[3. iOS 9学习系列：打通 iOS 9 的通用链接（Universal Links）](http://www.cocoachina.com/ios/20150902/13321.html)

[4.测试JSON文件格式是否合法（Apple官方工具）](https://search.developer.apple.com/appsearch-validation-tool/)

