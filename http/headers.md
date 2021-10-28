# headers

#### request headers

user-agent: 浏览器信息
content-type: 发送的数据格式

（1）application/x-www-form-urlencoded：浏览器的原生 form 表单，如果不设置 enctype 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数据。该种方式提交的数据放在 body 里面，数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL转码。
（2）multipart/form-data：该种方式也是一个常见的 POST 提交方式，通常表单上传文件时使用该种方式。
（3）application/json：服务器消息主体是序列化后的 JSON 字符串。
（4）text/xml：该种方式主要用来提交 XML 格式的数据。
（5）text/plain：纯文本数据。

accept: 浏览器可接受的数据格式
accept-encoding: 浏览器可接受的压缩方式
accept-languange: 浏览器可接受的的语言
connection: keep-alive 一次 tcp 链接重复使用
cookie
host: 域名

#### response headers

content-type: 返回的数据格式
