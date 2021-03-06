1. 浏览器区别这是链接还是搜索关键字
2. 获取 URL 指向目的主机的 IP 地址，DNS解析
3. 建立 TCP 连接， 三次握手
4. 如果是 https，还要进行 https 握手的过程
5. 构造 http 请求，发送到服务器
6. 拿到结果后，浏览器解析 html 内容，展示到屏幕





## 导航流程

![url页面展示流程](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5Curl%E9%A1%B5%E9%9D%A2%E5%B1%95%E7%A4%BA%E6%B5%81%E7%A8%8B.png)

首先我们将用户发出 URL 请求到页面开始解析的这个过程叫做**导航流程**，这个阶段我们可以分为四步：

### 1. 处理用户输入

* 如果用户输入的是搜索内容，搜索引擎会将它合成为一个包含搜索关键字的 URL
* 如果用户输入的是一个 URL 地址，浏览器还会根据协议把它包装为一个完整的 URL，如 `github.com` -> `https://github.com`

### 2. URL 请求过程

浏览器进程会通过 `IPC` 将最终的 URL 发送给网络进程，网络进程会先查询本地缓存，如果有缓存则直接将资源返回给浏览器进程，如果没有就开始网络请求的流程。就是常规的 DNS 查询，建立 TCP 连接，如果是 https 还需要进行 TLS 连接。连接建立后，浏览器生成相应的请求行、请求头等请求信息发送给服务器，服务器生成相应的响应数据返回给网络进程。

网络进程在拿到响应数据后，还要做两件事情：

1. 解析响应头，如果状态码是 301/302 还需要读取响应头的 `location` 字段重新发起一次 http 请求
2. 响应数据类型处理。浏览器会根据 `Content-Type` 字段的值来决定如何处理响应的数据。如果响应数据是一个下载资源，比如字段值为 `application/octect-stream`，那么该请求会被交给浏览器的下载管理器，同时 URL 请求导航流程就此结束

### 3. 准备渲染进程

通常来说，chrome 会为每个页面都分配一个渲染进程，但如何两个页面是属于**同一站点** (same-site，指协议和域名相同)，那么这两个页面会属于同一个渲染进程

### 4. 提交文档

网络进程接受完数据后，浏览器进程会发出一个 "提交文档" 的消息，渲染进程接收到这个消息就会与网络进程建立传输数据的“管道”，数据传输完成会返回"确认提交"消息给浏览器进程，浏览器收到消息会更新浏览器界面状态



## 渲染流程

![渲染流程](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E6%B8%B2%E6%9F%93%E6%B5%81%E7%A8%8B.png)

渲染流程指浏览器如何将 `HTML, CSS, JavaScript` 转化为页面，这整个过程会有很多个子阶段，我们一个一个的来看

### 1. 构建 DOM 树

![dom树](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5Cdom%E6%A0%91.png)

第一阶段是将 HTML 转化为浏览器能够理解的结构--DOM 树。DOM 是保存在内存中的树状结构，可以通过 JS 来查询或修改其内容，一般浏览器的 `document` 对象中就包含了整个 DOM 树

### 2. 样式计算

样式计算的目的是计算出 DOM 节点中每个元素的具体样式，有三个步骤：

1. 将 CSS 文本转化为浏览器能理解的结构，`doucment.styleSheets`，这是一个类数组的对象
2. 标准化样式表中的属性值，如将 `2em` 解析为 `32px`，`blue` 解析为 `rgb(0, 0, 255)`
3. 计算出每个元素的具体样式，主要涉及到 CSS 中的继承和层叠

### 3. 布局

![生成布局树](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E7%94%9F%E6%88%90%E5%B8%83%E5%B1%80%E6%A0%91.png)

布局目的是计算出 DOM 中可见元素的几何位置，有两步：

1. 生成布局树
2. 计算布局树节点的坐标位置

### 4. 分层

![图层树](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E5%9B%BE%E5%B1%82%E6%A0%91.png)

渲染引擎需要为特定的节点生成专用的图层，并生成一颗对应的图层树。并不是布局树中的每个节点都包含一个图层，如果一个节点没有对应的图层，那么它就从属于父节点的图层。有两种情况节点会被提升为单一的图层：

1. 拥有层叠上下文(使用了定位属性，opacity, filter) 等元素
2. 需要裁剪的地方也会被创建为图层。比如将文本放在 200*200 的区域内，放不下就会产生裁剪

### 5. 图层绘制

完成图层树的构建后，渲染引擎会对图层树中的每个图层进行绘制。一个图层的绘制会被拆分为很多个小的绘制指令，这些绘制指令按顺序组成一个待绘制列表

### 6. 栅格化 (raster)

![GPU栅格化](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5CGPU%E6%A0%85%E6%A0%BC%E5%8C%96.png)

绘制列表只是用来记录绘制顺序和绘制指令的列表，而实际上绘制操作是由渲染引擎中的合成线程来完成的。当图层的绘制列表准备好后，主线程会把绘制列表提交给合成线程。有时候一个图层很大，可能远超视口 (viewport) 的尺寸，所以我们没有必要一次性绘制出图层的全部内容。合成线程会将图层划分为图块 (tile) ，大小通常是 `256*256` 或 `512*512`。**合成线程会按照视口附近的图块来优先生成位图，实际生成位图的操作是由栅格化来执行的。所谓栅格化，是指将图块转换为位图**。渲染进程维护了一个栅格化的线程池，所有的图块栅格化都是在线程池内执行的。如果栅格化操作使用了 GPU进行加速，那么最终生成位图的操作是在 GPU 中完成的。

### 7. 合成与显示

一旦所有图块都被栅格化，合成线程就会生成一个绘制图块的命令 `DrawQuad` 并将其发送给浏览器进程。浏览器进程里面有一个叫 viz 的组件，用来接收合成线程发过来的 DrawQuad 命令，然后根据 DrawQuad 命令，将其页面内容绘制到内存中，最后再将内存显示在屏幕上。



## 相关概念

### 重排

![重排](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E9%87%8D%E6%8E%92.png)

如果更新了元素的几何属性，我们就要进行重排，比如修改了元素的宽高。**重排会更新完整的渲染流水线，开销最大**

### 重绘

![重绘](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E9%87%8D%E7%BB%98.png)

如果只是更新了元素的绘制属性，我们则需要重绘，比如修改了元素的背景颜色。相较于重排操作，**重绘省去了布局和分层阶段，所以执行效率会比重排操作要高一些。**

### 合成

![合成](F:%5Cdesktop%5CDaily_Notes%5Cpublic%5C%E5%90%88%E6%88%90.png)

如果更改一个既不要布局也不要绘制的属性，比如 `transform` 动画。染引擎将跳过布局和绘制，只执行后续的合成操作。这样的效率是最高的，因为是在非主线程上合成，并没有占用主线程的资源，另外也避开了布局和绘制两个子阶段，所以**相对于重绘和重排，合成能大大提升绘制效率**。

