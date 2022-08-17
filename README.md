

##   前端页面监控





###  项目结构





监控、数据上报























###  性能指标监控

https://w3c.github.io/navigation-timing/

https://juejin.cn/post/6844904182202253325

Performance API有三个api组成：

- Resource Timing API：与网页资源（脚本、样式、图片等）加载相关的耗时信息，定义了接口 PerformanceResourceTiming。
- Navigation Timing API：从页面导航开始一直到 load 事件结束，中间经历过程的耗时信息。定义了接口 PerformanceNavigationTiming，此接口继承自 PerformanceResourceTiming 接口。
- Paint Timing：与网页绘制相关的耗时信息。定义了接口 PerformancePaintTiming。

上面三个接口，又都继承自Performance Timeline API中的 PerformanceEntry 接口，所以每个接口实例叫做 Entry。PerformanceEntry 中定义了四个基本的只读属性：name、entryType、startTime 和 duration。

- ==*Performance Navigation Timing属性*==

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



###  错误监控

https://juejin.cn/post/6867773840768909326



js报错、promise错误、

- ==*js错误监控*== ----- 重写`window.onerror `方法 

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

 window.onerror缺点：只能绑定一个回调函数，如果想在不同文件中想绑定不同的回调函数，不同回调函数直接容易造成互相覆盖。(2) 回调函数的参数过于离散 

```js
/**
 * @param event 事件名
 * @param function 回调函数
 * @param useCapture 回调函数是否在捕获阶段执行，默认是false，在冒泡阶段执行
 */
window.addEventListener('error', (event) => {
  // addEventListener 回调函数的离散参数全部聚合在 error 对象中
}, true)
```

一般情况下用 addEventListener 来代替，在一些特殊情况下依然需要使用 window.onerror。比如，不期望在控制台抛出错误时，因为**只有 window.onerror 才能阻止抛出错误到控制台**

- ==*promise错误*== ----- 重写`window.onunhandledrejection`方法

Promise错误事件有两种，unhandledrejection以及rejectionhandled。

- - 当 Promise被 reject 且没有 reject 处理器的时候，会触发 unhandledrejection事件。

- - 当Promise被 reject 且有 reject 处理器的时候，会触发 rejectionhandled事件。


```js
// unhandledrejection 推荐处理方案
window.addEventListener('unhandledrejection', (event) => {
  console.log(event)
}, true);
// unhandledrejection 备选处理方案
window.onunhandledrejection = function (error) {
  console.log(error)
}

window.onunhandledrejection = function(e) {
	var errorMsg = typeof e.reason === "object" ?  JSON.stringify(e.reason) :  e.reason;
      sendMessage(errorMsg);
 }
```

- ==*Vue框架错误*==

在Vue2.2.0+ 中，框架提供了 errorHandler 这个 API 来捕获并处理错误。

由于Vue会捕获所有Vue单文件组件或者Vue.extend继承的代码，所以在Vue里面出现的错误，并不会直接被window.onerror捕获，而是会抛给Vue.config.errorHandler

```js
Vue.config.errorHandler = function (err, vm, info) {
  // handle error
  // `info` 是 Vue 特定的错误信息，比如错误所在的生命周期钩子
}
```

- ==*console错误*==





- ==*sourcemap*==

利用 source-map来对这些压缩过的代码报错信息进行还原。

https://juejin.cn/post/6844903901393584142

https://mp.weixin.qq.com/s/O2ANZE1vYG-DT6eCB_hf4A

```js
// 根据sourcemap创建SourceMapCustomer对象
var smc = await new sourceMap.SourceMapConsumer(mapfileData);

// 根据每行堆栈信息获取源文件及错误行和列
var po = smc.originalPositionFor({ line: line, column: column });
console.log(po)// line: 1, column:200, source: xxx.js

// 根据源文件内容及行列获取源文件信息
var co = smc.sourceContentFor(po.source)
var coList = co.split("\n")  // 按需取行即可
```



###  请求监控



