title: OkHttp解析系列-重定向和出错重试
date: 2015-11-18 22:00:10
tags: 
	- OkHttp
categories:
	- Android
	- NetWork
---

&emps;这是OkHttp系列博文的第一篇，之前写过一篇草稿，介绍OkHttp的整体框架，但是感觉涉及的知识太多，无法在一篇中讲述清楚，所以，之后的博文都只关注某一方面的知识，争取文章短小精悍。
&emsp;今天主要研究一下OkHttp发送`Http`请求过程中的重定向和出错重试，主要涉及的源码文件有`Call.java``HttpEngine.java`。
&emsp;我们今天研究`Call`的`Response getResponse(Request request, boolean forWebSocket) throws IOException`函数，它是你调用`Call.execute()`返回`Response`所调用的核心函数，主要功能是新建一个`HttpEngine`发送`Request`然后处理出错重试和重定向问题。
#### 设置Headers
``` java
  // Copy body metadata to the appropriate request headers.
    RequestBody body = request.body();
    if (body != null) {
      Request.Builder requestBuilder = request.newBuilder();//拷贝了内部数据

      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }

      request = requestBuilder.build();
    }
```
&emsp;这是函数的第一部分，主要是将`RequestBody`的一些元数据拷贝到`Header`的首部中,主要是`Content-Type`和`Transfer-Encoding`。`Content-Type`相信大家都了解，标示`RequestBody`的`Mime-Type`，格式为`主类型/子类型`，比如`text/xml`。而`Transfer-Encoding`是表示一种网络传输的方式，想具体了解的同学可以看一下这个链接[点我](https://imququ.com/post/transfer-encoding-header-in-http.html).
#### 出错重试
``` java
// Create the initial HTTP engine. Retries and redirects need new engine for each attempt.
    // 建立一个初始的http 引擎，每次重试和重定向都需要新的引擎
    engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null, null);

    int followUpCount = 0; //连续发送请求
    while (true) {
      if (canceled) { //如果被取消啦
        engine.releaseConnection();
        throw new IOException("Canceled");
      }

      try {
        engine.sendRequest();
        engine.readResponse();
      } catch (RequestException e) {
        // The attempt to interpret the request failed. Give up.
        throw e.getCause();
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        HttpEngine retryEngine = engine.recover(e); //重试引擎
        if (retryEngine != null) {
          engine = retryEngine;
          continue;
        }
        // Give up; recovery is not possible.
        throw e.getLastConnectException();
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        HttpEngine retryEngine = engine.recover(e, null);
        if (retryEngine != null) {
          engine = retryEngine;
          continue;
        }

        // Give up; recovery is not possible.
        throw e;
      }
      .......
```
&emsp;在这段代码中，`OkHttp`建立一个`HttpEngine`对象来负责`Http`层级的请求的发送和回复的接收，`HttpEngine`会在之后的博文中详细讲解。然后进入了一个`while`循环,这个循环其实主要是处理重定向问题的。我们在这一节中主要关注`catch`中的逻辑，这是用于处理出错重试的逻辑。由于外层有一个`while`循环，所以在`catch`中尝试获得`retryEngine`，如果有就`continue`,没有就抛出异常。
#### 重定向处理
``` java
Response response = engine.getResponse();
      // followUp这个是优化http connection的使用率的吗？
      Request followUp = engine.followUpRequest();

      if (followUp == null) {
        if (!forWebSocket) { //如果没有followup并且不是为了websocket
          engine.releaseConnection();//关闭连接
        }
        return response;
      }

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (!engine.sameConnection(followUp.httpUrl())) { //如果followup的httpUrl不是同一个连接,也就是
        //schema，host or port 有一个不同
        engine.releaseConnection();
      }
      //复用了上一次的connection啊！！！！
      Connection connection = engine.close();
      request = followUp;
      //继续处理，有可能是重定向啦
      engine = new HttpEngine(client, request, false, false, forWebSocket, connection, null, null,
          response);
```
&emsp;这里我们可以看到`Http`重定向的机制。`Request request = engine.followUpRequest()`来获得重定向需要发送的`Request`，如果没有或者重定向次数大于`MAX_FOLLOW_UPS`就不会重新发送重定向请求。然后判断重定向请求和原请求的HttpUrl是否相同，否则也不会发送重定向请求。然后`Connection connection = engine.close()`会释放资源并且复用上次的连接，然后新建一个`HttpEngine`然后继续`While`循环发送请求。
#### 重定向状态码解析
``` java
  public Request followUpRequest() throws IOException {
    if (userResponse == null) throw new IllegalStateException();
    Proxy selectedProxy = getRoute() != null
        ? getRoute().getProxy()
        : client.getProxy();
    int responseCode = userResponse.code();

    switch (responseCode) {
      case HTTP_PROXY_AUTH: //407 Proxy authentication required 要先经过代理服务器认证
        if (selectedProxy.type() != Proxy.Type.HTTP) {
          throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
        }
        // fall-through
      case HTTP_UNAUTHORIZED: //401 没有身份认证
        return OkHeaders.processAuthHeader(client.getAuthenticator(), userResponse, selectedProxy);

      case HTTP_PERM_REDIRECT:// 308
      case HTTP_TEMP_REDIRECT: //307
        // "If the 307 or 308 status code is received in response to a request other than GET
        // or HEAD, the user agent MUST NOT automatically redirect the request"
        if (!userRequest.method().equals("GET") && !userRequest.method().equals("HEAD")) {
            return null;
        } //如果不是get和head 那么就不能自动转发
        // fall-through
      case HTTP_MULT_CHOICE: //300
      case HTTP_MOVED_PERM:// 301
      case HTTP_MOVED_TEMP://302
      case HTTP_SEE_OTHER: //303
        // Does the client allow redirects?
        if (!client.getFollowRedirects()) return null;//如果不允许重定向

        String location = userResponse.header("Location");//从response的头部获得的location
        if (location == null) return null;
        HttpUrl url = userRequest.httpUrl().resolve(location);//使用request的解析location

        // Don't follow redirects to unsupported protocols.
        if (url == null) return null;

        // If configured, don't follow redirects between SSL and non-SSL.
        boolean sameScheme = url.scheme().equals(userRequest.httpUrl().scheme());
        if (!sameScheme && !client.getFollowSslRedirects()) return null;

        // Redirects don't include a request body.
        Request.Builder requestBuilder = userRequest.newBuilder();
        if (HttpMethod.permitsRequestBody(userRequest.method())) {
          requestBuilder.method("GET", null);
          requestBuilder.removeHeader("Transfer-Encoding");
          requestBuilder.removeHeader("Content-Length");
          requestBuilder.removeHeader("Content-Type");
        }

        // When redirecting across hosts, drop all authentication headers. This
        // is potentially annoying to the application layer since they have no
        // way to retain them.
        if (!sameConnection(url)) {
          requestBuilder.removeHeader("Authorization");
        }

        return requestBuilder.url(url).build();

      default:
        return null;
    }
```
&emsp;这一段就是根据回复的状态码生成重定向请求的代码逻辑。

- HTTP_PROXY_AUTH 407  表示需要经过代理服务器认证 ，这时抛出异常，不进行重定向
- HTTP_UNAUTHORIZED 401 身份未认证
- HTTP_PERM_REDIRECT 308 HTTP_TEMP_REDIRECT 307 这两种状态码时，只有当请求的`method`不为`GET`和`HEAD`时不进行重定向，否则按照下边一列状态码的方式处理
- HTTP_MULT_CHOICE  300 HTTP_MOVED_PERM 301 HTTP_MOVED_TEMP 302 HTTP_SEE_OTHER 303 当是这些状态码时，先判断是否运行重定向，然后获得`Response`中的`Location`首部的值，然后用`HttpUrl`去解析，如果是`host`不同，那么去掉所有的认证首部，这是为了安全。
#### 结语
&emsp;今天所总结的只是`Http`的重定向部分和`OkHttp`中的关于重定向的逻辑部分。之后会陆陆续续的继续总结关于`Http`的知识。
