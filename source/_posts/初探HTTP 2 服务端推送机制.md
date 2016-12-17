title: "初探HTTP/2 服务器推送机制"
date: 2016-02-16 19:18:09
tags: [node.js]
---
> HTTP/2（超文本传输协议第2版，最初命名为HTTP 2.0），是HTTP協議的的第二个主要版本，使用於全球資訊網。HTTP/2是HTTP協議自1999年HTTP 1.1发布后的首个更新，主要基於SPDY協定。它由互联网工程任务组（IETF）的Hypertext Transfer Protocol Bis（httpbis）工作小组进行开发。该组织于2014年12月将HTTP/2标准提议递交至IESG进行讨论，于2015年2月17日被批准。HTTP/2标准于2015年5月以RFC 7540正式发表

<!-- more -->

> 设计SPDY的目的在于降低网页的加载时间。通过优先级和多路复用，SPDY使得只需要建立一个TCP连接即可传送网页内容及图片等资源。SPDY中广泛应用了TLS加密，传输内容也均以gzip或DEFLATE格式压缩（与HTTP不同，HTTP的头部并不会被压缩）。另外，除了像HTTP的网页服务器被动的等待浏览器发起请求外，SPDY的网页服务器还可以主动推送内容。

HTTP/2新增特性，服务器根据客户端一次请求内容主动推送与之相关的请求过去，避免客户端在解析出初次请求页面内容时，再逐一发送资源请求，节省网络资源利用效率。 一些注意事项：

* 客户端可以通过设置SETTINGS_ENABLE_PUSH为0值通知服务器端禁用推送
* 承诺请求应该是可缓存、安全，并且不能够携带请求的负载内容，这需要客户端做检测
* 推送的响应若不可缓存，客户端不能作为HTTP cache存储，这对单独的非浏览器环境特别适合
* 服务器必须包含一个:authority伪头部字段，标明自身被授权。客户端若检测不到需要作为PROTOCOL_ERROR类型流错误对待
* 中介设备接收到服务器的推送后，可以决定是否要转发给客户端，中介可以单独选择推送内容发送给客户端。这是一个特别需要注意的点
* 客户端必须拒绝来自服务器端的对SETTINGS_ENABLE_PUSH属性非0值的修改，也就是说服务器不能要求客户端打开PUSH开关，客户端一旦遇到需要响应PROTOCOL_ERROR类型连接错误
* 客户端不能够发送推送，PUSH_PROMISE帧只能够来自于服务器端（作为推送请求者发送），否则将会作为PROTOCOL_ERROR类型的连接错误对待
* PUSH_PROMISE需要包含伪头部:method，若客户端认为不安全，必须响应一个PROTOCOL_ERROR类型流错误
* 服务器端应该尽可能早的发送PUSH_PROMISE帧，以避免与来自客户端对相同资源的请求两者产生冲突
* 发送PUSH_PROMISE帧会创建一个新的流，然后处于两端的保留状态，reserved (local/remote)
* 发送完PUSH_PROMISE帧，服务器需要马上发送具体DATA数据帧
* 客户端接收完PUSH_PROMISE帧后，选择接收PUSH响应内容，这期间不能触发请求承诺的响应内容，直到承诺流关闭
* 客户端不需要接收推送内容时，可以选择发送RST_STREAM帧，包含CANCEL/REFUSED_STREAM代码，以及PUSH流标识符发送给服务器端，重置推送流
* 客户端可以通过设置SETTINGS_MAX_CONCURRENT_STREAMS限制响应数，值为0禁用。但不能阻止服务器发送PUSH_PROMISE帧

比如，服务器接收到来自客户端的请求某个HTML文档资源，该文档包含了若干图片连接，服务器应该优先发送图片数据到客户端，这需要优先发送推送承诺早于包含完整HTML文档内容的DATA帧，这样客户端优先接收到承诺资源，后面接收到DATA数据帧进行解析出图片连接的时候，就避免再次发送图片资源请求。

### Express 中使用服务端推送
```js
var fs = require('fs');
var spdy = require('spdy');
var express = require('express');
var app = express();

app.get('/', function (req, res) {
    res.write('<h1>Hello World!</h1>');
    var stream = res.push('/main.js', {
        request: {
            accept: '*/*'
        },
        response: {
            'content-type': 'application/javascript'
        }
    });
    stream.on('error', function() {
    });
    stream.end('alert("hello from push stream!");');
    res.end('<script src="/main.js"></script>');
});

var options = {
    key: fs.readFileSync('./keys/localhost.key'),
    cert: fs.readFileSync('./keys/localhost.crt')
};

spdy.createServer(options, app).listen(443);
```

### Koa 中使用服务端推送
```js
var fs = require('fs');
var Readable = require('stream').Readable;
var inherits = require('util').inherits;
var spdy = require('spdy');
var koa = require('koa');
var co = require('co');
var app = koa();

function View(context){
    this.context = context;
    Readable.call(this, {});

    co.call(this, this.render).catch(context.onerror);
}

inherits(View, Readable);

View.prototype.render = function *() {
    this.push('<!DOCTYPE html><html><head><title>Hello World</title></head>');
    this.push('<body>Hello World!</body>');
    var stream = this.context.res.push('/main.js', {
        request: {
            accept: '*/*'
        },
        response: {
            'content-type': 'application/javascript'
        }
    });
    stream.on('error', (err) => {});
    stream.end('alert("hello from push stream!");');
    stream.on('finish', () => {});
    this.push('<script src="/main.js"></script>');
    this.push('</html>');
    this.push(null);
}

app.use(function *(){
    this.type = 'html';
    this.body = new View(this);
});

var options = {
    key: fs.readFileSync('./keys/localhost.key'),
    cert: fs.readFileSync('./keys/localhost.crt'),
};

spdy.createServer(options, app.callback()).listen(443);
```

当然，这推送机制有个弊端，当客户端HTML加载时发现已经缓存了所需资源文件，并发送RST_STREAM帧，此时服务端在收到RST_STREAM帧之前已经发送出了资源造成浪费，同时收到RST_STREAM帧的服务端会停止推送剩余数据。
推送机制是用于代替过去将图片通过Data URI放置在页面内来减少请求数量，而在HTTP/2内，请求变得十分廉价，不再需要内联方式合并请求。但在请求外部资源时依然会造成延迟，此时服务端推送的作用才显现出来。
另外客户端可以通过限制推送数目，推送数据大小等方面控制推送用途。
所以采用服务端推送时应考虑推送核心资源来减少客户端加载的延迟感受。
