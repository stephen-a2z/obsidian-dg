---
{"dg-publish":true,"permalink":"/Clippings/Chrome DevTools 原理、拓展与整合 · Issue 78 · Pines-Chengblog/","title":"Chrome DevTools 原理、拓展与整合 · Issue #78 · Pines-Cheng/blog","tags":["clippings"],"noteIcon":"","created":"2026-04-17T11:05:50.699+08:00","updated":"2026-04-17T11:07:23.357+08:00"}
---

## DevTools 原理

DevTools 本质上可以看成是一个前端小应用，代码在这里： [ChromeDevTools/devtools-frontend](https://github.com/ChromeDevTools/devtools-frontend)，当然，你也可以在 Chrome 浏览器直接打开：`devtools://devtools/bundled/inspector.html` 查看运行效果。

[![image](https://user-images.githubusercontent.com/9441951/76933926-3317e100-6929-11ea-9703-0bdc97569f8d.png)](https://user-images.githubusercontent.com/9441951/76933926-3317e100-6929-11ea-9703-0bdc97569f8d.png)

DevTools 是通过 [Chrome 远程调试协议（Remote Debugger Protocal)](https://chromedevtools.github.io/devtools-protocol/) 来和后端进行交互和调试的，这里说的后端一般指的是：Chrome 的远程调试功能，可以通过

```
sudo /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
```

在指定端口开启，然后在浏览器地址栏输入 `http://localhost:${port}` 能看到一个列表页面，列出了当前所有可调试的页面和插件。

[![image](https://user-images.githubusercontent.com/9441951/76934290-d23cd880-6929-11ea-9ce3-3d21476311a7.png)](https://user-images.githubusercontent.com/9441951/76934290-d23cd880-6929-11ea-9ce3-3d21476311a7.png)

点击Example Page，会导向到 `http://localhost:9222/devtools/inspector.html?ws=localhost:9222/devtools/page/55A4F84F6A66845F72388146E3B8F986`。长得和内嵌devtools 一样的 html 页面。

[![image](https://user-images.githubusercontent.com/9441951/76934282-cc46f780-6929-11ea-9279-c6155761f904.png)](https://user-images.githubusercontent.com/9441951/76934282-cc46f780-6929-11ea-9279-c6155761f904.png)

inspector.html 和 Chrome Host 之间通过 WebSocket 建立连接，这个 WebSocket 地址就是 url 中 ws参数的值。其中 `55A4F84F6A66845F72388146E3B8F986`是 `page id`，每个页面都有一个唯一的`page id`，Chrome 就是通过这个 id 确定哪个是目标页面。页面和 Chrome 内核之间就是通过这个连接交换数据的。Chrome 调试器实例和目标页面实例之间是进程通信，所以 `inspector.html` 可以通过Chrome 调试器实例加载目标页面的 Source 文件，还可以操作目标页面，例如加断点、刷新、记录Network 信息等。

通过 [Chrome 远程调试协议（Remote Debugger Protocal)](https://chromedevtools.github.io/devtools-protocol/) 建立连接之后，就会向调试后台发送很多请求来展现数据并进行交互。

[![image](https://user-images.githubusercontent.com/9441951/76934797-c7367800-692a-11ea-8fe6-485951c926c5.png)](https://user-images.githubusercontent.com/9441951/76934797-c7367800-692a-11ea-8fe6-485951c926c5.png)

此外，你还可以使用 [Chrome Remote Debugger Protocal](https://chromedevtools.github.io/devtools-protocol/)，通过 Node 与 Chrome 调试后台进行交互，并直接控制 Chrome。

### Chrome Debugging Protocol

简单来说，远程调试协议就是利用 WebSocket 建立连接 DevTools 和浏览器内核的快速数据通道。那么我们也可以自己打开这个 WebSocket，遵从它的协议来发送消息。

在前面 inspector.html 和 Chrome Host 之间通过 WebSocket 建立连接后，从整个调试过程中的 Websocket 通讯可以看出，这个接口里面有两种通讯模式：Cmmand 和 Event。

- Command：包含 request/response ，就如同一个异步调用，通过请求的信息，获取相应的返回结果。这样的通讯必然有一个message id，否则两方都无法正确的判断请求和返回的匹配状况。
```json
request: {"id":1,"method":"Page.canScreencast"}
response: {"id":1,"result":{"result":false}}
```
- Event：类似于 notification ，和第一种不同，这种模式用于由一方单方面的通知另一方某个信息。和“事件”的概念类似。
```json
{"method":"Network.loadingFinished","params：{"requestId":"14307.143","timestamp":1424097364.31611,"encodedDataLength":0}}
```

远程调试协议把操作划分为不同的域 domain ，比如

- DOM
- Debugger
- Network
- Console
- Timeline

可以理解为 DevTools 中的不同功能模块。每个域（domain）定义了它所支持的 command 和它所产生的 event（就是上面讲的两种通讯方式）。每个 command 包含 request 和 response 两部分，request 部分指定所要进行的操作以及操作说要的参数，response 部分表明操作状态，成功或失败。command 和 event 中可能涉及到非基本数据类型，在 domain 中被归为 Type，比如：'frameId': ，其中 FrameId 为非基本数据类型。

至此，不难理解： domain = command + event + type

[![image](https://user-images.githubusercontent.com/9441951/91041742-21a6d380-e643-11ea-9d2b-915c12392ad4.png)](https://user-images.githubusercontent.com/9441951/91041742-21a6d380-e643-11ea-9d2b-915c12392ad4.png)

很多工具都使用了Chrome Debugging Protocol，包括 PhantomJS，Selenium 的 ChromeDriver，本质都是一样的实现，它就相当于 Chrome 内核提供的 API 让应用调用。官网列出了很多有意思的工具：[awesome-chrome-devtools/Developing with the protocol](https://github.com/ChromeDevTools/awesome-chrome-devtools#chrome-devtools-protocol)，因为 API 丰富，所以才有了这么多的 Chrome 插件。

- [chrome-debug-protocol](https://github.com/DickvdBrink/chrome-debug-protocol) 使用了ES6和TypeScript
- [chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) 官网推荐的
- [chrome-har-capturer](https://github.com/cyrus-and/chrome-har-capturer) 传入url，直接获取har format文件

### 协议调试

使用 "Protocol Monitor"：

[![image](https://camo.githubusercontent.com/3d43e286f579ca3df56218bb737561428e3e40f794cf0b57b07a88b5eeea5021/68747470733a2f2f6368726f6d65646576746f6f6c732e6769746875622e696f2f646576746f6f6c732d70726f746f636f6c2f696d616765732f70726f746f636f6c2d6d6f6e69746f722e706e67)](https://camo.githubusercontent.com/3d43e286f579ca3df56218bb737561428e3e40f794cf0b57b07a88b5eeea5021/68747470733a2f2f6368726f6d65646576746f6f6c732e6769746875622e696f2f646576746f6f6c732d70726f746f636f6c2f696d616765732f70726f746f636f6c2d6d6f6e69746f722e706e67)

- DevTools-on-DevTools

打开 [devTools-on-devTools](https://stackoverflow.com/a/12291163/89484)，然后在内部 DevTools 窗口中，使用 Main。控制台中的 `MainImpl.sendOverProtocol() `:

```js
let Main = await import('./main/main.js');
await Main.MainImpl.sendOverProtocol('Emulation.setDeviceMetricsOverride', {
  mobile: true,
  width: 412,
  height: 732,
  deviceScaleFactor: 2.625,
});

const data = await Main.MainImpl.sendOverProtocol("Page.captureScreenshot");
```

## DevTools 拓展与整合

Chrome DevTools 本身就具备很好的拓展性。如果 DevTools 缺少一个你需要的特性，你可以找一找现成的扩展（extension），或者干脆自己写一个，同时你也可以选择将 DevTools 功能集成到你的应用中。

使用 DevTools 构建自定义解决方案有两种基本方式：

- DevTools Extension：一个插入到 DevTools 中的 [Chrome extension](http://developer.chrome.com/extensions/)，可以增加功能和扩展用户界面。
- Debugging Protocol Client：使用 [Chrome remote debugging protocol](https://developer.chrome.com/devtools/docs/debugger-protocol.html) 插入 Chrome 的底层调试支持的第三方应用程序。

下面的小节将讨论这两种方法。

### DevTools Chrome extensions

DevTools UI 是一个嵌入在 Chrome 中的 web 应用程序。 Devtools 扩展使用 [Chrome extensions system](http://developer.chrome.com/extensions/) 为 DevTools 添加功能。DevTools 扩展可以向 DevTools 添加新的面板（panels），向 Elements 和 Sources 面板侧边栏（panel sidebar）添加新的窗格（panes），检查 resources 和 network 事件，以及在被 inspected 的浏览器选项卡（tab）中执行 JavaScript 表达式。

如果你想开发一个 DevTools 扩展：

- 如果你以前没有开发过 Chrome 扩展，请看 [Overview of Chrome Extensions](http://developer.chrome.com/extensions/overview.html) 。

[![image](https://camo.githubusercontent.com/1558f152263fa4a0e6b989de40ede80e7570b773a4d525334049bc1ee907bef8/68747470733a2f2f646576656c6f7065722e6368726f6d652e636f6d2f7374617469632f696d616765732f6f766572766965772f6d6573736167696e676172632e706e67)](https://camo.githubusercontent.com/1558f152263fa4a0e6b989de40ede80e7570b773a4d525334049bc1ee907bef8/68747470733a2f2f646576656c6f7065722e6368726f6d652e636f6d2f7374617469632f696d616765732f6f766572766965772f6d6573736167696e676172632e706e67)

- 通过 [Extending DevTools](http://developer.chrome.com/extensions/devtools.html) 查看创建一个 Chrome DevTools 扩展的细节。

[![image](https://camo.githubusercontent.com/3abf3c440f6354d599c6662dead81527a7c36a9422ca7052de111043d7bffa50/68747470733a2f2f646576656c6f7065722e6368726f6d652e636f6d2f7374617469632f696d616765732f646576746f6f6c732d657874656e73696f6e2e706e67)](https://camo.githubusercontent.com/3abf3c440f6354d599c6662dead81527a7c36a9422ca7052de111043d7bffa50/68747470733a2f2f646576656c6f7065722e6368726f6d652e636f6d2f7374617469632f696d616765732f646576746f6f6c732d657874656e73696f6e2e706e67)

有关 DevTools 扩展的实例列表，请参考 [Sample DevTools Extensions](https://developer.chrome.com/devtools/docs/sample-extensions.md)。 这些示例包括许多可供参考的 Extensions 源码。

### Debugging protocol clients

第三方应用程序，如 IDE、编辑器、持续集成框架和测试框架都可以与 Chrome 调试器集成，以调试代码、实时预览代码和 CSS 更改，并控制浏览器。 客户端使用 [Chrome debugging protocol](https://developer.chrome.com/devtools/docs/debugger-protocol.html) 与 Chrome 实例进行交互，该实例既可以在同一个系统上运行，也可以远程运行。

注意：目前，Chrome debugging protocol 每个 Page 只支持一个客户端。 因此，您可以使用 DevTools inspect 页面，或者使用第三方客户端，但两者不能同时 inspect。

有两种方法可以与调试协议集成：

- 运行在 Chrome 中的应用程序（比如基于 Web 的 IDE）可以使用调试器模块 [chrome.debugger](http://developer.chrome.com/extensions/debugger.html) 创建 Chrome 扩展，此模块允许扩展与调试器直接交互，绕过 DevTools 用户界面。参见：[Using the debugger extension API](https://developer.chrome.com/devtools/docs/debugger-protocol.html#extension)
- 其他应用程序可以使用 [wire protocol](https://developer.chrome.com/devtools/docs/debugger-protocol.html#remote) 和 debugger 直接交互，该协议需要通过 WebSocket 连接交换 JSON 消息。

[![image](https://camo.githubusercontent.com/3d43e286f579ca3df56218bb737561428e3e40f794cf0b57b07a88b5eeea5021/68747470733a2f2f6368726f6d65646576746f6f6c732e6769746875622e696f2f646576746f6f6c732d70726f746f636f6c2f696d616765732f70726f746f636f6c2d6d6f6e69746f722e706e67)](https://camo.githubusercontent.com/3d43e286f579ca3df56218bb737561428e3e40f794cf0b57b07a88b5eeea5021/68747470733a2f2f6368726f6d65646576746f6f6c732e6769746875622e696f2f646576746f6f6c732d70726f746f636f6c2f696d616765732f70726f746f636f6c2d6d6f6e69746f722e706e67)

有关一些集成示例，请参考：[Sample Debugging Protocol Clients](https://developer.chrome.com/devtools/docs/debugging-clients.md)

## 参考

- [Integrating with DevTools](https://developer.chrome.com/devtools/docs/integrating)
- [ChromeDevTools/devtools-frontend](https://github.com/ChromeDevTools/devtools-frontend)
- [Chrome Remote Debugger Protocal](https://chromedevtools.github.io/devtools-protocol/)
- [cyrus-and/chrome-remote-interface](https://github.com/cyrus-and/chrome-remote-interface) The most-used JavaScript API for the protocol
- [ChromeDevTools/awesome-chrome-devtools](https://github.com/ChromeDevTools/awesome-chrome-devtools)
- [深入理解 Chrome DevTools](https://zhaomenghuan.js.org/blog/chrome-devtools.html#%E5%89%8D%E8%A8%80)

---

## Comments

> **Pines-Cheng** · 2020-03-23
> 
> 主要内容：
> 
> 1. 内容脚本（content scripts）和 扩展（extension）
> 2. 跨插件通信（chrome.runtime.sendMessage(laserExtensionId，）
> 3. 从网页发送信息（ chrome.runtime.sendMessage(editorExtensionId,）
> 4. Native messaging
> 
> ## Message Passing
> 
> 由于内容脚本（content scripts）运行在 web 页面的上下文中，而不是在扩展（extension）中，因此它们通常需要某种方式与扩展（extension）的其余部分进行通信。 例如，RSS 阅读器扩展可能使用内容脚本（content scripts）检测页面上是否存在 RSS feed，然后通知后台页面以显示该页面的页面操作图标。
> 
> 扩展（extension）和它们的内容脚本（content scripts）之间的通信是通过消息传递（message passing）进行的。 任何一方都可以 监听（listen） 从另一端发送的消息，并在同一个通道（channel）上作出响应。 消息可以包含任何有效的 JSON 对象（null、boolean、number、string、array 或 object）。 有一个用于 [一次性请求（one-time requests）](https://developer.chrome.com/extensions/messaging#simple)的简单 API ，还有一个更复杂的 API，它们允许你使用 [长连接（long-lived connections)](https://developer.chrome.com/extensions/messaging#connect) 通过共享上下文（shared context）交换（exchanging）多个消息。 如果您知道另一个扩展的 ID，也可以将消息发送到该扩展，这在 [跨插件消息（cross-extension messages）](https://developer.chrome.com/extensions/messaging#external)消息部分中有介绍。
> 
> 另外还有 [Sending messages from web pages](https://developer.chrome.com/extensions/messaging#external-webpage) 以及 [Native messaging](https://developer.chrome.com/extensions/messaging#native-messaging)。
> 
> ### [Simple one-time requests](https://developer.chrome.com/extensions/messaging#simple)
> 
> 如果您只需要向扩展的另一部分发送一条消息(并且可以选择获得回复) ，那么您应该使用简化的 `runtime.sendMessage` 或 `tabs.sendMessage`。 这允许您分别将一次性的 `JSON-serializable` 消息从内容脚本（content script）发送到扩展，或者反之亦然。 可选的回调参数允许您处理来自另一端的响应(如果有的话)。
> 
> 从内容脚本发送请求如下：
> 
> ```
> chrome.runtime.sendMessage({greeting: "hello"}, function(response) {
>   console.log(response.farewell);
> });
> ```
> 
> 从扩展向内容脚本发送请求看起来非常相似。
> 
> ```js
> chrome.tabs.query({active: true, currentWindow: true}, function(tabs) {
>   chrome.tabs.sendMessage(tabs[0].id, {greeting: "hello"}, function(response) {
>     console.log(response.farewell);
>   });
> });
> ```
> 
> ### [Long-lived connections](https://developer.chrome.com/extensions/messaging#connect)
> 
> 有时候，比起单一的请求和响应，有一个持续时间更长的对话是有用的。 在这种情况下，您可以分别使用 `runtime.connect` 或 `tabs.connect` 打开从内容脚本（content script）到扩展页面的 long-lived channel，反之亦然。 通道可以有选择地有一个名称，允许您区分不同类型的连接。
> 
> 下面是如何从内容脚本中打开一个频道，并发送和监听消息：
> 
> ```js
> var port = chrome.runtime.connect({name: "knockknock"});
> port.postMessage({joke: "Knock knock"});
> port.onMessage.addListener(function(msg) {
>   if (msg.question == "Who's there?")
>     port.postMessage({answer: "Madame"});
>   else if (msg.question == "Madame who?")
>     port.postMessage({answer: "Madame... Bovary"});
> });
> ```
> 
> 为了处理传入的连接，需要设置 `runtime.onConnect` 事件侦听器。 从内容脚本或扩展页面来看，这看起来是一样的。
> 
> ```js
> chrome.runtime.onConnect.addListener(function(port) {
>   console.assert(port.name == "knockknock");
>   port.onMessage.addListener(function(msg) {
>     if (msg.joke == "Knock knock")
>       port.postMessage({question: "Who's there?"});
>     else if (msg.answer == "Madame")
>       port.postMessage({question: "Madame who?"});
>     else if (msg.answer == "Madame... Bovary")
>       port.postMessage({question: "I don't get it."});
>   });
> });
> ```
> 
> 需要注意 Port lifetime。
> 
> ### [Cross-extension messaging](https://developer.chrome.com/extensions/messaging#external)
> 
> 除了在扩展（extension）中的不同组件之间发送消息外，还可以使用消息传递 API 与其他扩展进行通信。 这使您可以公开其他扩展可以利用的公共 API。
> 
> 侦听传入的请求和连接与内部情况类似，只是使用 `runtime.onMessageExternal` 或 `runtime.onConnectExternal` 方法。 下面是每种方法的一个例子:
> 
> ```js
> // For simple requests:
> chrome.runtime.onMessageExternal.addListener(
>   function(request, sender, sendResponse) {
>     if (sender.id == blocklistedExtension)
>       return;  // don't allow this extension access
>     else if (request.getTargetData)
>       sendResponse({targetData: targetData});
>     else if (request.activateLasers) {
>       var success = activateLasers();
>       sendResponse({activateLasers: success});
>     }
>   });
> 
> // For long-lived connections:
> chrome.runtime.onConnectExternal.addListener(function(port) {
>   port.onMessage.addListener(function(msg) {
>     // See other examples for sample onMessage handlers.
>   });
> });
> ```
> 
> 同样，向另一个扩展发送消息类似于在扩展中发送消息。 唯一的区别是，您必须传递要与之通信的扩展的 ID。 例如：
> 
> ```js
> // The ID of the extension we want to talk to.
> var laserExtensionId = "abcdefghijklmnoabcdefhijklmnoabc";
> 
> // Make a simple request:
> chrome.runtime.sendMessage(laserExtensionId, {getTargetData: true},
>   function(response) {
>     if (targetInRange(response.targetData))
>       chrome.runtime.sendMessage(laserExtensionId, {activateLasers: true});
>   });
> 
> // Start a long-running conversation:
> var port = chrome.runtime.connect(laserExtensionId);
> port.postMessage(...);
> ```
> 
> ### [Sending messages from web pages](https://developer.chrome.com/extensions/messaging#external-webpage)
> 
> 与跨扩展消息传递（cross-extension messaging,）类似，你的应用程序或扩展程序可以接收和响应来自常规网页的消息。 要使用这个功能，你必须首先在你的 manifest.json 中指定你想要与哪些网站通信。 例如:
> 
> ```json
> "externally_connectable": {
>   "matches": ["*://*.example.com/*"]
> }
> ```
> 
> 这将向任何与你指定的 URL 模式匹配（patterns matches）的页面公开消息传递 API。 URL 模式（patterns）必须至少包含一个二级域（second-level domain），即主机名模式（hostname patterns），如 `“ * ”、“ * .com”、“ *.co.uk” 和 “ *.appspot.com” `是禁止的。 在网页上，使用 `runtime.sendMessage` 或 `runtime.connect` API 向特定的应用程序或扩展发送消息。 例如：
> 
> ```js
> // The ID of the extension we want to talk to.
> var editorExtensionId = "abcdefghijklmnoabcdefhijklmnoabc";
> 
> // Make a simple request:
> chrome.runtime.sendMessage(editorExtensionId, {openUrlInEditor: url},
>   function(response) {
>     if (!response.success)
>       handleError(url);
>   });
> ```
> 
> 从您的应用程序（app）或扩展（extension），您可以通过 `runtime.onMessageExternal` 或 `runtime.onConnectExternal` API 监听来自网页的消息，类似于跨扩展消息传递（cross-extension messaging）。 只有网页可以启动连接。 下面是一个例子:
> 
> ```js
> chrome.runtime.onMessageExternal.addListener(
>   function(request, sender, sendResponse) {
>     if (sender.url == blocklistedWebsite)
>       return;  // don't allow this web page access
>     if (request.openUrlInEditor)
>       openUrl(request.openUrlInEditor);
>   });
> ```
> 
> ### [Native messaging](https://developer.chrome.com/extensions/messaging#native-messaging)
> 
> 扩展和应用程序可以与注册为 \[原生消息主机（native messaging host）\] ([https://developer.chrome.com/extensions/nativeMessaging#native-messaging-host)的](https://developer.chrome.com/extensions/nativeMessaging#native-messaging-host\)%E7%9A%84) 原生应用（native applications） [交换消息 exchange messages](https://developer.chrome.com/extensions/nativeMessaging#native-messaging-client) 。 要了解关于此特性的更多信息，请参见 [Native messaging](https://developer.chrome.com/extensions/nativeMessaging)。
> 
> ### [Security considerations](https://developer.chrome.com/extensions/messaging#security-considerations)
> 
> - Content scripts are less trustworthy
> - Cross-site scripting
> 
> ### [Examples](https://developer.chrome.com/extensions/messaging#examples)
> 
> 你可以在 [examples/api/messaging](https://chromium.googlesource.com/chromium/src/+/master/chrome/common/extensions/docs/examples/api/messaging/) 目录中找到通过消息进行通信的简单示例。 [native messaging sample](https://developer.chrome.com/extensions/nativeMessaging#examples)演示了 Chrome 应用程序如何与本地应用程序通信。 有关更多示例和查看源代码的帮助，请参见 [示例](https://developer.chrome.com/extensions/samples)。
> 
> ## 参考
> 
> - [Message Passing](https://developer.chrome.com/extensions/messaging)