==*ajax、axios、fetch之间的区别与联系*==

https://zhuanlan.zhihu.com/p/89089088

- AJAX—Asynchronous JavaScript and XML，用JS执行异步网络请求，而不需要重载（刷新）整个页面。

  ```js
  var xmlhttp;
  if(window.XMLHttpRequest){//IE7+,Firefox,Chrome,Opera, Safari
    xmlhttp = new XMLHttpRequest();
  } else {   // code for IE6, IE5
    xmlhttp = new ActiveXObject("Microsoft.XMLHTTP");
  }
  xmlhttp.open("POST", 'http://xxx', true);
  // 如果需要像 HTML 表单那样 POST 数据，使用 setRequestHeader() 来添加 HTTP 头。然后在 send() 方法中规定您希望发送的数据
  xmlhttp.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
  xmlhttp.send("name=小王&age=18");
  xmlhttp.onreadystatechange = function() {
    if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
      document.getElementById("myDiv").innerHTML = xmlhttp.responseText;
    } else {
      console.error("请求错误");
    }
  }
  ```

- axios 是一个基于Promise 用于浏览器和 nodejs 的 HTTP 客户端，本质上也是对原生XHR的封装，只不过它是Promise的实现版本，符合最新的ES规范；

  优点：

  > 从浏览器中创建 XMLHttpRequest
  > 从 node.js 发出 http 请求
  > 支持 Promise API
  > 拦截请求和响应
  > 转换请求和响应数据
  > 取消请求
  > 自动转换JSON数据
  > 客户端支持防止CSRF/XSRF

- Fetch是基于promise设计的，window 自带了 window.fetch 方法。但不是ajax的进一步封装，而是原生js，没有使用XMLHttpRequest对象。

  缺点：

    >1）fetch只对网络请求报错，对400，500都当做成功的请求，服务器返回 400，500 错误码时并不会 reject，只有网络错误这些导致请求不能完成时，fetch 才会被 reject。 
    >
    >2）fetch默认不会带cookie，需要添加配置项： fetch(url, {credentials: 'include'}) 
    >
    >3）fetch不支持abort，不支持超时控制，使用setTimeout及Promise.reject的实现的超时控制并不能阻止请求过程继续在后台运行，造成了流量的浪费 
    >
    >4）fetch没有办法原生监测请求的进度，而XHR可以



备注：

> 防止CSRF：让每个请求都带一个从cookie中拿到的key, 根据浏览器同源策略，假冒的网站是拿不到你cookie中得key的，这样后台就可以轻松辨别出这个请求是否是用户在假冒网站上的误导输入，从而采取正确的策略。





目的：监控所有的接口请求、记录接口请求的返回状态结果、监控接口的报错情况及定位线上问题产生的原因、分析接口的性能以辅助对前端应用的优化



==*监听ajax请求*==



XMLHttpRequest.readyState的五种就绪状态：

> 0：请求未初始化（还没有调用 open()）。
> 1：请求已经建立，但是还没有发送（还没有调用 send()）。
> 2：请求已发送，正在处理中（通常现在可以从响应中获取内容头）。
> 3：请求在处理中；通常响应中已有部分数据可用了，但是服务器还没有完成响应的生成。
> 4：响应已完成；您可以获取并使用服务器的响应了。

XMLHttpRequest事件

请求完成  xhr.onload
请求结束  xhr.onloadend
请求出错  xhr.onerror
请求超时  xhr.ontimeout

- 实现原理--- 完全重写XMLHttpRequest 

https://juejin.cn/post/7110757403523563557

https://juejin.cn/post/6844903544982765575

https://www.cnblogs.com/warm-stranger/p/11001077.html

