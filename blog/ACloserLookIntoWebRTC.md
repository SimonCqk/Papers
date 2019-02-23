# WebRTC更进一步

我们在最近的一篇WebKit博客中宣布了对High Sierra平台和iOS中Safari的WebRTC支持。现在，我们希望能够带领大家深入实现中的一些细节，并且对您网站中的WebRTC应用带来一些建议。

一个应用了WebRTC和网页摄像头的网站可以获取并传播一些非常私人的信息。用户必须显式地对网站进行授权，并假设他们的图片和声音会被合理使用。WebKit为了利用技术保护用户的隐私权，要求网站满足一些特定的条件。除此之外，当用户的摄像头设备被使用时，Safari会提醒用户，并且用户可以控制网站对摄像头设备的访问权限。对于在app中使用了WebKit的开发者发来说，`RTCPeerConnection`和`RTCDataChannel`可以在任意网页视图中使用，但对摄像头和麦克风的访问目前受限于Safari。

## 开发菜单

[Safari技术预览版34](https://webkit.org/blog/7760/release-notes-for-safari-technology-preview-34/)向开发者展示了一些选项，可以更容易的测试他们的WebRTC网站或将Safari集成到持续集成系统中，入口在**开发>WebRTC**子菜单中。

![f1](https://webkit.org/wp-content/uploads/webrtc-menu.png)

我们将介绍每个选项，并将在下方解释它们是如何帮助你进行开发的。

此外，WebKit将WebRTC的状态作为日志记录在了系统日志中，包括SDP邀请和应答，ICE候选项，WebRCT状态统计，传入及传出视频帧计数器等。

## 网页摄像头的安全源策略

希望访问捕获设备的网站需要满足两个约束。

首先，请求需要使用摄像头和麦克风的文档需要来自HTTPS域。由于在本地开发和测试时可能会很麻烦，因此可以通过在**开发>WebRTC***菜单中选中"在不安全的站点上允许使用网页摄像头"来旁路HTTPS限制。

其次，当子帧请求使用摄像头设备时，通向主帧的帧链需要来自相同的安全源。用户可能无法识别与主帧相关的子帧的第三方来源，因此该约束避免了混淆用户授予其访问权限的用户。

## 模拟网页摄像头

在**开发>WebRTC**菜单中，可以选择"使用模拟摄像头设备"来替代真实的摄像头设备。如下所示，模拟循环一个bip-bop AV流。当用作传入流时，模拟的可预测数据可以轻松评估流媒体播放的各个方面，包括同步，延迟和输入设备的选择。

![f2](https://webkit.org/wp-content/uploads/bip-bop.png)

模拟对于在持续集成系统中运行自动化测试也很有用。如果您正在使用并且想要避免来自`getUserMedia`的提示，可以通过Safari**首选项...>网站**面板将网站的摄像头和麦克风策略设置为允许。

## ICE候选项限制

ICE候选项在WebRTC连接的早期阶段进行信息交换，以识别两个对等连接之间的所有可能的网络路径。为此，WebKit必须将每个对等连接的ICE候选项暴露给网站并共享它们。ICE候选项公开IP地址，特别是那些可以用于跟踪的主机IP地址。

但是，在许多网络拓扑中，主机ICE候选项不必进行连接。无论是否用于交换视频或任意数据，Server Reflexive和TURN ICE候选项通常都足以确保连接。如果不访问摄像头设备，WebKit只会公开Server Reflexive和TURN ICE候选项，这些候选者会公开可能已经被网站收集的IP。授予访问权限后，WebKit将公开主机ICE候选项，从而最大限度地提高连接成功的可能性。我们做出此例外的原因是：我们认为用户通过授予对其摄像流的访问权限来表达对网站的高度信任。

一些测试页面可能会对主机ICE候选项的可用性作出假设。要对此进行测试，请从**开发>WebRTC**菜单中启用"禁用ICE候选限制"，然后重新加载页面。

## 旧版WebRTC及媒体流API

随着WebRTC标准化过程的推进，`RTCPeerConnection` API以各种方式逐步改进。API从最初基于回调，到变为完全基于promise，从最初专注于将`MediaStream`，移动到专注于`MediaStreamTrack`。感谢[WebRTC in WebKit](http://www.webrtcinwebkit.org/blog/2016/11/9/openwebrtc-in-webkit-upstream-complete)团队的努力，`RTCPeerConnection` API的改进与这两个主要变化保持一致。

我们已经在Safari技术预览版34上默认关闭了旧版WebRTC API，并计划在没有这些API的情况下在macOS High Sierra和iOS 11上发布Safari 11。保留遗留API限制了我们在WebRTC上更快推进的能力。任何希望为Safari提供支持的网站都可能需要进行其他调整，因此这是摆脱这些遗留API的好时机。现有网站仍然可以依赖这些遗留API，您可以通过在**开发>WebRTC**菜单中启用"启用旧版WebRTC API"来达到目的。

更准确地说，以下API仅在打开旧版API开关时可用，并提供了有关如何更新的建议：

```cs
partial interface Navigator {
    // Switch to navigator.mediaDevices.getUserMedia
    void getUserMedia(MediaStreamConstraints constraints, NavigatorUserMediaSuccessCallback successCallback, NavigatorUserMediaErrorCallback errorCallback);
};

partial interface RTCPeerConnection {
    // Switch to getSenders, and look at RTCRtpSender.track
    sequence<MediaStream> getLocalStreams();
    // Switch to getReceivers, and look at RTCRtpReceiver.track
    sequence<MediaStream> getRemoteStreams();

    // Switch to getSenders/getReceivers
    MediaStream getStreamById(DOMString streamId);
    // Switch to addTrack
    void addStream(MediaStream stream);
    // Switch to removeTrack
    void removeStream(MediaStream stream);

    // Listen to ontrack event
    attribute EventHandler onaddstream;

    // Update to promise-only version of createOffer
    Promise<void> createOffer(RTCSessionDescriptionCallback successCallback, RTCPeerConnectionErrorCallback failureCallback, optional RTCOfferOptions options);
    // Update to promise-only version of setLocalDescription
    Promise<void> setLocalDescription(RTCSessionDescriptionInit description, VoidFunction successCallback, RTCPeerConnectionErrorCallback failureCallback);
    // Update to promise-only version of createAnswer
    Promise<void> createAnswer(RTCSessionDescriptionCallback successCallback, RTCPeerConnectionErrorCallback failureCallback);
    // Update to promise-only version of setRemoteDescription
    Promise<void> setRemoteDescription(RTCSessionDescriptionInit description, VoidFunction successCallback, RTCPeerConnectionErrorCallback failureCallback);
    // Update to promise-only version of addIceCandidate
    Promise<void> addIceCandidate((RTCIceCandidateInit or RTCIceCandidate) candidate, VoidFunction successCallback, RTCPeerConnectionErrorCallback failureCallback);
};
```

许多站点通过开源的adapter.js项目支持polyfill API。更新到最新版本是弥补API差异的一种方法，但我们建议切换到规范中列出的API。

以下是如何使用最新API的几个示例。典型的仅接收/在线研讨的WebRTC调用可以像这样完成：

```js
var pc = new RTCPeerConnection();
pc.addTransceiver('audio');
pc.addTransceiver('video');
var offer = await pc.createOffer();
await pc.setLocalDescription(offer);
// send offer to the other party
...
```

典型的音频-视频WebRTC调用也可以像以下方式完成：

```js
var stream = await navigator.mediaDevices.getUserMedia({audio: true, video: true});
var pc = new RTCPeerConnection();
var audioSender = pc.addTrack(stream.getAudioTracks()[0], stream);
var videoSender = pc.addTrack(stream.getVideoTracks()[0], stream);
var offer = await pc.createOffer();
await pc.setLocalDescription(offer);
// send offer to the other party
...
```

基于`MediaStreamTrack`的API很有意义，因为大多数处理都是在这一层完成的。假设捕获视频轨的640×480默认分辨率不够好，继续前面的示例，动态更改它可以按如下方式完成：

```js
videoSender.track.applyConstraints({width: 1280, height: 720});
```

或者我们可能想要视频静音但保持音频流动：

```js
videoSender.track.enabled = false;
```

等等，我们实际上想要对当前的视频轨应用一些很酷的滤镜效果，就像在这个[例子](https://webkit.org/blog-files/webrtc/pc-with-effects/index.html)中一样，所需要的只是一些不需要任何SDP重新协商的函数调用：

```js
videoSender.track.enabled = true;
renderWithEffects(video, canvas);
videoSender.replaceTrack(canvas.captureStream().getVideoTracks()[0]);
```