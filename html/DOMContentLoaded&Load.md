# DOMContentLoaded&Load
DOMContentLoaded: 当 HTML 文档解析完成就会触发 DOMContentLoaded (生成DOM（文档对象模型）, CSSOM（CSS 对象模型）, 下载执行js文件)
Load: 当整个页面及所有依赖资源如样式表和图片都已完成加载时，将触发load事件
Finish: Finish 时间是页面上所有 http 请求发送到响应完成的时间，HTTP1.0/1.1 协议限定，单个域名的请求并发量是 6 个，即Finish是所有请求（不只是XHR请求，还包括DOC，img，js，css等资源的请求）在并发量为6的限制下完成的时间