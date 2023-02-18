# 前端性能优化

## 量化设施的搭建

### 运行时——性能采集

#### 常用性能指标

1. [FCP](https://web.dev/fcp)：从页面开始加载到页面内容的任何部分在屏幕上完成渲染的时间。在`PerformanceObserver`中侦听名称为`first-contentful-paint`的`paint`条目；

2. [LCP](https://web.dev/lcp)：从页面开始加载到最大文本块或图像元素在屏幕上完成渲染的时间。在`PerformanceObserver`中侦听`largest-contentful-paint`条目；

3. [FID](https://web.dev/fid)：从用户首次交互到浏览器实际能够开始处理事件回调的时间。在`PerformanceObserver`中侦听`first-input`条目。

4. [TTI](https://web.dev/tti)：从页面开始加载到主要子资源完成渲染并能够流畅响应用户输入的时间。这里的“主要”、“流畅”都比较抽象，指标的计算也挺迷的，具体来说就是找到一个是5s的安静窗口，即5s内没有长任务（阻塞主线程的任务，可以用`PerformanceObserver`侦听`longtask`条目）且不超过两个正在处理的网络GET请求，通俗的理解就是浏览器这段时间已经闲下来了。在这个5s的安静窗口内反向搜索，TTI的耗时记为最后一个长任务的结束时间，如果没有长任务，则等于FCP。

5. [TBT](https://web.dev/tbt/)：FCP 与 TTI 之间的总时间，代表了主线程被阻塞无法作出输入响应的时长。

6. [CLS](https://web.dev/cls)：浏览器内部会将两帧之间起始位置发生了变更的起始元素判定为不稳定元素，新增元素或者现有元素更改大小则不算作布局偏移。布局偏移分数等于影响分数*距离分数，前者指两帧合起来看，不稳定元素占据可视区域的百分比，后者指不稳定元素的位移百分比。 在`PerformanceObserver`中侦听`layout-shift`条目。

### 运行时——异常体系

#### Source Map

### 运行时——上报

### 静态编译——依赖分析

## 前端自身的优化

### Code Split和按需加载

### Tree Shaking

### Sprite Image、图片压缩

### Prerender

### 减少请求数

## 与原生配合的优化

### 资源预拉取

### 资源缓存

## 与后端配合的优化

### HTTP缓存

1. Expires：使用确定的时间而非通过指定经过的时间来确定缓存的生命周期。由于时间格式难以解析，以及客户端服务端的时钟差异，目前更多的是使用`Cache-Control`的`max-age`来指定经过的时间。

2. Cache-Control

    1. `max-age`：以秒为单位，代表缓存在什么时间后会失效，从服务端创建报文算起；
    2. `no-store`：不要将响应缓存在任何存储中；
    3. `no-cache`：允许缓存，但无论缓存过期与否，使用缓存前必须去服务端验证，一般会配合ETag和Last-Modified使用：如果请求的资源已更新，客户端将受到`200 OK`响应，否则，服务端应返回`304 Not Modified`；
    4. `must-revalidate`：允许缓存，在缓存过期的情况下必须去服务端验证，看看服务端资源是否有变化，因此一般与`max-age`配合使用；
    5. `public`：响应可以被任何缓存区缓存，包括共享缓存区；
    6. `private`：缓存内容不可与其他用户共享，一般用于有cookie的个性化场景；
    7. `immutable`：资源不会变，无需验证；
    8. `proxy-revalidate`：过时的控制代理缓存的技术。

3. If-Modified-Since

    客户端缓存过期了也未必就要立即丢弃，很可能服务端资源没变，这时可以简单刷新下客户端缓存的生命周期而不必重新传输数据，即所谓的“重新验证（revalidate）”。具体实现方式是在请求时带上`If-Modified-Since`及之前缓存下来的`Last-Modified`时间，服务端将返回200或304指示是否需要刷新。

    由于`If-Modified-Since`和`Last-Modified`都是时间戳格式，与Expires存在类似问题，因此有`ETag/If-None-Match`方案。

4. ETag/If-None-Match

    ETag是服务端生成的任意值，通常是文件Hash或者版本号等。客户端缓存过期时，请求会带上`If-None-Match`及缓存的`ETag`值去询问服务端是否改变了，如果服务端为该资源确定的最新`ETag`与客户端发来的一致，说明无需重新发送资源，应返回`304`，客户端刷新缓存生命周期并复用即可。

5. Vary：区分响应的基本方式是利用它们的URL，但有些情况下同一个URL的响应也可能不同，例如根据`Accept-Language`返回了不同的语言版本，这时可利用`Vary`指明额外用于区分响应的字段。

#### 常见用例

1. 强制重新验证：`no-cache`或`max-age=0, must-revalidate`

2. 不使用缓存：`no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate`

3. 长期缓存且避免重新验证：`max-age=31536000, immutable`

### SSG、SSR

SSR即服务端渲染，以前做技术调研时写过[一篇小结](https://www.everseenflash.com/CS/Frontend/SSR%20Practise.md#Hdb33a7d5a16e1d97)。SSG指静态站点生成，适用于比较静态的产品介绍、技术文档等场景，像Vuepress等都可以归类为SSG框架。

除了SSR和SSG之外还有一些名词如ISR、DPR等，都是将各种渲染、缓存方式结合起来产生的模式。个人观点是无需强行记忆，而是由业务需要驱动，例如SSR渲染的页面偏静态但确实和用户状态绑定，很容易想到利用Redis做内存缓存；小团队不好应对服务端渲染这种CPU密集型架构，则介于完全静态的SSG和完全动态的SSR之间的“实时预渲染”便应运而生，渲染服务不对客，而是发布到CDN，ISR的概念便隐含其中了。
