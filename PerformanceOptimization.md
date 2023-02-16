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

### 业务实现层面的优化

## 与原生配合的优化

### 资源预拉取

### 资源缓存

## 与后端配合的优化

### HTTP缓存

### CDN缓存

### SSG、SSR

SSR即服务端渲染，以前做技术调研时写过[一篇小结](https://www.everseenflash.com/CS/Frontend/SSR%20Practise.md#Hdb33a7d5a16e1d97)。SSG指静态站点生成，适用于比较静态的产品介绍、技术文档等场景，像Vuepress等都可以归类为SSG框架。

除了SSR和SSG之外还有一些名词如ISR、DPR等，都是将各种渲染、缓存方式结合起来产生的模式。个人观点是无需强行记忆，而是由业务需要驱动，例如SSR渲染的页面偏静态但确实和用户状态绑定，很容易想到利用Redis做内存缓存；小团队不好应对服务端渲染这种CPU密集型架构，则介于完全静态的SSG和完全动态的SSR之间的“实时预渲染”便应运而生，渲染服务不对客，而是发布到CDN，ISR的概念便隐含其中了。
