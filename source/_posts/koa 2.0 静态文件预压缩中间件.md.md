title: koa 2.0 静态文件预压缩中间件
date: 2017-03-01 17:11:17
tags: [koa]
---

随着 nodejs 7.6.0 release, koa 2.0 也紧接着正式发布了
今天在使用koa做静态服务器的时候, 配合 `koa-static` `koa-compress`  时, 每次请求静态资源都会进行要进行一次压缩操作
虽写了个简单的中间件, 在第一次请求的时候对静态文件进行 `gzip` 压缩, 在之后的请求中均依据压缩后的 `.gz` 文件进行返回

<!-- more -->

使用时需配合 `koa-static` 中间件
这里的就没添加 demo`etag` 等缓存机制了

# server.js
```js

const serve = require('koa-static')
const createGzip = require('./modules/gzip')

const staticDir = path.join(__dirname, '/static')

app.use(createGzip(staticDir))
app.use(serve(staticDir))

```


# gzip.js
```js
const fs = require('mz/fs')
const path = require('path')
const Stream = require('stream')
const zlib = require('zlib')
const compressible = require('compressible')
const status = require('statuses')

module.exports = function (staticDir, options) {
    return async function (ctx, next) {
        await next()

        var body = ctx.body
        if (!body) return
        if (ctx.response.get('Content-Encoding')) return
        if (status.empty[ctx.response.status]) return
        if (ctx.request.method === 'HEAD') return

        var encoding = ctx.acceptsEncodings('gzip', 'deflate', 'identity')
        if (!encoding) ctx.throw(406, 'supported encodings: gzip, deflate, identity')
        if (encoding === 'identity') return

        options = options || {}

        const threshold = !options.threshold ? 1024
            : typeof options.threshold === 'number' ? options.threshold
            : typeof options.threshold === 'string' ? bytes(options.threshold)
            : 1024

        if (threshold && ctx.response.length < threshold) return

        const headers = ctx.res._headers

        if (compressible(headers['content-type'])) {
            var _path = ctx.path
            options.index = options.index || 'index.html'

            const _uri = path.parse(_path)
            if (!_uri.ext){
                _uri.dir += _uri.base
                _uri.base = options.index
            }
            const _target = path.join(staticDir, _uri.dir, _uri.base + '.gz')

            const exists = await fs.exists(_target)
            if (!exists) {
                const gzip = zlib.createGzip(options)

                ctx.set('Content-Encoding', encoding)
                ctx.res.removeHeader('Content-Length')

                const stream = ctx.body = gzip

                if (body instanceof Stream) {
                    body.pipe(stream).pipe(fs.createWriteStream(_target))
                }
            }
        }
    }
}
```
