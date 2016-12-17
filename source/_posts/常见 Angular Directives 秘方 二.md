title: 常见 Angular Directives 秘方 二
date: 2015-11-09 19:27:25
tags: [angular.js]
---

指令是Angular强大的概念之一,它能使你创造新的HTML元素到应用中.指令能使你构建封装着复杂DOM结构,样式甚至可复用的组件.

<!-- more -->

##  从HTML属性获取参数

### 问题
你希望通过参数配置控制渲染

### 方案
使用基于属性的指令并从中获取参数,你能从`link`方法的参数中获取属性

```html
<body ng-app="MyApp">
  <div my-widget="Hello World"></div>
</body>
```

```js
var app = angular.module("MyApp", []);

app.directive("myWidget", function() {
  var linkFunction = function(scope, element, attributes) {
    scope.text = attributes["myWidget"];
  };

  return {
    restrict: "A",
    template: "<p></p>",
    link: linkFunction
  };
});
```
这使文字通过HTML参数获取并展示

### 详情
link函数有权访问元素及其属性,因此简单的设置范围用以传递作为属性值的文本,并使用在模板中
作用范围是较为重要的部分, 我们可在父范围内定义可改变的文本模型并使用在其他的应用中.
为了隔离上下文,并只使用在当前指令中,我们需要返回一个额外的scope属性

```js
return {
  restrict: "A",
  template: "<p></p>",
  link: linkFunction,
  scope: {}
};
```

这就是Angular中的隔离域,它不是典型的从父域中继承而来.当我们创建一个可复用组件时隔离域非常有用.
现在看一看另外一种传递参数的方式.以下示例将定义一个HTML元素 `my-widget2`

```html
<my-widget2 text="Hello World"></my-widget2>
```

```js
app.directive("myWidget2", function() {
  return {
    restrict: "E",
    template: "<p></p>",
    scope: {
      text: "@text"
    }
  };
});
```

`@text` 绑定了标签属性中的text模型并定义为当前指令的域.注意任何改变父域的`text`都会改变指令域的`text`.
如果你想在父域与子域中进行双向绑定,你需要使用`=`等于号.

```js
scope: {
  text: "=text"
}
```
改变本地域也会改变父域
另一种方式可以使用`&`符号让指令将表达式作为函数进行调用

```html
<my-widget-expr fn="count = count + 1"></my-widget-expr>
```

```js
app.directive("myWidgetExpr", function() {
  var linkFunction = function(scope, element, attributes) {
    scope.text = scope.fn({ count: 5 });
  };

  return {
    restrict: "E",
    template: "<p></p>",
    link: linkFunction,
    scope: {
      fn: "&fn"
    }
  };
});
```

可通过属性中的fn把表达式传递给指令,我们在`linkFunction`中能传递参数并调用函数

## 重复渲染指令DOM子节点

### 问题
你希望使用指令的子节点作为标识内容重复渲染一个HTML片段

### 方案

在指令中实现一个复杂的编译函数
```html
<repeat-ntimes repeat="10">
  <h1>Header 1</h1>
  <p>This is the paragraph.</p>
</repeat-n-times>
```

```js
var app = angular.module("MyApp", []);

app.directive("repeatNtimes", function() {
  return {
    restrict: "E",
    compile: function(tElement, attrs) {
      var content = tElement.children();
      for (var i=1; i<attrs.repeat; i++) {
        tElement.append(content.clone());
      }
    }
  };
});
```
这将连续十次渲染`h1`和`p`标签的内容

### 详情

指令会依据`repeat`属性重复子节点.工作方式类似`ng-repeat`,依据Angulare的element方法去循环增加子节点
注意,编译方法只可以访问模板元件tElement和模板的属性,其他的范围无法访问.因此无法使用`$watch`去增添行为.相比之下`link`函数则可以访问DOM实例(编译阶段后),并获得域的范围并增加行为.
只为DOM模板控制使用编译,而`link`函数可以随时增加行为
注意,你能同时使用`compile`和`link`函数,在以下实例中,`compile `函数会返回一个`link`函数,能为点击h1做出反应

```js
compile: function(tElement, attrs) {
  var content = tElement.children();
  for (var i=1; i<attrs.repeat; i++) {
    tElement.append(content.clone());
  }

  return function (scope, element, attrs) {
    element.on("click", "h1", function() {
      $(this).css({ "background-color": "red" });
    });
  };
}
```
点击h1会改变背景色为红色

## 指令间的通讯

### 问题
你希望通过一个定义明确的接口来让指令间通讯

## 方案
构建一个有controller函数的`basket`指令和需要这个controller的两个指令`orange`和`apple`.

