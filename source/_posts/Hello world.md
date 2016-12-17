title: Hello World
date: 2014-08-06 21:04:26
tags: [hexo,node.js,io.js]
---

早早就想建立一个个人的博客，但是迟迟没有动手。早前制作了两个博客，虽然用了`bootstrap`但是大部分时间却还在前端修改，又因原服务器易主，只好放弃。这次的决定用`hexo`来制作静态的博客，前端方面用着最原始的主题（我才不会告诉你我觉得默认的比别的好看多了），或许最看起来最简单的页面最让人舒服吧。
本博客用作记录学习生活的点点滴滴，相信能够在学习中不断进步吧


<!-- more -->


#hexo 搭建个人博客
### 安装node.js 或者io.js
### 安装hexo
[hexo官网](http://hexo.io/)
```
npm install hexo -g
hexo init blog
cd blog
npm install
hexo server
```

因为hexo默认主题需要链接google的cdn资源,而世界上又不存在google这种网站(真是两难的选择),所以需要安装一个支持国内源的皮肤

本站选用了[landscape-plus](https://github.com/xiangming/landscape-plus)
其针对中国大陆地区对hexo官方主题landscape进行优化
### 安装皮肤
在项目文件根目录下执行,需要git
```
git clone https://github.com/xiangming/landscape-plus.git themes/landscape-plus
```
###启用
修改主题的设置文件_config.yml，把theme的值设置为landscape-plus

OK,一个简单的静态blog就建立好了

[详细文档](hexo.io/api/classes/Hexo.html)
如果需要重新生成文件就把根目录db.json文件删除后重新执行`hexo server`