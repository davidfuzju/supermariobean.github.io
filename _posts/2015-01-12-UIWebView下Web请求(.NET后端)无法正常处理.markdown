---
layout: post
category: Git
---

## BUG

  这是一个bug引发的故事，主要涉及的技术环境：客户端部分，在iOS 8.1.2系统下，使用控件UIWebView实现了native部分的Hybrid框架；Web部分，以.NET 4.0环境作为后端，H5+js+css作为了H5部分的hybrid前端框架，并简单基于框架写了一个应用，应用暂时只对向后端做一次简单ajax，在请求之前，通过将native中保存的登录态转移至UIWebView中的localstorage实现登录信息传递，而本次请求也主要是携带Auth信息确认用户登录态而已。

  既然是一个bug，那就按bug单来对待

  + What steps will reproduce the problem?

    通过native端的自定义UIWebView打开指定的URL链接，就会出现不正常的结果

  + What is the expected output? What do you see instead?

    期望输出：指定的URL链接正确通过登录态检查，显示出正常的登录验证后页面
    实际输出：登录态验证失败，页面逻辑失败

  + What version of the product are you using? On what operating system?

    iOS 8.1.2 iPhone5s, .NET 4.0

  + Please provide any additional information below. / Please use labels and text to provide additional information.

    就目前的测试来看，Android系统没有任何问题，该问题只发生在iOS系统上；
    而在iOS系统上，自带浏览器Safari没有任何问题；只有使用开发组件UIWebView才会出现；
    服务端日志可以看出，请求进入了请求处理逻辑，但是其中的POST参数为空，服务端按照普通的异常逻辑处理并响应结果

