---
title: Android-Async-Http 源码解析
tags:
  - 第三方库
  - Http
categories:
  - Android
  - Network
abbrlink: 68b9bfcd
date: 2015-09-27 14:37:49
---

&emsp;前几天去参加一个面试，被问到了一些android 网络方面的知识，发现自己在这个方面还有些不足，需要自我补充一下相关的知识，于是最近找了些开源的网络模块的第三方库来阅读，主要是想深入了解一下http协议和相关的代码框架组织问题。这篇博客就总结一下自己阅读android-async-http的一些体会和学习吧。

#### 简介
&emsp;介绍android async http 的相关事项，主要是翻译github上的话吧
&emsp;这段是官网翻译，大家请随意跳过,详细介绍请转到[http://loopj.com/android-async-http/]
&emsp;android-async-http是建立在Apache HttpClient之上的基于回调的异步android http client，它使用Handler机制，请求在UI线程之外发生，但是回调逻辑在UI线程中进行执行

> **特点**(部分)：
 + 执行异步request，在匿名回调中处理response
 + Http 请求 发生在UI线程之外
 + 使用ThreadPool来负载多线程消耗
 + GET/POST params 生成器
 + 支持文件断点续传
 + 支持自动重试
 + 支持流式Json数据上传
 + 可以处理重定向和请求循环
 + 自动的gzip压缩
 + 支持cookie
 + 可以通过BaseJsonResponseHandler和Jackson Json , Gson 和其他Json第三方库进行集成

#### 基本使用
```

  AsyncHttpClient client = new AsyncHttpClient();
  
  client.get("https://www.google.com", new       AsyncHttpResponseHandler() {   
      
      @Override
      public void onStart() {
          // 请求开始发生的回调    
      }
 
      @Override
      public void onSuccess(int statusCode, Header[] headers, byte[] responseBody) {
          // 成功获得response的回调
      }
 
      @Override
      public void onFailure(int statusCode, Header[] headers, byte[] responseBody, Throwable
  error)
  {
          // 失败的回调 :(
      }
 
      @Override
      public void onRetry(int retryNo) {
          // 请求被重试时的回调
      }
 
      @Override
      public void onProgress(long bytesWritten, long totalSize) {
          // 请求发生过程中的回调
      }
 
      @Override
      public void onFinish() {
          // 完成请求时的对调，未知成功还是失败
      }
  }); 
```

#### 源码解析

![主要类类图](http://7xjsjy.com1.z0.glb.clouddn.com/async_http_绘图1.jpg)
&emsp;上图就是android-async-http的主要类的类图，我们下面就来一个一个类解析一下
###### AsyncHttpClient
&emsp;先看AsyncHttpClient,它是这个http库的核心类之一，封装了发生http请求的所有逻辑，可以说它是这个库的中心类。它的构造函数如下:
```
      public AsyncHttpClient(SchemeRegistry schemeRegistry) {
        // http param
        BasicHttpParams httpParams = new BasicHttpParams();

        // connect params builder ?????
        ConnManagerParams.setTimeout(httpParams, connectTimeout);
        ConnManagerParams.setMaxConnectionsPerRoute(httpParams, new ConnPerRouteBean(maxConnections));
        ConnManagerParams.setMaxTotalConnections(httpParams, DEFAULT_MAX_CONNECTIONS);

        // httpConnectionParams
        HttpConnectionParams.setSoTimeout(httpParams, responseTimeout);
        HttpConnectionParams.setConnectionTimeout(httpParams, connectTimeout);
        HttpConnectionParams.setTcpNoDelay(httpParams, true);
        HttpConnectionParams.setSocketBufferSize(httpParams, DEFAULT_SOCKET_BUFFER_SIZE);

        HttpProtocolParams.setVersion(httpParams, HttpVersion.HTTP_1_1);

        ClientConnectionManager cm = createConnectionManager(schemeRegistry, httpParams);
        Utils.asserts(cm != null, "Custom implementation of #createConnectionManager(SchemeRegistry, BasicHttpParams) returned null");


        //thread poll
        threadPool = getDefaultThreadPool();

        /**
         * weakHashMap context:这是不会出现内存泄露
         * synchronizedMap 是建立一个线程安全的map加一个同步锁啊。
         */
        requestMap = Collections.synchronizedMap(new WeakHashMap<Context, List<RequestHandle>>());

        clientHeaderMap = new HashMap<String, String>();


        httpContext = new SyncBasicHttpContext(new BasicHttpContext());
        // 默认的apache的httpclient
        httpClient = new DefaultHttpClient(cm, httpParams);

        // 请求拦截器啊
        httpClient.addRequestInterceptor(new HttpRequestInterceptor() {
            @Override
            public void process(HttpRequest request, HttpContext context) {
                // 预处理,声明浏览器支持的编码类型啊
                if (!request.containsHeader(HEADER_ACCEPT_ENCODING)) { //加上GZIP的头部
                    request.addHeader(HEADER_ACCEPT_ENCODING, ENCODING_GZIP);
                }

                // 对于默认的clientHeader进行遍历，比较，添加
                for (String header : clientHeaderMap.keySet()) {
                    if (request.containsHeader(header)) {  //如果包含
                        Header overwritten = request.getFirstHeader(header);
                        log.d(LOG_TAG,
                                String.format("Headers were overwritten! (%s | %s) overwrites (%s | %s)",
                                        header, clientHeaderMap.get(header),
                                        overwritten.getName(), overwritten.getValue())
                        );

                        //remove the overwritten header
                        request.removeHeader(overwritten);
                    }
                    request.addHeader(header, clientHeaderMap.get(header)); //写入clientHeaderMap中的值
                }
            }
        });

        // response拦截器,对gzip进行解压缩啊
        httpClient.addResponseInterceptor(new HttpResponseInterceptor() {

            @Override
            public void process(HttpResponse response, HttpContext context) {
                final HttpEntity entity = response.getEntity(); //
                if (entity == null) {
                    return;
                }
                final Header encoding = entity.getContentEncoding();
                if (encoding != null) {
                    for (HeaderElement element : encoding.getElements()) { //遍历头部信息
                        if (element.getName().equalsIgnoreCase(ENCODING_GZIP)) {
                            response.setEntity(new InflatingEntity(entity)); //InflatingEntity 这是解压gzip的
                            break;
                        }
                    }
                }
            }
        });

        // 另外一个模块的请求拦截器 ???? 身份认证的
        httpClient.addRequestInterceptor(new HttpRequestInterceptor() {
            @Override
            public void process(final HttpRequest request, final HttpContext context) throws HttpException, IOException {
                AuthState authState = (AuthState) context.getAttribute(ClientContext.TARGET_AUTH_STATE);
                CredentialsProvider credsProvider = (CredentialsProvider) context.getAttribute(
                        ClientContext.CREDS_PROVIDER);
                HttpHost targetHost = (HttpHost) context.getAttribute(ExecutionContext.HTTP_TARGET_HOST);

                if (authState.getAuthScheme() == null) {
                    AuthScope authScope = new AuthScope(targetHost.getHostName(), targetHost.getPort());
                    Credentials creds = credsProvider.getCredentials(authScope);
                    if (creds != null) {
                        authState.setAuthScheme(new BasicScheme());
                        authState.setCredentials(creds);
                    }
                }
            }
        }, 0);

        // 重试Handler，用于重新发送请求
        httpClient.setHttpRequestRetryHandler(new RetryHandler(DEFAULT_MAX_RETRIES, DEFAULT_RETRY_SLEEP_TIME_MILLIS));
    }
```
&emsp;在构造函数中，AsyncHttpClient主要是初始化了threadPool作为发生请求的线程池，httpContext作为发生请求的网络context，还有最为重要的httpClient,它的类型是Apache的DefaultHttpClient，然后设置了设计gzip压缩和身份认证的请求拦截器和回应拦截器，具体逻辑代码中都有注释。
&emsp;AsyncHttp中还有一个我认为是整个库精髓所在的函数，如果你理解了这个函数，那么整个库的代码框架和思想其实你就已经知道了。它就是sendRequest(),AsyncHttp的关于网络请求的方法，比如get,post,head，最终都是调用了这个函数。在这个函数中，其他几个比较主要的类都有出现。
```
 /**
     *
     * 这是这里的重点啊，创建一个request放在队列中，等待一个thread去执行
     * Puts a new request in queue as a new thread in pool to be executed
     *
     * @param client          HttpClient to be used for request, can differ in single requests
     * @param contentType     MIME body type, for POST and PUT requests, may be null
     * @param context         Context of Android application, to hold the reference of request
     * @param httpContext     HttpContext in which the request will be executed
     * @param responseHandler ResponseHandler or its subclass to put the response into
     * @param uriRequest      instance of HttpUriRequest, which means it must be of HttpDelete,
     *                        HttpPost, HttpGet, HttpPut, etc.
     * @return RequestHandle of future request process
     */
    protected RequestHandle sendRequest(DefaultHttpClient client, HttpContext httpContext, HttpUriRequest uriRequest, String contentType, ResponseHandlerInterface responseHandler, Context context) {
        if (uriRequest == null) {
            throw new IllegalArgumentException("HttpUriRequest must not be null");
        }

        if (responseHandler == null) {
            throw new IllegalArgumentException("ResponseHandler must not be null");
        }

        if (responseHandler.getUseSynchronousMode() && !responseHandler.getUsePoolThread()) {
            throw new IllegalArgumentException("Synchronous ResponseHandler used in AsyncHttpClient. You should create your response handler in a looper thread or use SyncHttpClient instead.");
        }

        if (contentType != null) {
            if (uriRequest instanceof HttpEntityEnclosingRequestBase && ((HttpEntityEnclosingRequestBase) uriRequest).getEntity() != null && uriRequest.containsHeader(HEADER_CONTENT_TYPE)) {
                log.w(LOG_TAG, "Passed contentType will be ignored because HttpEntity sets content type");
            } else {
                uriRequest.setHeader(HEADER_CONTENT_TYPE, contentType);
            }
        }

        responseHandler.setRequestHeaders(uriRequest.getAllHeaders());
        responseHandler.setRequestURI(uriRequest.getURI());

        // runnable
        AsyncHttpRequest request = newAsyncHttpRequest(client, httpContext, uriRequest, contentType, responseHandler, context);

        threadPool.submit(request);

        // Handler 持有Request的弱引用，可以对其执行操作
        RequestHandle requestHandle = new RequestHandle(request);

        if (context != null) {  //如果此时context为不为空
            List<RequestHandle> requestList;
            // Add request to request map
            synchronized (requestMap) {  // 需要添加到context作为键的List中去
                requestList = requestMap.get(context);
                if (requestList == null) {
                    requestList = Collections.synchronizedList(new LinkedList<RequestHandle>());
                    requestMap.put(context, requestList);
                }
            }

            requestList.add(requestHandle);

            // 每次发送请求的时候进行一轮runnable删除
            Iterator<RequestHandle> iterator = requestList.iterator();
            while (iterator.hasNext()) {
                if (iterator.next().shouldBeGarbageCollected()) {
                    iterator.remove();
                }
            }
        }

        return requestHandle; //返回句柄啊
    }
```
&emsp;我们可以看到在函数中
+ 使用传入的参数和对象本身的成员变量来构造了一个AsynHttpRequest(本身继承Runnable,之后详细介绍),将其放入threadPool中
+ 然后生成RequestHandler对象来持有这个请求(用户可以通过RequestHandler对请求进行各类操作，比如取消),需要注意的是这个Handler并不是android的Handler，而是供用户操纵Request的句柄类
+ 然后更新requestMap,这是一个android Context作为主键的map，主要是记录各个Context的网络请求，需要注意的是这里使用的是[WeakHashMap](http://mikewang.blog.51cto.com/3826268/880775/),防止内存泄露
+ 添加Handler到requestList中，然后遍历requestList删除需要垃圾回收的对象
+ 最后返回requestHandler。

###### AsyncHttpRequest
&emsp;这个类是Runnable的子类,主要是用来进行发送请求和重试这套逻辑，而且ResponseHandlerInterface中的大多数回调函数都是在此对象中回调的。我们主要看一下它的`run()`,`makeRequest()`和 `makemakeRequestWithRetries()`方法
```
@Override
    public void run() {
        if (isCancelled()) {  //如果run的时候是取消状态，那么就关闭了
            return;
        }

        // Carry out pre-processing for this request only once.
        if (!isRequestPreProcessed) {  //必须进行一次预处理
            isRequestPreProcessed = true;
            onPreProcessRequest(this);
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendStartMessage();  // 回调handler，已经开始

        if (isCancelled()) {
            return;
        }

        // 进行带重试的请求
        try {
            makeRequestWithRetries();
        } catch (IOException e) {
            if (!isCancelled()) {
                responseHandler.sendFailureMessage(0, null, null, e);
            } else {
                AsyncHttpClient.log.e("AsyncHttpRequest", "makeRequestWithRetries returned error", e);
            }
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendFinishMessage();

        if (isCancelled()) {
            return;
        }

        // Carry out post-processing for this request.
        onPostProcessRequest(this);

        isFinished = true;
    }
```
&emsp;可以看出，在run中，根据发送网络请求的不同阶段调用了一系列的回调函数，其中比较重要的是`responseHandler.sendFinishMessage()`,在这里会回调函数进行解析response；在各个阶段开始前都有调用`isCancelled()`进行判断是否request被取消了。
&emsp;接下来是`makeRequest()`,在其中就调用了`HttpClient.execute(request,context)`进行正式的发送请求。
``` 
private void makeRequest() throws IOException {
        if (isCancelled()) {
            return;
        }

        // Fixes #115
        if (request.getURI().getScheme() == null) {
            // subclass of IOException so processed in the caller
            throw new MalformedURLException("No valid URI scheme was provided");
        }

        if (responseHandler instanceof RangeFileAsyncHttpResponseHandler) {
            ((RangeFileAsyncHttpResponseHandler) responseHandler).updateRequestHeaders(request);
        }

        HttpResponse response = client.execute(request, context);

        if (isCancelled()) {
            return;
        }

        // Carry out pre-processing for this response.
        responseHandler.onPreProcessResponse(responseHandler, response);

        if (isCancelled()) {
            return;
        }

        // The response is ready, handle it.
        responseHandler.sendResponseMessage(response);

        if (isCancelled()) {
            return;
        }

        // Carry out post-processing for this response.
        responseHandler.onPostProcessResponse(responseHandler, response);
    }
```
###### RequestHandler
&emsp;持有AsyncHttpRequest一个弱引用的句柄类，主要的功能是可以让客户端取消AsyncHttpRequest请求
###### AsyncHttpResponseHandler
&emsp;实现ResponseHandlerInterface的一个类，也是我们经常会用到的一个类，在android-async-http中有很多类都实现了ResponseHandlerInterface或者继承了这个类,比如BianryHttpResponseHandler,FileAsyncHttpResponseHandler,每个类都对应不同的网络请求的返回数据资源，有些可能是专门用于文件下载的，有些是解析json的，大家可以自己去了解各个类的作用。
&emsp;android-sync-http发送网络请求可以同步也可以异步，而且异步会在相应的thread中进行回调，其中涉及的逻辑就在这些类中。这个类也可以控制发送请求所使用的线程。
```
/**
     * Creates a new AsyncHttpResponseHandler with a user-supplied looper. If
     * the passed looper is null, the looper attached to the current thread will
     * be used.
     *
	 * 如果调用了这个，那么就不是异步，而且是在当前线程中调用了
     * @param looper The looper to work with
     */
    public AsyncHttpResponseHandler(Looper looper) {
        this.looper = looper == null ? Looper.myLooper() : looper;

        // Use asynchronous mode by default.
        setUseSynchronousMode(false);

        // Do not use the pool's thread to fire callbacks by default.
        setUsePoolThread(false);
    }
```
&emsp；大家可以发现构造函数传入了一个Looper,相信对android Handler机制比较了解的同学立即就知道这个库的异步调用是如何在当前线程中进行回调的了吧。这个类实现的回调函数中大多数都是进行`sendMessage(obtainMessage(FAILURE_MESSAGE, new Object[]{statusCode, headers, responseBody, throwable}));`这样的调用。
```
 protected void sendMessage(Message msg) {   // 如果是同步就自己处理，否则交由handler处理
        if (getUseSynchronousMode() || handler == null) {
            handleMessage(msg);
        } else if (!Thread.currentThread().isInterrupted()) { // do not send messages if request has been cancelled
            Utils.asserts(handler != null, "handler should not be null!");
            handler.sendMessage(msg);
        }
    }
```
&emsp;这就是发生message的函数，发现如果同步模式，那么就调用自己的handleMessage处理，否则交由构造函数中的Looper生成的handler进行处理,其实最终处理的还是这个对象的handlerMessage方法，但是是在另外一个Looper所在的线程中执行的。
&emsp；其实这个类中还有两个涉及http response解析的方法也是很重要的，就是`sendResponseMesssage(HttpResponse response)`和`getResponseData(HttpEntity entity)`，可是我对http协议不太了解，也害怕说错了，这里就只附上源码吧，上边有我的注释
```
// 获得网络请求返回进行处理
    @Override
    public void sendResponseMessage(HttpResponse response) throws IOException {
        // do not process if request has been cancelled
        if (!Thread.currentThread().isInterrupted()) {  //thread isInterrupted means that request is cancelled
            StatusLine status = response.getStatusLine(); // 状态行
            byte[] responseBody; // 回复体
            responseBody = getResponseData(response.getEntity());
            // additional cancellation check as getResponseData() can take non-zero time to process
            if (!Thread.currentThread().isInterrupted()) {
                if (status.getStatusCode() >= 300) {
                    sendFailureMessage(status.getStatusCode(), response.getAllHeaders(), responseBody, new HttpResponseException(status.getStatusCode(), status.getReasonPhrase()));
                } else {
                    sendSuccessMessage(status.getStatusCode(), response.getAllHeaders(), responseBody);
                }
            }
        }
    }

    /**
     * Returns byte array of response HttpEntity contents
	 * 解析http的response
     *
     * @param entity can be null
     * @return response entity body or null
     * @throws java.io.IOException if reading entity or creating byte array failed
     */
    byte[] getResponseData(HttpEntity entity) throws IOException {
        byte[] responseBody = null;
        if (entity != null) {
            InputStream instream = entity.getContent(); // 获得输入流
            if (instream != null) {
                long contentLength = entity.getContentLength(); // 获得size
                if (contentLength > Integer.MAX_VALUE) {
                    throw new IllegalArgumentException("HTTP entity too large to be buffered in memory");
                }
                int buffersize = (contentLength <= 0) ? BUFFER_SIZE : (int) contentLength; // contentLength有可能为负数吗
                try {
                    ByteArrayBuffer buffer = new ByteArrayBuffer(buffersize); // byte array的buffer类
                    try {
                        byte[] tmp = new byte[BUFFER_SIZE];
                        long count = 0;
                        int l;
                        // do not send messages if request has been cancelled
                        while ((l = instream.read(tmp)) != -1 && !Thread.currentThread().isInterrupted()) {
                            count += l;
                            buffer.append(tmp, 0, l);
                            sendProgressMessage(count, (contentLength <= 0 ? 1 : contentLength));
                        }
                    } finally {
                        AsyncHttpClient.silentCloseInputStream(instream);
                        AsyncHttpClient.endEntityViaReflection(entity);
                    }
                    responseBody = buffer.toByteArray();
                } catch (OutOfMemoryError e) {
                    System.gc();
                    throw new IOException("File too large to fit into available memory");
                }
            }
        }
        return responseBody;
    }
```
#### 总结
&emsp;android-async-http 算是源代码量最小的一个网络库了，当然它还有些却缺点，比如没有缓存机制，我看github中已经有人给它加上了缓存，大家也可以自己尝试一下，并且更好的是，这个库可以与其他的第三方库进行集成，大家可以打造属于自己的网络请求+处理数据的框架。
   

