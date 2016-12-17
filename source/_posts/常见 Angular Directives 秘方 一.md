title: 常见 Angular Directives 秘方 一
date: 2015-10-29 22:17:21
tags: [angular.js]
---

指令是Angular强大的概念之一,它能使你创造新的HTML元素到应用中.指令能使你构建封装着复杂DOM结构,样式甚至可复用的组件.

<!-- more -->

## 依据条件显示/隐藏DOM元素

### 问题
你希望依据`checkbox`状态隐藏`button`

### 方案
使用`ng-disabled`指令并绑定到checkbox状态
```html
<body ng-app>
  <label><input type="checkbox" ng-model="checked"/>Toggle Button</label>
  <button ng-disabled="checked">Press me</button>
</body>
```

### 原理
`ng-disabled`指令直接对HTML属性`disabled`进行转换而不需要考虑浏览器兼容情况.`checkbox`使用`ng-model`指令时,通过使用他的参数值绑定`checked`模型.其实`checked`属性值也是Angular的表达式.如:你能使用`!checked`作为替代品来改变逻辑
这只是Angular封装的其中一个指令.此外有许多其他指令如`ng-hide` `ng-checked` `ng-mouseenter` 等, 请查询Angular提供的[API](https://docs.angularjs.org/api)

## 依据用户行为改变DOM

### 问题
你希望依据点击事件改变一个HTML元素的CSS,并将此封装为可复用组件

### 方案
实现一个指令`my-widget`,它里面包含一个需要我们点击改变CSS的文字段落
```html
<body ng-app="MyApp">
  <my-widget>
    <p>Hello World</p>
  </my-widget>
</body>
```
我们在指令中使用`link function`去改变文字段落的CSS
```js
var app = angular.module("MyApp", []);

app.directive("myWidget", function() {
  var linkFunction = function(scope, element, attributes) {
    var paragraph = element.children()[0];
    $(paragraph).on("click", function() {
      $(this).css({ "background-color": "red" });
    });
  };

  return {
    restrict: "E",
    link: linkFunction
  };
});
```
当点击文字时背景变为红色

### 原理
在页面中使用一个新的指令`my-widget`作为HTML元素,在Javascript中他写作驼峰形式`myWidget`,指令返回`restrict`和`link`方法
`restrict`意思是指令只可用作标签元素,如果想把指令作用与标签属性,你需要把`restrict`参数改为`A`,网页的代码也需要改变
```html
<div my-widget>
  <p>Hello World</p>
</div>
```
你可以依据情况使用元素或属性机制.总体上来说会使用HTML元素即(restrict:"E")来定义可复用组件.属性机制则可用于你想额外设置的一些元素或增加一些行为.其他可选的指令调用方法还有依据类或注释
`directive`方法要求一个方法作为初始化及依赖注入
```js
app.directive("myWidget", function factory(injectables) {
  // ...
}
```
`link`方法能定义一些实际行为.scope变量,当前HTML元素`my-wiget`及HTML属性作为参数.注意Angular 依赖注入机制并没有做任何事,即必须填写参数.

首先我们选中了`my-widget`的子元素文字,然后调用jQuery绑定到click的事件方法并修改css.有趣的是我们可以混用`element`方法和jQuery方法. 其实如果jQuery被引入,Angular会在`children()`调用jQuery方法,没有jQuery Angular则会使用jqLite(封装在Angular内部的简单jQuery)
我们还可以使用如下版本实现相同功能
```js
element.on("click", function() {
  $(this).css({ "background-color": "red" });
});
```

## 在指令中渲染HTML片段

### 问题
你希望在指令中渲染一些HTML片段,并作为可复用的组件

### 方案
在指令中使用`template`定义HTML元素

```html
<body ng-app="MyApp">
  <my-widget/>
</body>
```

```js
var app = angular.module("MyApp", []);

app.directive("myWidget", function() {
  return {
    restrict: "E",
    template: "<p>Hello World</p>"
  };
});

```

### 原理
指令会渲染Hello World 段落作为`my-widget`的子节点. 如果你想整个替换掉指令,需要增加`replace`属性
```js
app.directive("myWidget", function() {
  return {
    restrict: "E",
    replace: true,
    template: "<p>Hello World</p>"
  };
});
```
另一种情形是需要使用文件模板.我们可以使用`templateUrl `参数.
```js
app.directive("myWidget", function() {
  return {
    restrict: "E",
    replace: true,
    templateUrl: "widget.html"
  };
});
```

`widget.html`文件应该与指令在同一目录内,只有在开启web服务时有效.

## 渲染指令的子节点

### 问题
需要指令元素的子节点与模板混合渲染

### 方案
使用`transclude` 属性 并使用`ng-transclude`指令

```html
<my-widget>
  <p>This is my paragraph text.</p>
</my-widget>
```
```js
var app = angular.module("MyApp", []);

app.directive("myWidget", function() {
  return {
    restrict: "E",
    transclude: true,
    template: "<div ng-transclude><h3>Heading</h3></div>"
  };
});
```
div元素包含h3元素并在底部添加指令的子节点

### 原理
`transclusion`包含一个文档的一部分到另一个文档中,`ng-transclusion`属性应该取决于你希望的子节点附加的位置
