title: CefSharp浅尝辄止
author: Salamander
tags:
  - 'C#'
  - CefSharp
  - WPF
categories:
  - 'C#'
date: 2019-11-16 20:00:00
---
![docker logo](/images/CefSharp-logo.png)

## CefSharp
[CEF](https://github.com/chromiumembedded/cef)全称：**Chromium Embedded Framework**。  
CefSharp是什么？[官网](http://cefsharp.github.io/)上它是这么写的：CefSharp是在C#或VB.NET应用程序中嵌入全功能标准兼容web浏览器的最简单方法。CefSharp有WinForms和WPF应用程序的浏览器控件，也有自动化项目的无标题（屏幕外）版本。CefSharp基于Chromium嵌入式框架，这是Google Chrome的开源版本。  
说白了，就是基于C#或VB语言的**可编程浏览器**（当然CEF也有其他语言的，如[Java](https://bitbucket.org/chromiumembedded/java-cef)，[Go](https://github.com/cztomczak/cef2go)）。

<!-- more -->

本文环境：
* CefSharp版本：75.1.143
* VS版本：2015
* 操作系统：Windows 10专业版

## WPF引入CefSharp
CefSharp有现成的NuGet包，先引入到项目中，然后在XAML中添加响应控件：
```
<cefSharp:ChromiumWebBrowser Name="myChrome" Loaded="myChrome_Loaded"/>
```
添加`cefSharp`命名空间：
```
xmlns:cefSharp="clr-namespace:CefSharp.Wpf;assembly=CefSharp.Wpf"
```
在`myChrome_Loaded`事件中，我们让浏览器打开百度首页：
```C#
private void myChrome_Loaded(object sender, RoutedEventArgs e)
{
    String url = "https://www.baidu.com";
    myChrome.Load(url);
}
```
运行程序，我们就可以看到百度首页了。


## 截断请求
根据[文档](http://cefsharp.github.io/api/75.1.x/html/T_CefSharp_Handler_RequestHandler.htm)，我们可以看到`RequestHandler`接口中的方法`GetResourceRequestHandler`会在每次发请求前被调用：
> GetResourceRequestHandler  
> Called on the CEF IO thread before a resource request is initiated.

所以我们先需要一个实现`IRequestHandler`接口的类
```

class CustomRequestHandler : IRequestHandler
{
    public bool GetAuthCredentials(IWebBrowser chromiumWebBrowser, IBrowser browser, string originUrl, bool isProxy, string host, int port, string realm, string scheme, IAuthCallback callback)
    {
        return false;
    }

    public IResourceRequestHandler GetResourceRequestHandler(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, bool isNavigation, bool isDownload, string requestInitiator, ref bool disableDefaultHandling)
    {
        return new CustomResourceRequestHandler();
    }

    public bool OnBeforeBrowse(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, bool userGesture, bool isRedirect)
    {
        return false;
    }

    public bool OnCertificateError(IWebBrowser chromiumWebBrowser, IBrowser browser, CefErrorCode errorCode, string requestUrl, ISslInfo sslInfo, IRequestCallback callback)
    {
        return false;
    }

    public bool OnOpenUrlFromTab(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, string targetUrl, WindowOpenDisposition targetDisposition, bool userGesture)
    {
        return false;
    }

    public void OnPluginCrashed(IWebBrowser chromiumWebBrowser, IBrowser browser, string pluginPath)
    {
    }

    public bool OnQuotaRequest(IWebBrowser chromiumWebBrowser, IBrowser browser, string originUrl, long newSize, IRequestCallback callback)
    {
        return false;
    }

    public void OnRenderProcessTerminated(IWebBrowser chromiumWebBrowser, IBrowser browser, CefTerminationStatus status)
    {
    }

    public void OnRenderViewReady(IWebBrowser chromiumWebBrowser, IBrowser browser)
    {
    }

    public bool OnSelectClientCertificate(IWebBrowser chromiumWebBrowser, IBrowser browser, bool isProxy, string host, int port, X509Certificate2Collection certificates, ISelectClientCertificateCallback callback)
    {
        return false;
    }
}
```
`GetResourceRequestHandler`是我们要重点关注的方法，里头我们返回了一个类实例，在这个类中我们就可以**自定义请求**。  
新版的CefSharp（75版本之后）把`OnBeforeResourceLoad`方法移动到了`ResourceRequestHandler`接口里，所以我们还需要一个实现`ResourceRequestHandler`的类（也就是上面代码中的`CustomResourceRequestHandler`类）：
```
public class CustomResourceRequestHandler : ResourceRequestHandler
{
    protected override CefReturnValue OnBeforeResourceLoad(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IRequestCallback callback)
    {
        var headers = request.Headers;
        headers["Custom-Header"] = "My Custom Header";
        request.Headers = headers;

        return CefReturnValue.Continue;
    }
}
```
最后，把自定义请求类设置到CefSharp实例中
```
myChrome.RequestHandler = new CustomRequestHandler();
```
通过Fiddler这样的抓包工具，我们就会发现，自定义的`Custom-Header`头已经加上了

![detail](https://s2.ax1x.com/2019/11/16/MBE5Dg.png)

### 添加自定义查询参数
上面的例子中，我们添加了自定义的header，如果我们想改写`URL`添加一些自定义的查询参数呢，譬如`name=foo`？这里有个坑，如果我们简单地把`request.Url += "?name=foo"`，这样会导致无限重定向（因为改了Url就会重定向）。解决方法也很简单，就是判断一下我们想要的查询参数是否已经在`Url`里了：
```
protected override CefReturnValue OnBeforeResourceLoad(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IRequestCallback callback)
{
    var headers = request.Headers;
    headers["Custom-Header"] = "My Custom Header";
    request.Headers = headers;

    if (!request.Url.Contains("name=foo"))
    {
        request.Url += "?" + "name=foo";
    }

    return CefReturnValue.Continue;
}
```

### 添加自定义Body
未完待续。。。









参考：
* [StackOverflow](https://stackoverflow.com/questions/31250797/chromium-send-custom-header-info-on-initial-page-load-c-sharp)