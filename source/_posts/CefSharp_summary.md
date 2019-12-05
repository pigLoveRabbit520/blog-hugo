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
根据[文档](http://cefsharp.github.io/api/75.1.x/html/T_CefSharp_Handler_RequestHandler.htm)，我们可以看到`RequestHandler`类中的方法`GetResourceRequestHandler`会在每次发请求前被调用：
> GetResourceRequestHandler  
> Called on the CEF IO thread before a resource request is initiated.

`RequestHandler`类是[`IRequestHandler`](http://cefsharp.github.io/api/75.1.x/html/T_CefSharp_IRequestHandler.htm)接口的默认实现，我们自定义请求可以继承这个类：
> Default implementation of IRequestHandler. 
> This class provides default implementations of the methods from IRequestHandler, therefore providing a convenience base class for any custom request handler.


所以我们可以创建一个继承`RequestHandler`的类
```

class CustomRequestHandler : RequestHandler
{
    protected override IResourceRequestHandler GetResourceRequestHandler(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, bool isNavigation, bool isDownload, string requestInitiator, ref bool disableDefaultHandling)
    {
        return new CustomResourceRequestHandler();
    }
}
```

`GetResourceRequestHandler`是我们要重点关注的方法，里头我们返回了一个类实例，在这个类中我们就可以**自定义请求**。  
新版的CefSharp（75版本之后）把`OnBeforeResourceLoad`方法移动到了`IResourceRequestHandler`接口里（[文档](http://cefsharp.github.io/api/75.1.x/html/T_CefSharp_IResourceRequestHandler.htm)），同样的CefSharp也提供了这个接口的默认实现：`ResourceRequestHandler`，所以我们还需要一个继承`ResourceRequestHandler`的类（也就是上面代码中的`CustomResourceRequestHandler`类）：
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

根据[IRequest](http://cefsharp.github.io/api/75.1.x/html/T_CefSharp_IRequest.htm)的文档，我们可以利用`PostData`属性：
```
protected override CefReturnValue OnBeforeResourceLoad(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IRequestCallback callback)
{
    var headers = request.Headers;
    headers["Custom-Header"] = "My Custom Header";
    request.Headers = headers;

    string body = "name=foo";
    byte[] byteArray = System.Text.Encoding.UTF8.GetBytes(body);

    request.InitializePostData();
    var element = request.PostData.CreatePostDataElement();
    element.Bytes = byteArray;
    request.PostData.AddElement(element);

    return CefReturnValue.Continue;
}
```
通过Fiddler这样的抓包工具，我们就会发现，POST 数据已经加上了：

![detail](https://s2.ax1x.com/2019/11/30/QVjqp9.png)

## 加载本地HTML字符串

有时候，我们可能需要渲染一个内存中的HTML字符串，CefSharp也提供这样的接口，代码很简单：
```
private void myChrome_Loaded(object sender, RoutedEventArgs e)
{
    string html = @"<!DOCTYPE html>
<html>
    <head>
        <title>这是个标题</title>
        <meta charset='utf-8' />
        <meta name = 'viewport' content = 'width=device-width, initial-scale=1' />
     </head>
    <body>
        <h1>这是一个一个简单的HTML</h1>
        <p>Hello World！</p >
    </body>
</html>";
    String url = "https://www.baidu.com";
    myChrome.LoadHtml(html, url);
}

```

## 截断响应
这里的关键在于`GetResourceResponseFilter`方法，它的签名如下：
```
IResponseFilter GetResourceResponseFilter(
	IWebBrowser chromiumWebBrowser,
	IBrowser browser,
	IFrame frame,
	IRequest request,
	IResponse response
)
```
它返回了一个`IResponseFilter`接口，在这个接口中，我们可以截取到请求响应的内容。在CefSharp最新版本中，`GetResourceResponseFilter`已经被放入到`IResourceRequestHandler`接口中，[最新文档](http://cefsharp.github.io/api/75.1.x/html/M_CefSharp_IResourceRequestHandler_GetResourceResponseFilter.htm)。  
下面我放了一个截断网页XHR请求的例子：
```
public class TestJsonFilter : IResponseFilter
{
    public List<byte> DataAll = new List<byte>();

    public FilterStatus Filter(System.IO.Stream dataIn, out long dataInRead, System.IO.Stream dataOut, out long dataOutWritten)
    {
        try
        {
            if (dataIn == null || dataIn.Length == 0)
            {
                dataInRead = 0;
                dataOutWritten = 0;

                return FilterStatus.Done;
            }

            dataInRead = dataIn.Length;
            dataOutWritten = Math.Min(dataInRead, dataOut.Length);

            dataIn.CopyTo(dataOut);
            dataIn.Seek(0, SeekOrigin.Begin);
            byte[] bs = new byte[dataIn.Length];
            dataIn.Read(bs, 0, bs.Length);
            DataAll.AddRange(bs);

            dataInRead = dataIn.Length;
            dataOutWritten = dataIn.Length;

            return FilterStatus.NeedMoreData;
        }
        catch (Exception ex)
        {
            dataInRead = dataIn.Length;
            dataOutWritten = dataIn.Length;

            return FilterStatus.Done;
        }
    }

    public bool InitFilter()
    {
        return true;
    }

    public void Dispose()
    {

    }
}

public class FilterManager
{
    private static Dictionary<string, IResponseFilter> dataList = new Dictionary<string, IResponseFilter>();

    public static IResponseFilter CreateFilter(string guid)
    {
        lock (dataList)
        {
            var filter = new TestJsonFilter();
            dataList.Add(guid, filter);

            return filter;
        }
    }

    public static IResponseFilter GetFileter(string guid)
    {
        lock (dataList)
        {
        
            if (dataList.ContainsKey(guid))  // 这里要检测key存在，不然会报异常，会导致ContextSwitchDeadlock
            {
                return dataList[guid];
            }
            else
            {
                return null;
            }
        }
    }
}

public class CustomResourceRequestHandler : ResourceRequestHandler
{
    protected override CefReturnValue OnBeforeResourceLoad(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IRequestCallback callback)
    {
        // 截断请求的代码...
        return CefReturnValue.Continue;
    }


    protected override IResponseFilter GetResourceResponseFilter(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IResponse response)
    {
        if (!(request.ResourceType == ResourceType.Xhr))  // 不是XHR类型就不去过滤
        {
            return null;
        }
        var filer = FilterManager.CreateFilter(request.Identifier.ToString());
        return filer;
    }

    protected override void OnResourceLoadComplete(IWebBrowser chromiumWebBrowser, IBrowser browser, IFrame frame, IRequest request, IResponse response, UrlRequestStatus status, long receivedContentLength)
    {
        var filer = FilterManager.GetFileter(request.Identifier.ToString()) as TestJsonFilter;
        if (filer != null)
        {
            Console.WriteLine(ASCIIEncoding.UTF8.GetString(filer.DataAll.ToArray()));  // 打印body内容
        }
    }
}


private void myChrome_Loaded(object sender, RoutedEventArgs e)
{
    String url = "https://github.com/salamander-mh";  // github首页上有ajax请求，可以看效果
    myChrome.Load(url);
}

```
运行程序，在`输出`视图就可以看到Ajax请求的body数据。

## 截取cookie
建立Cookie读取对象，继承接口 ICookieVisitor
```
public class CookieVisitor : CefSharp.ICookieVisitor
{
    public event Action<CefSharp.Cookie> SendCookie;


    public bool Visit(Cookie cookie, int count, int total, ref bool deleteCookie)
    {
        deleteCookie = false;
        if (SendCookie != null)
        {
            SendCookie(cookie);
        }

        return true;
    }

    public void Dispose()
    {
    }
}
```
在browser事件中进行处理
```
private void browser_FrameLoadEnd(object sender, CefSharp.FrameLoadEndEventArgs e)
{
    var cookieManager = myChrome.GetCookieManager();

    CookieVisitor visitor = new CookieVisitor();
    visitor.SendCookie += visitor_SendCookie;
    cookieManager.VisitAllCookies(visitor);
}
```
**回调事件**
```
private void visitor_SendCookie(CefSharp.Cookie obj)
{
    Console.WriteLine("获取cookie：" + obj.Domain.TrimStart('.') + "^" + obj.Name + "^" + obj.Value + "$");
}
```
设置CefSharp实例事件：
```
private void myChrome_Loaded(object sender, RoutedEventArgs e)
{
    String url = "https://www.baidu.com";
    myChrome.Load(url);
    myChrome.FrameLoadEnd += browser_FrameLoadEnd;
}
```
运行程序，在`输出`视图就可以看到**cookie**数据了。

## Javascript交互

### C#执行js方法
```
myChrome.GetBrowser().MainFrame.ExecuteJavaScriptAsync("document.getElementById('testid').click();");  
```
以上代码就会触发id为`testid`的元素的`click`事件。  
注意：**脚本是在 Frame 级别执行**，页面永远至少有一个Frame（ MainFrame ）。

### 获取Javascript方法结果
这里需要使用`Task<JavascriptResponse> EvaluateScriptAsync(string script, TimeSpan? timeout)`方法。 JavaScript代码是异步执行的，因此使用.NET Task 类返回一个响应，其中包含错误消息，结果和一个成功（bool）标志。
```
// Get Document Height  
var task = frame.EvaluateScriptAsync("(function() { var body = document.body, html = document.documentElement; return  Math.max( body.scrollHeight, body.offsetHeight, html.clientHeight, html.scrollHeight, html.offsetHeight ); })();", null);
  
task.ContinueWith(t =>  
{  
    if (!t.IsFaulted)  
    {  
        var response = t.Result;  
        EvaluateJavaScriptResult = response.Success ? (response.Result ?? "null") : response.Message;  
    }  
}, TaskScheduler.FromCurrentSynchronizationContext());  
```



## 资源清理
关闭应用，发现`CefSharp.BrowserSubprocess.exe`进程会发现没有结束，其实在退出事件中，我们需要调用`Cef.Shutdown()`方法
```
try  
{  
    if (browser != null)  
    {  
        browser.Dispose();  
        Cef.Shutdown();  
    }  
}  
catch { }  
```





参考：
* [StackOverflow](https://stackoverflow.com/questions/31250797/chromium-send-custom-header-info-on-initial-page-load-c-sharp)
* [How to read the JSON response content from a XMLHttpRequest?](https://stackoverflow.com/questions/40944056/how-to-read-the-json-response-content-from-a-xmlhttprequest/43652932#43652932)
* [**CefSharp中文帮助文档**](https://github.com/cefsharp/CefSharp/wiki/CefSharp%E4%B8%AD%E6%96%87%E5%B8%AE%E5%8A%A9%E6%96%87%E6%A1%A3)