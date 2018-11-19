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
**隔离流**（isolated stream），**对等身份**（peer identity），**请求身份断言**（request an identity assertion）和**身份认证**（validate the identity）在[WEBRTC-IDENTITY](http://w3c.github.io/webrtc-pc/#bib-WEBRTC-IDENTITY)中定义。

> 注意：
> 通常使用Javascript API的原则包括：`持续运行直到完成`和`无数据竞争`，它们都在[API-DESIGN-PRINCIPLES](http://w3c.github.io/webrtc-pc/#bib-API-DESIGN-PRINCIPLES)中定义了。也就是说，当一个任务正在运行时，任何外部事件都不会影响Javascript应用的可见性。例如，当Javascript执行时，缓存在数据通道里的数据数量将会随着"send"的调用而增长，并且直到任务的检查点之后，由于发送数据包导致的减少才被应用可见。
> 用户代理负责确保呈现给应用程序的的数据是一致的——例如`getContributingSources()`（同步调用）会返回当前所有被检测的数据源的值。

## 4. 对等连接

### 4.1 介绍

一个[RTCPeerConnection](http://w3c.github.io/webrtc-pc/#dom-rtcpeerconnection)实例允许与另一个浏览器，或实现了制定协议的终端中的`RTCPeerConnection`实例建立对等通信。通信的过程通过交换信号通道中的控制信息（被称为信令协议）来协调，信号通道并没有明确的制定，但通常是服务页面中的一段脚本，例如[XMLHttpRequest](http://w3c.github.io/webrtc-pc/#bib-XMLHttpRequest)，也可以是[WebSockets](http://w3c.github.io/webrtc-pc/#bib-WEBSOCKETS-API)。

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

`RTCConfiguration`字典成员变量：

- sequence<RTCIceServer>类型的的`iceServers`：描述可供ICE使用的服务对象数组，例如STUN服务和TURN服务。
- RTCIceTransportPolicy类型的`iceTransportPolicy`，缺省值为"all"：指示哪个候选`ICE Agent`可用。
- RTCBundlePolicy类型的`bundlePolicy`，缺省值为"balanced"：当收集候选ICE时指示使用什么媒体捆绑策略。
- RTCRtcpMuxPolicy类型的`rtcMuxPolicy`，缺省值为"require"：当收集候选ICE时指示使用什么RTCP复用策略。
- DOMString类型的`peerIdentity`：为RTCPeerConnection设置目标对等终端的身份。只有成功地对身份进行鉴权，RTCPeerConnection才能与远程对等终端建立起连接。
- sequence<RTCCertificate>类型的`certificates`：RTCPeerConnection鉴权时所需的一系列证书。<br>  此参数的合法值通过调用`generateCertificate`函数得到。<br>  尽管任意给定的DTLS连接只会使用一份证书，但这一属性使得调用方可以提供多种证书以支持不同的算法。在DTLS连接的握手阶段，它会最终选择一份允许范围内的证书。RTCPeerConnection的具体实现中完成了对给定连接的证书选择过程，但证书是如何选择的并不在本规范的讨论范围之内。<br>  如果值为空，则每个RTCPeerConnection实例都会生成默认的证书集合。<br>  此选项还使得应用的密钥连续性成为可能。一个`RTCCertificate`可以被持久化存储在[INDEXEDDB](https://www.w3.org/TR/webrtc/#bib-INDEXEDDB)中并被复用。持久化和复用避免了密钥重复生成的开销。<br>  此配置选项的值在初始化阶段被选择后就不能再被改变。
- `octet`类型的`iceCandidatePoolSize`，缺省值为`0`：预先获取的ICE池的大小在[JSEP](https://www.w3.org/TR/webrtc/#bib-JSEP)的第3.5.4节和4.1.1节被定义。

#### 4.2.2 `RTCIceCredentialType`枚举值

```webidl
enum RTCIceCredentialType {
    "password",
    "oauth"
};
```

枚举值简述：

- password：此凭据是依托于用户名和密码的长期认证方式，[RFC5389](https://www.w3.org/TR/webrtc/#bib-RFC5389)的10.2节有详细描述

- oauth：一个基于OAuth2.0的认证方法，在[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)有描述。<br>  对于OAuth认证，需要向ICE Agent提供3份凭证信息：`kid`（用于RTCIceServer成员变量username），`macKey`和`accessToken`（存在于RTCOAuthCredential字典类型内）。<br>  **注意：本规范并没有定义应用（起OAuth Client的作用）是如何从`Authorization Server`获取`accessToken, kid, macKey`这些凭证的，因为WebRTC只处理ICE Agent与TURN Server之间的交互。例如，应用可能使用PoP（Proof-of-Possession）的Token凭证类型，使用OAuth 2.0隐式授权类型。[RFC](https://www.w3.org/TR/webrtc/#bib-RFC7635)的附录B中有此示例。** <br>  OAuth Client应用，负责刷新凭证信息，并且在`accessToken`失效前利用新的凭证信息更新ICE Agent。OAuth Client可以利用RTCPeerConnection的setConfiguration方法来周期性的刷新TURN凭证。<br>  HMAC密钥（RTCOAuthCredential.macKey）的长度应是一个大于20字节的整数（160位）。<br> **注意：根据[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)4.1节，HMAC密钥必须是对称密钥，但对称密钥会生成大型的访问令牌，可能和单个STUN信息不兼容。** <br>  **注意：目前的STUN/TURN协议只是用了SHA-1/SHA-2族哈希算法来保证消息完整性，这在[RFC5389]的15.3节和[STUN-BIS]的14.6节作了定义。**

#### 4.2.3 `RTCOAuthCredential`字典

`RTCOAuthCredential`字典被STUN/TURN客户端（内置于ICE Agent内）用于描述OAuth的鉴权凭证信息，对STUN/TURN服务器进行身份认证，[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)有相关描述。注意`kid`参数并不在此字典类型中，而在`RTCIceServer`的`username`成员变量中。

```webidl
dictionary RTCOAuthCredential {
    required DOMString macKey;
    required DOMString accessToken;
};
```

`RTCOAuthCredential`字典的成员变量：

- DOMString类型的`macKey`，非空："mac_key"是一串base64-url格式的编码，在[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)的6.2节有相关描述。它被用在STUN的消息完整性哈希计算中（密码使用的则是基于密码的认证方式）。注意，OAuth响应里的"key"参数是一个JSON Web Key（JWK）或JWK编码后JWE格式的消息。同样注意，这是OAuth中唯一一个不被直接使用的参数，它只能从JWK的"k"参数中提取出来，"k"参数包含了需要的base-64编码的"mac_key"。
- DOMString类型的`accessToken`，非空："access_token"是一串base64格式的编码，在[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)的6.2节有相关描述。这是一个自持有的令牌，应用不可见。认证加密被用于消息的加密和完整性保护。访问令牌包括了一个未加密的nonce值，供认证服务生成唯一的`mac_key`。令牌的第二部分由认证加密服务保护着，包括mac_key，时间戳和生存时间。时间戳和生存时间共同组成了过期信息，过期信息描述了令牌凭证合法且能被TURN服务接受的时间窗口。

RTCOAuthCredential字典的一个例子：

```js
// EXAMPLE 1
{
  macKey: 'WmtzanB3ZW9peFhtdm42NzUzNG0=',
  accessToken: 'AAwg3kPHWPfvk9bDFL936wYvkoctMADzQ5VhNDgeMR3+ZlZ35byg972fW8QjpEl7bx91YLBPFsIhsxloWcXPhA=='
}
```

#### 4.2.4 `RTCIceServer`字典

`RTCIceServer`字典被ICE Agent用来描述和对等终端建立连接的STUN/TURN服务器信息。

```webidl
dictionary RTCIceServer {
    required (DOMString or sequence<DOMString>) urls;
             DOMString                          username;
             (DOMString or RTCOAuthCredential)  credential;
             RTCIceCredentialType               credentialType = "password";
};
```

`RTCIceServer`字典的成员变量：

- DOMString或sequence<DOMString>类型的`urls`，非空：[RFC7064]和[RFC7065]中定义的STUN/TURN的URI(s)，或其他的URI类型。
- DOMString类型的`username`：如果`RTCIceServer`代表了一个TURN服务器，且`credentialType`是`"password"`，那么这一属性指定的是TURN服务器使用的用户名。<br>  如果`RTCIceServer`代表了一个TURN服务器，且`credentialType`是`"oauth"`，那么这一属性指定的是TURN服务器和认证服务器之间共享的对称密钥的密钥id，[RFC7635](https://www.w3.org/TR/webrtc/#bib-RFC7635)有相关描述。这是一个短暂且唯一的密钥标识符。`kid`允许TURN服务器选择合适的密钥材料对访问令牌进行解密，因此以`kid`为代表的密钥标识符被用于"access_token"的加密。`kid`值和OAuth响应中的"kid"参数相同，这被定义在[RFC7515](https://www.w3.org/TR/webrtc/#bib-RFC7515)的4.1.4节。
- DOMString或RTCOAuthCredential类型的`credential`：如果`RTCIceServer`代表了一个TURN服务器，那么这一属性指定的是TURN服务器使用的凭证。<br> 如果`credentialType`是`"password"`，那么`credential`是`DOMString`类型，代表了长期使用的认证密码，这在[RFC5389](https://www.w3.org/TR/webrtc/#bib-RFC5389)的10.2节有相关描述。<br>  如果`credentialType`是`"oauth"`，那么`credential`是`RTCOAuthCredential`类型，包含了OAuth访问令牌和MAC值。
- RTCIceCredentialType类型的`credentialType`，默认值为"password"：如果`RTCIceServer`代表了一个TURN服务器，那么这一属性指定了当TURN服务器请求认证的时候*credential*是如何被使用的。

一个RTCIceServer对象数组的例子：

```js
[
  {urls: 'stun:stun1.example.net'},
  {urls: ['turns:turn.example.org', 'turn:turn.example.net'],
    username: 'user',
    credential: 'myPassword',
    credentialType: 'password'},
  {urls: 'turns:turn2.example.net',
    username: '22BIjxU93h/IgwEb',
    credential: {
      macKey: 'WmtzanB3ZW9peFhtdm42NzUzNG0=',
      accessToken: 'AAwg3kPHWPfvk9bDFL936wYvkoctMADzQ5VhNDgeMR3+ZlZ35byg972fW8QjpEl7bx91YLBPFsIhsxloWcXPhA=='
    },
    credentialType: 'oauth'}
];
```

#### 4.2.5 `RTCIceTransportPolicy`枚举值

如[JSEP](https://tools.ietf.org/html/draft-ietf-rtcweb-jsep-24#section-4.1.1)4.1.1节所定义，如果`RTCConfiguration`的`iceTransportPolicy`成员被指定，它将定义浏览器使用的ICE候选策略[JSEP 3.5.3节](https://www.w3.org/TR/webrtc/#bib-JSEP)，以向应用提供允许的候选者；只有候选者可被用于连接性检查。

```webidl
enum RTCIceTransportPolicy {
    "relay",
    "all"
};
```

枚举值的非规范描述：

- relay：ICE Agent仅适用媒体中级候选者，例如通过TURN服务器传递的候选者。**注意：这可以在某些特定场景下防止远程终端获取用户的IP地址。例如，在一个基于“调用”的应用中，应用可能想防止某个未知的调用者获得被调用方得IP地址，除非被调用方以某些同意。**
- all：当被指定为"all"时，ICE Agent可以使用任意类型的候选者。**注意：在具体实现中，仍然可以使用自己的候选者过渡策略来限制暴露给应用的IP地址，这在RTCIceCandidate.address中有提到。**

#### 4.2.6 `RTCBundlePolicy`枚举值

如[JSEP 4.1.1节](https://www.w3.org/TR/webrtc/#bib-JSEP)提到，如果远程端点不支持捆绑，则捆绑策略会影响哪些媒体轨参与协商，以及哪些ICE候选者被收集。如果远程端点支持捆绑，所有媒体轨和数据通道都会被捆绑到同一传输路径上。

```webidl
enum RTCBundlePolicy {
    "balanced",
    "max-compat",
    "max-bundle"
};
```

枚举值的非规范描述：

- balanced：为所有正在使用中的媒体类型（音频，视频和数据）收集ICE候选者。如果远程端点不支持捆绑，则只会为每个独立的传输协商一个音频或视频。
- max-compat：为每个流媒体轨收集ICE候选者。如果远程端点不支持捆绑，为每个独立传输协商所有的媒体轨。
- max-bundle：只为一个媒体轨收集ICE候选者。如果远程端点不支持捆绑，只协商一个媒体轨。

#### 4.2.7 `RTCRtcpMuxPolicy`枚举值

如[JSEP 4.1.1节](https://www.w3.org/TR/webrtc/#bib-JSEP)中描述的，RtcpMuxPolicy会影响ICE候选者收集哪些内容以支持非多路复用RTCP。

```webidl
enum RTCRtcpMuxPolicy {
    // At risk due to lack of implementers' interest.
    "negotiate",
    "require"
};
```

枚举值的非规范描述：

- negotiate：同时收集RTP候选者和RTCP候选者。如果远程端点能够复用RTCP，则在RTP候选者之上复用RTCP。如果不能，独立地使用RTP和RTCP候选者。注意，[JSEP 4.1.1节](https://www.w3.org/TR/webrtc/#bib-JSEP)提到，用户代理可能没有实现不复用的RTCP，在这种情况下所有试图以`negotiate`策略构造`RTCPeerConnection`的操作都会被拒绝。
- require：只收集RTC候选者和在RTP基础上复用了RTCP的候选者。如果远程端点不支持rtcp复用，那么会话协商将失败。

> <big>风险特征</big>：支持非多路复用RTP/RTCP的本规范的各个方面被标记为存在风险的特征，因为实现者没有明确的承诺。包括：1. 对于`negotiate`值，实现者没有明确承诺与此相关的行为。 2.在`RTCRtpSender`和`RTCRtpReceiver`之内支持`rtcpTransport`属性。

#### 4.2.8 提供/应答选项

这些字典类型描述了可用于提供/应答创建过程的选项。

```webidl
dictionary RTCOfferAnswerOptions {
    boolean voiceActivityDetection = true;
};
```

`RTCOfferAnswerOptions`成员变量：

- boolean类型的`voiceActivityDetection`，缺省值为"true"：很多编解码器和系统都能够检测到“静音”，并改变它们的行为，例如不传输任何媒体信息。在很多场景下，例如处理紧急呼叫或不仅仅人声之外的语音时，我们希望能够关闭这个选项。这个选项允许应用提供关于是否希望开启或关闭这类处理的信息。

```webidl
dictionary RTCOfferOptions : RTCOfferAnswerOptions {
    boolean iceRestart = false;
};
```

`RTCOfferOptions`成员变量：

- boolean类型的`iceRestart`，缺省值为"false"：当此值为true时，会生成与当前凭证（在`localDescription`属性的SDP中可见）不同的ICE凭证。应用此描述将重启ICE，具体描述在[ICE](https://www.w3.org/TR/webrtc/#bib-ICE)的9.1.1.1节。<br>  当此值为false，`localDescription`属性具有有效的ICE凭证，生成的描述将和当前的`localDescription`属性一致。 **注意：当`iceConnectionState`转换为"failed"时，建议执行ICE重启。应用也可能额外监听`iceConnectionState`到"disconnected"的变化，然后使用其他信息来源（比如使用`getStats`测量接下来几秒内发送或接收的字节数是否增加）确定是否应该重启ICE。**

`RTCAnswerOptions`字典描述了指定`answer`类型会话的选项。

```webidl
dictionary RTCAnswerOptions : RTCOfferAnswerOptions {
};
```

### 4.3 状态定义

#### 4.3.1 `RTCSignalingState`枚举值

```webidl
enum RTCSignalingState {
    "stable",
    "have-local-offer",
    "have-remote-offer",
    "have-local-pranswer",
    "have-remote-pranswer",
    "closed"
};
```

枚举类型描述：

- stable：过程中无提供/应答的交换。这也是初始状态，本地和远程描述都是空的。
- have-local-offer：本地的`提供`类型的描述已经被成功应用了。
- have-remote-offer：远程的`提供`类型的描述已经被成功应用了。
- have-local-pranswer：远程的`提供`类型的描述已经被成功应用且本地的`对端应答`类型的描述已被成功应用。
- have-remote-pranswer：本地的`提供`类型的描述已经被成功应用且远程的`对端应答`类型的描述已被成功应用。
- closed：`RTCPeerConnection`已经被关闭，其[IsClosed]槽值变为true。

**信号状态转移图**
![Figure 1](https://www.w3.org/TR/webrtc/images/peerstates.svg)

一个状态转移的例子：
调用方的转移：

- 创建新RTCPeerConnection(): `stable`
- setLocalDescription(offer): `have-local-offer`
- setRemoteDescription(pranswer): `have-remote-pranswer`
- setRemoteDescription(answer): `stable`

被调用方的转移：

- 创建新RTCPeerConnection(): `stable`
- setRemoteDescription(offer): `have-remote-offer`
- setLocalDescription(pranswer): `have-local-pranswer`
- setLocalDescription(answer): `stable`

#### 4.3.2 `RTCIceGatheringState`枚举值

```webidl
enum RTCIceGatheringState {
    "new",
    "gathering",
    "complete"
};
```

枚举类型描述：

- new：所有`RTCIceTransports`都在"new"的收集状态，没有任何一个处于"gathering"状态，或当前还没有传输。
- gathering：所有`RTCIceTransports`都在"gathering"的收集状态
- complete：至少有一个`RTCIceTransports`存在，且都处于"completed"状态。

#### 4.3.3 RTCPeerConnectionState枚举值

```webidl
enum RTCPeerConnectionState {
    "new",
    "connecting",
    "connected",
    "disconnected",
    "failed",
    "closed"
};
```

枚举类型描述：

- new：所有`RTCIceTransport`或`RTCDtlsTranport`都在"new"状态，且没有任何一个处于"connecting"，"checking"，"failed"，"disconnected"状态，也可以是所有传输都处于"closed"状态，或当前还没有传输。
- connecting：所有`RTCIceTransport`或`RTCDtlsTranport`都在"connecting"或"checking"状态，且没有一个处于"failed"状态。
- connected：所有`RTCIceTransport`或`RTCDtlsTranport`都在"connected"，"completed"或"closed"状态，且其中至少有一个处于"connected"或"completed"状态。
- disconnected：所有`RTCIceTransport`或`RTCDtlsTranport`都在"disconnected"状态，且没有一个处于"failed"，"connecting"或"checking"状态。
- failed：所有`RTCIceTransport`或`RTCDtlsTranport`都在"failed"状态。
- closed：`RTCPeerConnection`对象的[IsClosed]槽为值true。

#### 4.3.4 `RTCIceConnectionState`枚举值

```webidl
enum RTCIceConnectionState {
    "new",
    "checking",
    "connected",
    "completed",
    "disconnected",
    "failed",
    "closed"
};
```

枚举类型描述：

- new：所有`RTCIceTransport`都在"new"状态，且没有任何一个处于"disconnected"，"checking"，"failed"，"disconnected"状态，也可以是所有`RTCIceTransports`都处于"closed"状态，或当前还没有传输。
- checking：所有`RTCIceTransport`都在"checking"状态且没任何一个处于"disconnected"或"failed"状态。
- connected：所有`RTCIceTransport`都在"connected"，"completed"或"closed"状态，且其中至少有一个处于"connected"状态。
- completed：所有`RTCIceTransport`都在"completed"或"closed"状态，且其中至少有一个处于"completed"状态。
- disconnected：所有`RTCIceTransport`都在"disconnected"状态，且没有一个处于"failed"状态。
- failed：所有`RTCIceTransport`都在"failed"状态。
- closed：`RTCPeerConnection`对象的[IsClosed]槽为值true。

值得注意的是，如果`RTCIceTransport`由于信令的存在而被丢弃（如RTCP复用或执行捆绑），或被信令创建（如增加新的媒体描述），则状态可以从某一状态跳变到另一状态。