```js
(function () {
  var oldXMLHttpRequest = XMLHttpRequest;
  window.XMLHttpRequest = function () {
    var actual = new oldXMLHttpRequest();
    var self = this;
    this.onreadystatechange = null;
    actual.onreadystatechange = function () {
      if (this.readyState == 1) {
      onLoadStart.call(this);
      } else if (this.readyState == 4) {
        if(this.status==200)
           onLoadEnd.call(this);
        else{
           onError.call(this);
        }
      }
     if (self.onreadystatechange) {
        return self.onreadystatechange();
      }
    };

["status", "statusText", "responseType", "response",
"readyState", "responseXML", "upload"
].forEach(function (item) {
  Object.defineProperty(self, item, {
  get: function () { return actual[item];
   },
  set: function (val) { actual[item] = val;
  }
  });
});

["ontimeout, timeout", "withCredentials", "onload", "onerror", "onprogress","onabort","onloadend","onloadstart"].forEach(function (item) {
  Object.defineProperty(self, item, {
    get: function () { return actual[item]; },
    set: function (val) { actual[item] = val; }
  });
});

// add all pure proxy pass-through methods
["addEventListener", "send", "open", "abort", "getAllResponseHeaders",
"getResponseHeader", "overrideMimeType", "setRequestHeader", "removeEventListener"
].forEach(function (item) {
  Object.defineProperty(self, item, {
    value: function () {
      return actual[item].apply(actual, arguments);
    }
   });
  });
  }
})();
```



> 注意，监听到的接口请求时间和 chrome devtool 上检测到的时间可能不一样。
>
> 因为 chrome devtool 上检测到的是 HTTP 请求发送和接口整个过程的时间。
>
> 但是 xhr 和 fetch 是异步请求，接口请求成功后需要调用回调函数。
>
> 事件触发时会把回调函数放到消息队列，然后浏览器再处理，中间有个等待过程。



==*监听fetch请求*==

对于 fetch，可以根据返回数据中的的 ok 字段判断请求是否成功，如果为 true 则请求成功，否则失败。

```js
import { lazyReportCache } from '../utils/report'
const originalFetch = window.fetch
function overwriteFetch() {
    window.fetch = function newFetch(url, config) {
        const startTime = Date.now()
        const reportData = {
            startTime,
            url,
            method: (config?.method || 'GET').toUpperCase(),
            subType: 'fetch',
            type: 'performance',
        }

        return originalFetch(url, config)
        .then(res => {
            reportData.endTime = Date.now()
            reportData.duration = reportData.endTime - reportData.startTime

            const data = res.clone()
            reportData.status = data.status
            reportData.success = data.ok

            lazyReportCache(reportData)
            return res
        })
        .catch(err => {
            reportData.endTime = Date.now()
            reportData.duration = reportData.endTime - reportData.startTime
            reportData.status = 0
            reportData.success = false

            lazyReportCache(reportData)
            throw err
        })
    }
}
export default function fetch() { overwriteFetch() }
```

















### 访问速度





- ==*卡顿情况——刷新率FPS*==

网页的 FPS 是只浏览器在渲染这些变化时的帧率。帧率越高，用户感觉网页越流畅，反之则会感觉卡顿。最优的帧率是 60，即16.5ms 左右渲染一次。

```js
const next = window.requestAnimationFrame 
    ? requestAnimationFrame : (callback) => { setTimeout(callback, 1000 / 60) }

const frames = []
export default function fps() {
    let frame = 0
    let lastSecond = Date.now()
    function calculateFPS() {
        frame++
        const now = Date.now()
        if (lastSecond + 1000 <= now) {
            // 由于now-lastSecond 的单位是毫秒，所以frame要 * 1000
            const fps = Math.round((frame * 1000) / (now - lastSecond))
            frames.push(fps)                
            frame = 0
            lastSecond = now
        }   
        
        if (frames.length >= 60) { //避免上报太快，缓存一定数量再上报
            report(deepCopy({
                frames,
                type: 'performace',
                subType: 'fps',
            }))    
            frames.length = 0
        }
        next(calculateFPS)
    }
    calculateFPS()
}

// 当连续三个低于 20 的 FPS 出现时，我们可以断定页面出现了卡顿
export function isBlocking(fpsList, below = 20, last = 3) {
    let count = 0
    for (let i = 0; i < fpsList.length; i++) {
        if (fpsList[i] && fpsList[i] < below) { count++ } 
        else { count = 0 }
        if (count >= last) { return true }
    }
    return false
}
```



