---
title: React Native Android Cookie Problem
tags:
  - React Native
  - Cookie
categories:
  - Java
  - JavaScript
abbrlink: 65079f86
date: 2015-11-04 13:50:31
---

#### 背景
&emsp;最近使用react native 来写一个公司内部使用的app，使用`fetch`去登陆，发现在android平台上无法获取cookie，iOS平台上却可以。即使是`response.headers.get()`也获得不了相关信息。于是上网google并且阅读源码，终于找到了问题出现的原因和解决方案。
#### 问题原因
&emsp;我们查看`native react`的`fetch.js`的代码，发现它的底层是使用`XmlHttpRequest`来实现的，然后再次找到'XmlHttpRequest'的相关源码，发现了三个文件`XMLHttpRequest.android.js`,`XMLHttpRequest.ios.js`和`XMLHttpRequestBase.js`。我们主要研究了android相关的文件。
&emsp;先看'XMLHttpRequest.android.js'。它继承了`XMLHttpRequestBase`
``` javascript
class XMLHttpRequest extends XMLHttpRequestBase {

  _requestId: ?number;

  constructor() {
    super();
    this._requestId = null;
  }

  sendImpl(method: ?string, url: ?string, headers: Object, data: any): void {
    var body;
    if (typeof data === 'string') {
      body = {string: data};
    } else if (data instanceof FormData) {
      body = {
        formData: data.getParts().map((part) => {
          part.headers = convertHeadersMapToArray(part.headers);
          return part;
        }),
      };
    } else {
      body = data;
    }
    //RCTNetWorking是android的native module，使用okhttp实现，我们之后会看到相关的代码
    this._requestId = RCTNetworking.sendRequest(
      method,
      url,
      convertHeadersMapToArray(headers),
      body,
      this.callback.bind(this)//这里是调用native module的回调，具体callback实现在XMLHttpRequestBase中。
    );
  }

  abortImpl(): void {
    this._requestId && RCTNetworking.abortRequest(this._requestId);
  }
}
```
&emsp;通过源码，我们可以了解，`XMLHttpRequest`就是通过Android Native Module 来发送网络请求的，然后会回调到`callback`函数中。我们接下来看看一下`callback`函数.
``` javascript
  callback(status: number, responseHeaders: ?Object, responseText: string): void {
    if (this._aborted) {
      return;
    }
    this.status = status;
    this.setResponseHeaders(responseHeaders || {});
    this.responseText = responseText;
    this.setReadyState(this.DONE);
  }
```
&emsp;这里我们发现`callback`回调有三个参数,`status`,`responseHeaders`和`responseText`,那么为什么在外层的`fetch`会拿不到`header`中的`cookie`呢？这里就需要研究android native module的实现啦。
&emsp;`RCTNetworking`对应的java文件为`NetworkingModule.java`，找到这个文件，直接看`sendRequest`函数。
``` java
@ReactMethod
public void sendRequest(
    String method,
    String url,
    int requestId,
    ReadableArray headers,
    ReadableMap data,
    final Callback callback) {
   
  //....... 无关代码省略
  mClient.newCall(requestBuilder.build()).enqueue(
      new com.squareup.okhttp.Callback() {
        @Override
        public void onFailure(Request request, IOException e) {
          if (mShuttingDown) {
            return;
          }
          callback.invoke(0, null, e.getMessage());
        }

        @Override
        public void onResponse(Response response) throws IOException {
          if (mShuttingDown) {
            return;
          }
          // TODO(5472580) handle headers properly
          String responseBody;
          try {
            
            responseBody = response.body().string();
          } catch (IOException e) {
            // The stream has been cancelled or closed, nothing we can do
            //这里是重点，我们发现第二个参数本该传递header,但是现在确实传的null，导致上层的js代码无法获得header!!!!
            callback.invoke(0, null, e.getMessage());
            return;
          }
          callback.invoke(response.code(), null, responseBody);
        }
      });
}
```
####解决方案
&emsp;在`fetch`或者`XMLHttpRequest`拿不到头部信息的问题找到了，那么如何让react native android实现cookie呢？
&emsp;方案有两套，一是：`callback.invoke`时把头部信息传递上去，让js层去做cookie的相关逻辑；二是:给`Okhttp`添加`CookieHandler`让`Okhttp`自己管理`Cookie`。
&emsp;第二套方案是我在github上看到的[github相关讨论和code](https://github.com/facebook/react-native/pull/3723/files)，相关代码在github上已经被merge到master上去啦，相信不久之后，新版本的`React Native Android`就可以把这个坑填上啦。

