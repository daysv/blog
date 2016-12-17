title: res-interceptor
date: 2015-01-01 19:45:20
tags: [node.js,io.js]
---

在使用TJ大神koa之前,我们用Express写一些中间件有一定的局限,只能通过hack的方式实现一些功能,`res-interceptor`主要是通过hack的方式来构建一个较为灵活的Express中间件

<!-- more -->

在介绍本中间件之前,还是先说说koa的一些优势

###这是koa github中的一段经典代码
```
var koa = require('koa');
var app = koa();

// logger

app.use(function *(next){
  var start = new Date;
  yield next;
  var ms = new Date - start;
  console.log('%s %s - %s', this.method, this.url, ms);
});

// response

app.use(function *(){
  this.body = 'Hello World';
});

app.listen(3000);
```

1. 在logger步骤中,当运行到`yield next;`时则进入到response步骤
2. 执行response步骤,其中设置好body(即Express中res.send())
3. 继续执行logger步骤中`yield next;`后面的步骤
可以看到koa的中间件可以实现一层嵌套一层,需要给用户发送的内容通过`this.body`设置

####koa中实现覆盖
```
var koa = require('koa');
var app = koa();

app.use(function *(next){
    yield next;
    this.body='Overwrite it';
});

app.use(function *(){
    this.body = 'Hello World';
});


app.listen(3000);
```

显示结果为Overwrite it ,简单轻松就把之前设置的Hello World 覆盖掉


###Express 代码
```
var express = require('express');
var app = express();

app.use(function(req,res,next){
    next();
    res.send('Overwrite it');
});

app.get('/', function (req, res) {
    res.send('Hello World');
});

app.listen(3000);
```
很明显会报错误`Error: Can't set headers after they are sent.`
Express 在调用`res.send`等方法后会立即调用`res.end`,使得中间件编写产生一定局限

#res-interceptor
用于在Express中获取及修改响应信息的中间件

##安装

```
npm install res-interceptor -save
```

##使用

```js
var express = require('express');
var interceptor=require('../interceptor');
var app = express();

app.use(
    interceptor (function (req,res,next,data) {
        console.log(data);
        // => { headers: { 'x-powered-by': 'Express', id: '1' }, status: 200, body: 'hello world' }
        this.set('id','2'); // 重写了headerss中的id:1
        // or
        this.set({
            foo:'bar'
        });
        this.body('Goodbye'); // 重写了hello world
    })
);

app.get('/', function (req, res) {
    res.set('id','1');
    res.send('hello world');
});

app.listen(3000);
```

这样就能轻松进行更加灵活的`response`操作