## 解决过程

  bug单的信息已经描述的比较清晰，但是还是要自己亲自验证一遍，经过验证后结果也基本相同。问题的重点就在Safari可用和UIWebView不可用上了。

  有一点需要提一下的，由于使用了Hybrid框架，所以通过Safari打开URL，Web APP的生命周期是一个单纯的Web应用；但是通过natitve端Hybrid的UIWebView打开URL，因为桥接JS代码的作用，该Web APP的部分数据通过桥接JS代码在native和H5之间相互传递。

  为了排除是自己代码的问题，因为继承了框架中的UIWebView的子类，虽然实现的逻辑不多，但也需要排除。通过应用中的其他native Hybrid入口加载URL，得到了同样结果，所以首先可以确认自己的代码没有问题。既然是Web APP，浏览器调试少不了，通过Safari与iOS APP内的UIWebVIew联调，没有发现其它什么明显的错误，于是直接找到登录验证的ajax请求，查看详细的请求和响应内容，发现了以下问题：

  + 1.请求路径被加入了一段奇怪的字符串，而正常运行的Safari环境下请求的路径没有发生任何改变。

        http://m.supermariobean.com/(F(vy2ebt45imfkmjjwboow3l55 ))/ajax.ashx

  + 2.请求POST参数正常。这与服务端的请求处理逻辑中打出的日志显示POST参数传入为空矛盾。

  + 3.响应体结构正常，内容是处理POST参数为空情况下返回的响应。这与服务端的日志吻合。

  以上大致说明了几个问题，在自定制的Hybrid框架环境中，ajax请求的路径被重写了，增加了一个字符串；请求被IIS接受，并被路由到了正确了处理函数中，说明这一段字符串并不是无意义的，既然可以被IIS接受，那么很可能是.NET框架在处理请求响应的过程中加入的；请求POST参数在发送时存在，IIS接收后路由到相应的处理函数后就没有了，进一步说明了问题可能出在IIS和.NET框架中。
  
  在验证这个猜想之前，我们还是尽量排除自定制的Hybrid框架的干扰。分别对Hybrid框架的native端和H5端都进行了跟踪调试，发现，native端压根就没有捕获并修改ajax请求的机会，而H5端，跟踪js代码直到请求url被移交给XHttpRequest对象，url都没有任何改动。
  
  ***至此，基本可以确定，公司框架也没有问题，问题应该是iOS的Cocoa Touch框架、IIS Web服务器和.NET框架***
  
  一想到微软和苹果这两家公司的恩怨，我就知道应该没错了...

  因为大致定位了问题的方向，搜索的关键字确定如下：

    iOS UIWebView urlrewrite .NET IIS

  在搜索结果中定位了微软关于IIS[Using the URL Rewrite Module](http://www.iis.net/learn/extensions/url-rewrite-module/using-the-url-rewrite-module)的文章，已知问题这一节如下：

  >Known Issues

  >UseUri mode in ASP.NET Forms authentication uses rewritten URL for redirection. For example, if the requested URL is "/article.htm" and URL Rewrite module rewrites the URL to "/article.aspx", which is protected by Forms authentication, then ASP.NET will redirect to "/(S(vy2ebt45imfkmjjwboow3l55 ))/article.aspx".

  >ASP.NET rewrites back to the original URL when using URI-based authentication or cookie-less session state. For example, when a request is made to "/(S(vy2ebt45imfkmjjwboow3l55 ))/article.htm" and URL rewrite module rewrites "/article.htm" to "/article.aspx", then ASP.NET will rewrite the URL back to "/article.htm", which may result in a "404 - File Not Found" error.

  如前所述，Web APP并没有使用任何authentication，所以排除这里面的authentication条件，就只剩下cookie-less session state。大致的意思是在IIS中，如果发送请求的UserAgent处于cookie disable的状态，则会是请求的url重写为带有`/(S(vy2ebt45imfkmjjwboow3l55 ))`的形式，虽然目前还不知道重写后为何可以响应请求，但无法得到POST参数的原因。至少找到一条新的线索，也就是cookie开启的问题。

  接下来搜索关键字如下：

    iOS UIWebView enable cookie .NET

  [Enable Cookies in UIWebView (iPhone)](http://stackoverflow.com/questions/9794999/enable-cookies-in-uiwebview-iphone)中介绍了一种非常类似的情况。当使用ASP.NET作为后端，而使用UIWebView作为UserAgent时，需要修改请求的UserAgent字段，否则会导致问题，同时也会重定向到另一中模式的url

    /(F(vy2ebt45imfkmjjwboow3l55 ))/

  情况很类似，但是仍旧不是正确解答。通过重新查看Hybrid框架的native代码，发现框架中已经考虑过了UserAgent的重写和cookie问题，也就是说，客户端应该是正确处理的。这是虽然有点气馁，但是还有一个地方，就是服务端和WEB服务器配置。后面又得知后端并没有使用公司现有的gateway模式，所以很有可能是后端问题，于是搜索如下：

    UIWebView cookie .NET

  搜到了SO中的问答[Asp.Net Forms Authentication when using iPhone UIWebView](http://stackoverflow.com/questions/4158550/asp-net-forms-authentication-when-using-iphone-uiwebview)，在题主所遇到的问题和处境和我们非常相似：

  + safari正常，UIWebView有问题（题注的测试更详细，几乎所有的浏览器都正常）

  + cookie开启后依旧不正确

  最终的解答是需要在.NET项目中加入一个`.browser`文件，并在文件中声明如下xml:

    <browser refID="Mozilla" >
      <capabilities>
        <capability name="cookies"  value="true" />
      </capabilities>
    </browser>

  有关`browser`这个元素的解释链接在[MSDN](http://msdn.microsoft.com/en-us/library/ms228122(v=vs.85).aspx)。虽然题主的问题是基于form authentication，但是同样适用于cookieless session state的情况。最终可以解决这个BUG的原因可能是，.NET框架对于`Mozilla`的每一种Useragent(safari也是其中之一)都有相应的xml配置，但是由于UIWebView这种特殊的组件，导致了.NET对于其Useragent的默认处理，从服务端disable了其cookie的使用，这也就解释了为何在客户端设置cookie enable不起作用的原因。在.NET项目中设置`Mozilla`的默认cookie设置，也就开启了UIWebView对应Useragent的cookie功能，从而解决了这个问题。
