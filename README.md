

##   前端页面监控





###  性能指标监控

https://w3c.github.io/navigation-timing/

https://juejin.cn/post/6844904182202253325

Performance API有三个api组成：

- Resource Timing API：与网页资源（脚本、样式、图片等）加载相关的耗时信息，定义了接口 PerformanceResourceTiming。
- Navigation Timing API：从页面导航开始一直到 load 事件结束，中间经历过程的耗时信息。定义了接口 PerformanceNavigationTiming，此接口继承自 PerformanceResourceTiming 接口。
- Paint Timing：与网页绘制相关的耗时信息。定义了接口 PerformancePaintTiming。

上面三个接口，又都继承自Performance Timeline API中的 PerformanceEntry 接口，所以每个接口实例叫做 Entry。PerformanceEntry 中定义了四个基本的只读属性：name、entryType、startTime 和 duration。

#####  Performance Navigation Timing属性

`const navTimes = performance.getEntriesByType('navigation')`

卸载旧文档、重定向/卸载、应用缓存、DNS 解析、TCP 握手、HTTP 请求处理、HTTP 响应处理、DOM 处理、文档装载完成。每个小块的首尾、中间做事件分界，取 Unix 时间戳，两两事件之间计算时间差，从而获取中间过程的耗时（精确到毫秒级别）

![web-monitor](https://w3c.github.io/navigation-timing/timestamp-diagram.svg)

- DNS (domainLookupEnd - domainLookupStart): DNS 查询话费的时间。如果是重用连接或是使用了本地 DNS 数据缓存的话，此值为 0。

- TCP (connectEnd - connectStart): 建立 TCP 连接花费的时间。如果是走 HTTPS 协议，中间还多一步 TLS 协商生成会话密钥的过程。

- Request (responseStart - requestStart): 从开始发起请求，到接收到第一个字节的响应数据，中间经历的时间。

- Response (responseEnd - responseStart): 从接受第一个字节的响应数据到接收最后一个字节的响应数据中间经历的时间，也可以认为是资源下载时间。

- Processing (domComplete - domInteractive): 渲染页面花费的时长。如果数值很大，那么你可能就要考虑去优化文档结构或是资源大小了。

- Load Event (loadEventEnd - loadEventStart): 当浏览器完成页面中所有的资源加载的时候，会触发一个 load 事件.  在这个阶段，你可以在事件处理函数中，添加额外的处理逻辑。

- Total Time: 上图中各项指标的花费时间总和。

  

可以根据 PerformanceTiming API 中的字段处理后得到以下关键指标：

| 描述                                       | 计算方式                                      | 备注                                                         |
| ------------------------------------------ | --------------------------------------------- | ------------------------------------------------------------ |
| First Paint Time，首次渲染时间（白屏时间） | fpt = responseEnd - fetchStart                | 从请求开始到浏览器开始解析第一批HTML文档字节的时间差。       |
| Time to Interact，首次可交互时间           | tti = domInteractive - fetchStart             | 浏览器完成所有HTML解析并且完成DOM构建，此时浏览器开始加载资源。 |
| HTML加载完成时间， DOM Ready时间           | ready = domContentLoadEventEnd - fetchStart   | 如果页面有同步执行的JS，则同步JS执行时间=ready-tti。         |
| 页面完全加载时间                           | load = loadEventStart - fetchStart            | load=首次渲染时间+DOM解析耗时+同步JS执行+资源加载耗时。      |
| 首包时间                                   | firstbyte = responseStart - domainLookupStart | DNS + TCP + request                                          |



区间阶段耗时指标：

| 描述                                       | 计算公式                     |
| ------------------------------------------ | -----------------------------------------------  |
| DNS查询                                | dns = domainLookupEnd - domainLookupStart        |
| TCP连接                                | tcp = connectEnd - connectStart                |
| SSL安全连接 | ssl = connectEnd - secureConnectionStart |
| Time to First Byte（TTFB），请求响应 | ttfb = responseStart - requestStart             |
| 数据内容传输                        | trans = responseEnd - responseStart             |
| DOM解析                                | dom = domInteractive - responseEnd             |
| 资源加载耗时-页面中的同步加载资源                               | res = loadEventStart - domContentLoadedEventEnd    |
| SSL安全连接耗时- 只在HTTPS下有效                             | ssl = connectEnd - secureConnectionStart            |



- 慢加载追踪

  在页面 `onload` 后，可以利用 `performance.getEntriesByType('resource')` 获取资源加载时间，超过阈值的时候，进行上报。

- 浏览器内存

  可使用 ` performance.memory` 进行数据采集。

- 性能导航接口

PerformanceNavigation接口呈现了如何导航到当前文档的信息。PerformanceNavigation有两个属性，一个是type，表示如何导航到当前页面的，主要有4个值。

> type=0：表示当前页面是通过点击链接，书签和表单提交，或者脚本操作，或者在url中直接输入地址访问的。
> type=1: 表示当前页面是点击刷新或者调用Location.reload()方法访问的。
> type=2: 表示当前页面是通过历史记录或者前进后退按钮访问的。
> type=255: 其他方式访问的

另外一个属性是redirectCount，表示到达当前页面之前经过几次重定向。

```js
[Exposed=Window]
interface PerformanceNavigation {
  const unsigned short TYPE_NAVIGATE = 0;
  const unsigned short TYPE_RELOAD = 1;
  const unsigned short TYPE_BACK_FORWARD = 2;
  const unsigned short TYPE_RESERVED = 255;
  readonly attribute unsigned short type;
  readonly attribute unsigned short redirectCount;
  [Default] object toJSON();
};
```



实现：：https://juejin.cn/post/6844903662020460552

https://juejin.cn/post/6862559324632252430#heading-7



###  报错及console日志监控



js报错、promise错误

- js错误监控 ----- 重写`window.onerror `方法 

```js
/** 
   * @param {String} errorMessage  错误信息 
   * @param {String} scriptURI   出错的文件 
   * @param {Long}  lineNumber   出错代码的行号 
   * @param {Long}  columnNumber  出错代码的列号 
   * @param {Object} errorObj    错误的详细信息，Anything 
*/
window.onerror = function(errorMessage, scriptURI, lineNumber,columnNumber,errorObj){
	console.log(errorMessage, scriptURI, lineNumber,columnNumber,errorObj)
    sendMessage(errorMessage, scriptURI, lineNumber,columnNumber,errorObj)
}
```

- promise错误 ----- 重写`window.onunhandledrejection`方法

当用到Promise，忘记写reject捕获方法的时候，系统总是会抛出一个叫Unhandled Promise rejection

```js








```
















































































==web-monitor-sdk-master==






















































































