- ==*Vue 路由变更渲染时间*==

非首次进入页面计算渲染世间

非首屏渲染时间，如何计算 SPA 应用的页面路由切换导致的页面渲染时间



1. 监听路由钩子，在路由切换时会触发 router.beforeEach() 钩子，在该钩子的回调函数里将当前时间记为渲染开始时间。

2. 利用 Vue.mixin() 对所有组件的 mounted() 注入一个函数。每个函数都执行一个防抖函数。

3. 当最后一个组件的 mounted() 触发时，就代表该路由下的所有组件已经挂载完毕。可以在 this.$nextTick() 回调函数中获取渲染时间。

> 同时，还要考虑到一个情况。不切换路由时，也会有变更组件的情况，这时不应该在这些组件的 `mounted()` 里进行渲染时间计算。所以需要添加一个 `needCalculateRenderTime` 字段，当切换路由时将它设为 true，代表可以计算渲染时间了。 



```js
export default function onVueRouter(Vue, router) {
    let isFirst = true
    let startTime
    router.beforeEach((to, from, next) => {
        if (isFirst) {  // 首次进入页面已经有其他统计的渲染时间可用
            isFirst = false
            return next()
        }
        // 给router新增字段--是否要计算渲染时间，只有路由跳转才需要计算
        router.needCalculateRenderTime = true
        startTime = performance.now()
        next()
    })
    let timer
    Vue.mixin({
        mounted() {
            if (!router.needCalculateRenderTime) return
            this.$nextTick(() => {
                // 仅在整个视图都被渲染之后才会运行的代码
                const now = performance.now()
                clearTimeout(timer)
                timer = setTimeout(() => {
                    router.needCalculateRenderTime = false
                    lazyReportCache({
                        type: 'performance',
                        subType: 'vue-router-change-paint',
                        duration: now - startTime,
                        startTime: now,
                        pageURL: window.location.href,
                    })
                }, 1000)
            })
        },
    })
}
```





###   用户行为监控

PV、UV、用户停留时间

PV(page view) 是页面浏览量，UV(Unique visitor)用户访问量。PV 只要访问一次页面就算一次，UV 同一天内多次访问只算一次。

对于前端来说，只要每次进入页面上报一次 PV 就行，UV 的统计放在服务端来做，主要是分析上报的数据来统计得出 UV。













小程序做法-----监听页面加载的生命周期--DOMContentLoaded、load 事件

https://mp.weixin.qq.com/s/O2ANZE1vYG-DT6eCB_hf4A

```js
export function onBFCacheRestore(callback) {
    window.addEventListener('pageshow', event => {
        if (event.persisted) { callback(event) }
    }, true)
}

export default function observerLoad() {
    ['load', 'DOMContentLoaded'].forEach(type => onEvent(type))

    onBFCacheRestore(event => {
        requestAnimationFrame(() => {
            ['load', 'DOMContentLoaded'].forEach(type => {
                lazyReportCache({
                    startTime: performance.now() - event.timeStamp,
                    subType: type.toLocaleLowerCase(),
                    type: 'performance',
                    pageURL: window.location.href,
                    bfc: true,
                })
            })
        })
    })
}

function onEvent(type) {
    function callback() {
        lazyReportCache({
            type: 'performance',
            subType: type.toLocaleLowerCase(),
            startTime: performance.now(),
        })
        window.removeEventListener(type, callback, true)
    }
    window.addEventListener(type, callback, true)
}
```



- ==***页面跳转***==







Vue 路由变更—— Vue 可以利用 `router.beforeEach` 钩子进行路由变更的监听。 







### 终端信息



















### 移动端监控

















### Node做性能监控





https://zhuanlan.zhihu.com/p/516098478
























































































==web-monitor-sdk-master==






















































































































