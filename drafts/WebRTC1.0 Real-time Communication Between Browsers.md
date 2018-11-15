# WebRTC1.0: 浏览器间的实时通信

## 1.介绍

本规范涵盖了对等通信（peer-to-peer communications）和网页视频会议的多个方面：

- 利用ICE，STUN，TURN等NAT穿透技术连接至远程对等终端。
- 将本地产生的流媒体轨发送至远程对等终端并从远程对等终端接收流媒体轨。
- 直接向远程对等终端发送任意数据。

本文档针对这些特性定义了一些API。本规范是在[IETF RTCWEB](https://datatracker.ietf.org/wg/rtcweb/)工作组的协议规范，和[Media Capture Task Force](https://www.w3.org/wiki/Media_Capture)工作组的访问本地媒体设备API规范[GEtUSERMEDIA]，两者共同催动下发展出来的。系统的概览可以在引用列表中的[RTCWEB-OVERVIW](http://w3c.github.io/webrtc-pc/#bib-RTCWEB-OVERVIEW)和[RTCWEB-SECURITY](http://w3c.github.io/webrtc-pc/#bib-RTCWEB-SECURITY)找到。

## 2.一致性

关键词`MAY, MUST, MUST NOT, SHALL, SHOULD`的语义解释在RFC2119中都有描述。
本规范定义了适用于某个产品的一致性标准：**用户代理**应实现规范包含的接口。
一致性的要求可以被表述为一些算法，或被实现为任意行为的特定步骤，只要它们最终的结果是等效的。（特别地，规范中定义的算法更易理解，但性能也许不尽如人意。）
本规范中定义的API，必须（MUST）以WEBIDL中规定的行为一致的ECMAScript绑定的方式实现，毕竟我们使用了它们的规范与术语。

## 3.术语

[EventHandler](https://www.w3.org/TR/html51/webappapis.html#event-handler)接口代表了一个事件回调，ErrorEvent接口定义在[HTML51](http://w3c.github.io/webrtc-pc/#bib-HTML51)<br>
[任务入队（queue a task）](https://www.w3.org/TR/html51/webappapis.html#queuing)和[网络任务源（networking task source）](https://www.w3.org/TR/html51/webappapis.html#networking-task-source)的概念定义在[HTML51](http://w3c.github.io/webrtc-pc/#bib-HTML51)。<br>
[构造事件（fire an event）](https://dom.spec.whatwg.org/#firing-events)的概念定义在[DOM](http://w3c.github.io/webrtc-pc/#bib-DOM)。<br>
**事件（event）**，[事件句柄（event handler）](https://www.w3.org/TR/html51/webappapis.html#events-event-handlers)和[事件句柄类型（event handler event types）](https://www.w3.org/TR/html51/webappapis.html#event-handler-event-type)定义在[HTML51](http://w3c.github.io/webrtc-pc/#bib-HTML51)。<br>
[performance.timeOrigin](https://www.w3.org/TR/hr-time-2/#dom-performance-timeorigin)和[performance.now()](https://www.w3.org/TR/hr-time-2/#dom-performance-now)定义在[HIGHRES-TIME](http://w3c.github.io/webrtc-pc/#bib-HIGHRES-TIME)。<br>
[可序列化对象（serializable objects）](https://html.spec.whatwg.org/multipage/structured-data.html#serializable-objects)，[序列化步骤（serialization step）](https://html.spec.whatwg.org/multipage/structured-data.html#serialization-steps)，[反序列化步骤（deserialization steps）](https://html.spec.whatwg.org/multipage/structured-data.html#deserialization-steps)定义在[HTML](http://w3c.github.io/webrtc-pc/#bib-HTML)。<br>
**媒体流**（MediaStream），**媒体流轨**（MediaStreamTrack），**媒体流约束**（MediaStreamConstraints）定义在[GETUSERMEDIA](http://w3c.github.io/webrtc-pc/#bib-GETUSERMEDIA)。<br>
**Blob** 定义在[FILEAPI](http://w3c.github.io/webrtc-pc/#bib-FILEAPI)。<br>
**媒体描述**（media description）定义在[RFC4566](http://w3c.github.io/webrtc-pc/#bib-RFC4566)。<br>
**媒体传输**（media transport）定义在[RFC7656](http://w3c.github.io/webrtc-pc/#bib-RFC7656)。<br>
**代**（generation）定义在[TRICKLE-ICE](http://w3c.github.io/webrtc-pc/#bib-TRICKLE-ICE)的第二节。<br>
**RTCStatsType，stats object和monitored object** 定义在[WEBRTC-STATS](http://w3c.github.io/webrtc-pc/#bib-WEBRTC-STATS)。<br>
当引入异常时，[WEBIDL-1](http://w3c.github.io/webrtc-pc/#bib-WEBIDL-1)中定义了 **throw和create**。<br>
"throw"作为[INFRA](http://w3c.github.io/webrtc-pc/#bib-INFRA)中的规定来使用：它会终止目前正在运行的操作。<br>
`Promises`的上下文中使用的 **fulfilled, rejected, resolved, pending和settled** 在[ECMASCRIPT-6.0](http://w3c.github.io/webrtc-pc/#bib-ECMASCRIPT-6.0)中定义。<br>
**捆绑**（bundle），**只捆绑**（bundle-only）和**捆绑策略**（bundle-only）在[JSEP](http://w3c.github.io/webrtc-pc/#bib-JSEP)中定义。<br>
**OAuth客户端**（OAuth Client）和**授权服务**（Authorization Server）在[RFC6749](http://w3c.github.io/webrtc-pc/#bib-RFC6749)的1.1节被定义。<br>
**隔离流**（isolated stream），**对等身份**（peer identity），**请求身份断言**（request an identity assertion）和**身份验证**（validate the identity）在[WEBRTC-IDENTITY](http://w3c.github.io/webrtc-pc/#bib-WEBRTC-IDENTITY)中定义。

> 注意：
> 通常使用Javascript API的原则包括：`持续运行直到完成`和`无数据竞争`，它们都在[API-DESIGN-PRINCIPLES](http://w3c.github.io/webrtc-pc/#bib-API-DESIGN-PRINCIPLES)中定义了。也就是说，当一个任务正在运行时，任何外部事件都不会影响Javascript应用的可见性。例如，当Javascript执行时，缓存在数据通道里的数据数量将会随着"send"的调用而增长，并且直到任务的检查点之后，由于发送数据包导致的减少才被应用可见。
> 用户代理负责确保呈现给应用程序的的数据是一致的——例如`getContributingSources()`（同步调用）会返回当前所有被检测的数据源的值。

## 4. 对等连接

### 4.1 介绍

一个[RTCPeerConnection](http://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection)实例允许与另一个浏览器，或实现了制定协议的终端中的`RTCPeerConnection`实例建立对等通信。通信的过程通过交换信号通道中的控制信息（被称为信号协议）来协调，信号通道并没有明确的制定，但通常是服务页面中的一段脚本，例如[XMLHttpRequest](http://w3c.github.io/webrtc-pc/#bib-XMLHttpRequest)，也可以是[WebSockets](http://w3c.github.io/webrtc-pc/#bib-WEBSOCKETS-API)。

### 4.2 配置

#### 4.2.1 `RTCConfiguration`字典

`RTCConfiguration`定义了一系列用于配置如何通过`RTCPeerConnection`建立/重建对等通信的参数。

```webidl
dictionary RTCConfiguration {
  sequence<RTCIceServer> iceServers;
  RTCIceTransportPolicy iceTransportPolicy = "all";
  RTCBundlePolicy bundlePolicy = "balanced";
  RTCRtcpMuxPolicy rtcpMuxPolicy = "require";
  DOMString peerIdentity;
  sequence<RTCCertificate> certificates;
  [EnforceRange]
  octet iceCandidatePoolSize = 0;
};
```