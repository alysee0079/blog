#### 资源合并与压缩

    服务端压缩: Gzip
    html压缩:
      html压缩就是压缩这些在文本文件中有意义,但是在html中不显示的字符,包括空格,制表符,换行符等,还有一些其他意义的字符,如html注释也可以被压缩.
    如何进行html压缩:
      1.使用在线工具压缩
      2.nodejs提供了html-minifier工具(在构建层压缩)
      3.后端模板引擎渲染压缩
    css压缩:无效代码删除,css语义合并.
      如何进行css压缩:
      1.在线网站
      2.html-minifier
      3.使用clean-css
    js压缩与混乱: 无效字符删除,删除注释,代码语义的缩短和优化,代码保护.
      如何进行js压缩与混乱:
      1.使用在线工具
      2.html-minifier
      3.使用uglifyjs对js进行压缩
    文件合并
      1. 在线合并
      2. webpack, fis3
    图片压缩:
      不同格式图片常用的业务场景:
      jpg有损压缩，压缩率高，不支持透明
      png支持透明，浏览器兼容好
      webp压缩程度更好，在ios webview有兼容性问题
      svg矢量图，代码内嵌，相对较小，图片样式相对简单的场景
      CSS雪碧图:把你的网站上用到的一些图片整合到一张单独的图片中, 减少你的网站的HTTP请求数量.
      Image inline(base64): 将图片的内容内嵌到html当中, 减少你的网站的HTTP请求数量.
      矢量图: 使用SVG进行矢量图的绘制, 使用iconfont解决icon问题.
      在线压缩: tinypng

#### http 缓存

###### 强缓存(不向服务器查询)(优先级高于协商缓存)

Cache-Control:

- max-age: 设置缓存最大时间,超过时间则请求网络,否则从缓存中获取资源,优先级高于 Expires.
  - max-age=0: 走协商缓存
- s-max-age: 设置 public(CDN)缓存最大时间,优先级高于 max-age
- private: 只能被浏览器缓存
- public: 允许被代理服务器缓存
- no-cache: 使用协商缓存, 向服务器发请求查询缓存是否过期
- no-store: 不进行缓存

Expires(http1.0):

- 缓存过期时间,用来指定资源到期的时间点,是服务器端的具体时间点.告诉浏览器在过期时间前浏览器可以直接从缓存取数据,而无需再次请求.优先级低于 max-age.

###### 协商缓存(向服务器发送请求查询)

- Last-Modified/If-Modified-Since: 上次文件修改时间.
- Etag/If-None-Match: 文件内容 hash 值.

#### CDN 加速

内容分发到最近服务器

#### DNS 预解析(仅对跨域有效)

作用：根据浏览器定义的规则，提前解析之后可能会用到的域名，使解析结果缓存到系统缓存中，缩短 DNS 解析时间，来提高网站的访问速度

1）是减少 DNS 的请求次数，2）是进行 DNS 预获取——DNS

```javascript
  1、打开和关闭DNS预解析
  <meta http-equiv="x-dns-prefetch-control" content="on">
  3、手动添加解析
  <link rel="dns-prefetch" href="//www.zhix.net">
```

#### 重用 TCP 链接

尽可能的使用持久链接，以消除 TCP 握手和慢启动导致的延迟

1. connection: keep-alive 要求服务器不要关闭 TCP 链接，以便其他请求复用

#### 服务端性能优化

构建层模板编译
数据无关的 prerender 的方式
服务端渲染 ssr

#### http2