```html
<body ng-app="MyApp">
  <basket apple orange>Roll over me and check the console!</basket>
</body>
```
`basket`指管理一个数组可增加苹果和橙子
```js
var app = angular.module("MyApp", []);

app.directive("basket", function() {
  return {
    restrict: "E",
    controller: function($scope, $element, $attrs) {
      $scope.content = [];

      this.addApple = function() {
        $scope.content.push("apple");
      };

      this.addOrange = function() {
        $scope.content.push("orange");
      };
    },
    link: function(scope, element) {
      element.bind("mouseenter", function() {
        console.log(scope.content);
      });
    }
  };
});
```

最终`apple`和`orange`指令可以使用`basket`控制器

```js
app.directive("apple", function() {
  return {
    require: "basket",
    link: function(scope, element, attrs, basketCtrl) {
      basketCtrl.addApple();
    }
  };
});

app.directive("orange", function() {
  return {
    require: "basket",
    link: function(scope, element, attrs, basketCtrl) {
      basketCtrl.addOrange();
    }
  };
});
```
如果鼠标移动到文本上,会输出`basket`的内容

### 详情

鉴于`apple`和`orange`作为`basket`指令的属性,演示了指令如何使用controller函数,他们都定义需要依赖于`basket`控制器,所以`link`函数得到`basketCtrl`的注入,可为可复用组件增加额外指令

## 测试指令

### 问题
希望以单元测试方式测试编写的指令,虾米是一个简单的tab组件例子.
```html
<tabs>
  <pane title="First Tab">First pane.</pane>
  <pane title="Second Tab">Second pane.</pane>
</tabs>
```

```js
app.directive("tabs", function() {
  return {
    restrict: "E",
    transclude: true,
    scope: {},
    controller: function($scope, $element) {
      var panes = $scope.panes = [];

      $scope.select = function(pane) {
        angular.forEach(panes, function(pane) {
          pane.selected = false;
        });
        pane.selected = true;
        console.log("selected pane: ", pane.title);
      };

      this.addPane = function(pane) {
        if (!panes.length) $scope.select(pane);
        panes.push(pane);
      };
    },
    template:
      '<div class="tabbable">' +
        '<ul class="nav nav-tabs">' +
          '<li ng-repeat="pane in panes"' +
              'ng-class="{active:pane.selected}">'+
            '<a href="" ng-click="select(pane)"></a>' +
          '</li>' +
        '</ul>' +
        '<div class="tab-content" ng-transclude></div>' +
      '</div>',
    replace: true
  };
});
```

```js
app.directive("pane", function() {
  return {
    require: "^tabs",
    restrict: "E",
    transclude: true,
    scope: {
      title: "@"
    },
    link: function(scope, element, attrs, tabsCtrl) {
      tabsCtrl.addPane(scope);
    },
    template:
      '<div class="tab-pane" ng-class="{active: selected}"' +
        'ng-transclude></div>',
    replace: true
  };
});
```

### 方案
使用angular-seed与`jasmine`和`jasmine-jquery`写作进行单元测试

 ```js
describe('MyApp Tabs', function() {
  var elm, scope;

  beforeEach(module('MyApp'));

  beforeEach(inject(function($rootScope, $compile) {
    elm = angular.element(
      '<div>' +
        '<tabs>' +
          '<pane title="First Tab">' +
            'First content is ' +
          '</pane>' +
          '<pane title="Second Tab">' +
            'Second content is ' +
          '</pane>' +
        '</tabs>' +
      '</div>');

    scope = $rootScope;
    $compile(elm)(scope);
    scope.$digest();
  }));

  it('should create clickable titles', function() {
    console.log(elm.find('ul.nav-tabs'));
    var titles = elm.find('ul.nav-tabs li a');

    expect(titles.length).toBe(2);
    expect(titles.eq(0).text()).toBe('First Tab');
    expect(titles.eq(1).text()).toBe('Second Tab');
  });

  it('should set active class on title', function() {
    var titles = elm.find('ul.nav-tabs li');

    expect(titles.eq(0)).toHaveClass('active');
    expect(titles.eq(1)).not.toHaveClass('active');
  });

  it('should change active pane when title clicked', function() {
    var titles = elm.find('ul.nav-tabs li');
    var contents = elm.find('div.tab-content div.tab-pane');

    titles.eq(1).find('a').click();

    expect(titles.eq(0)).not.toHaveClass('active');
    expect(titles.eq(1)).toHaveClass('active');

    expect(contents.eq(0)).not.toHaveClass('active');
    expect(contents.eq(1)).toHaveClass('active');
  });
});
```

### 详情
示例中广泛使用了 `jasmine`结合`jasmine-jquery`提供的`toHaveClass`和`click`事件
在`beforeEach`前使用`$compile`和`$digest`准备模板,并测试最终的Angular元素
示例取自[Vojta Jina’ Github example](https://github.com/vojtajina/ng-directive-testing/tree/start)



































