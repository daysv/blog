title: Heroku部署node入门指南
date: 2014-08-12 18:33:20
tags: [cloud]
---
Heroku 作为难得的免费PaaS支持Ruby Java Node.js Python PHP等主流语言,相对同样免费的国内某企业,他强大的实在是太多  
如果是个人用或测试用,不在意网速的话强烈推荐使用Heroku部署nodejs  
另外Heroku现已支持Webscoket

<!-- more -->

# 安装Heroku Toolbelt
[Heroku Toolbelt](https://toolbelt.heroku.com)
集成Git版本控制

# 编辑package.json
通过npm init 初始化项目,确保所有的依赖项在`package.json`中被声明,通过`npm install <pkg> --save `安装包  
```json
    {
      "name": "node-example",
      "version": "0.0.0",
      "description": "This example app is so cool.",
      "main": "web.js",
      "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
      },
      "repository": {
        "type": "git",
        "url": "https://github.com/jane-doe/node-example.git"
      },
      "keywords": [
        "example",
        "heroku"
      ],
      "author": "jane-doe",
      "license": "MIT",
      "bugs": {
        "url": "https://github.com/jane-doe/node-example/issues"
      }
    }
```

最后应在`package.json`中添加node版本,如果没有设置,默认版本为部署时常用版本

```json
    {
      "engines": {
        "node": "0.10.x"
      }
    }
```
    
# 通过Procfile声明进程类型

通过在应用根目录创建`Procfile`文件声明如何执行一个`web dyno`

    web: node web.js
    
这样一个简单的配置就写好了.
`web`在这里很重要,他声明了当部署时进程类型将附加在Heroku的[HTTP routing](https://devcenter.heroku.com/articles/http-routing)栈接受流

#储存应用程序到Git
Heroku通过Git来部署,所以部署命令十分简单易懂,进入应用根目录

```bash
    git init
    git add .
    git commit -m "init"
 ```
    
#部署应用到Heroku
首先登录Heroku

```bash
    heroku login
 ```
 
通过命令初始化Heroku app

```bash
    heroku create
```
    
确认查询git remote为heroku

```bash
    git remote -v
```

如果不是可以将remote转为heroku

```bash
	git remote remove heroku
    git remote add heroku [yourgit]
```

部署

```bash
    git push heroku master
```

更改名字

```bash
    heroku rename [newName]
```

# 访问您的应用程序

```bash
    heroku ps:scale web=1
    heroku ps
    heroku open
```
# 查询heroku日志
```bash
    heroku logs
```
