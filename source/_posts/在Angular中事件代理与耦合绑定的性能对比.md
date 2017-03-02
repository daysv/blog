title: 在Angular中事件代理与耦合绑定的性能对比
date: 2015-10-19 21:21:51
tags: [angular.js]
---

上一次讨论了在Angular中通过延迟DOM树的transclusion能增加性能.延迟transclusion 作用于大对象,他减去了大量活动的`$watch()`绑定.但是我们从事件代理中得到了多少性能提升呢?即对每个节点进行绑定与在父节点上进行绑定时间差异.

<!-- more -->

[Demo project on github](http://bennadel.github.io/JavaScript-Demos/demos/event-delegation-vs-linking-angularjs/)

我设计了一个渲染1000条数据的列表用于实验.我们打算绑定`mouseenter`事件同时输出鼠标进入的元素.第一个版本中,我们在父节点中绑定一个简单的指令为每个列表元素使用事件代理.第二个版本中,我们为每个元素绑定指令

```html
<html ng-app="Demo" ng-controller="AppController" bn-window-teaser>
<head>
    <meta charset="utf-8"/>

    <title>
        Event Delegation Performance vs. Linking Performance In AngularJS
    </title>

    <link rel="stylesheet" type="text/css" href="./demo.css" />
</head>
<body>

    <h1>
        Event Delegation Performance vs. Linking Performance In AngularJS
    </h1>

    <h2>
        Event Delegation Performance
    </h2>
 
    <ul bn-friends class="friends">
        <li ng-repeat="friend in friends track by friend.id" class="friend">
            {{ friend.name }}
        </li>
    </ul>


    <!-- Load scripts. -->
    <script type="text/javascript" src="../../vendor/jquery/jquery-2.1.0.min.js"></script>
    <script type="text/javascript" src="../../vendor/angularjs/angular-1.3.6.min.js"></script>
    <script type="text/javascript">
 
        var app = angular.module("Demo", []);
  
        app.controller(
                "AppController",
                function ($scope) {
                    $scope.friends = []; 
                    for (var i = 1; i <= 1000; i++) {
                        $scope.friends.push({
                            id: i,
                            name: ( "Friend " + i )
                        });
                    }
                }
        ); 
        
        app.directive(
                "bnFriends",
                function () { 
                    return ({
                        link: link,
                        restrict: "A"
                    });
                     
                    function link($scope, element, attributes) { 
                        element.on(
                                "mouseenter",
                                "li.friend",
                                function handleMouseEnter(event) {
                                    var target = angular.element(event.target);
                                    console.log("Mousing", angular.element.trim(target.text()));
                                }
                        );

                    }
                }
        );

    </script>

</body>
</html>

```

能看出,指令作用与UI元素,并只绑定一次.他为每个LI元素共同监听`mouseenter`事件.

注意: Angular不允许事件代理使用`.on()`.demo因为引入 `jQuery` 作为 `angular.element()` 构造方法

现在看每个元素都绑定指令的第二个版本的代码

```html

<html ng-app="Demo" ng-controller="AppController" bn-window-teaser>
<head>
    <meta charset="utf-8" />

    <title>
        Event Delegation Performance vs. Linking Performance In AngularJS
    </title>

    <link rel="stylesheet" type="text/css" href="./demo.css"></link>
</head>
<body>

    <h1>
        Event Delegation Performance vs. Linking Performance In AngularJS
    </h1>

    <h2>
        Linking Performance
    </h2>


    <ul class="friends">
        <li ng-repeat="friend in friends track by friend.id" bn-friend class="friend">
            {{ friend.name }}
        </li>
    </ul>


    <!-- Load scripts. -->
    <script type="text/javascript" src="../../vendor/jquery/jquery-2.1.0.min.js"></script>
    <script type="text/javascript" src="../../vendor/angularjs/angular-1.3.6.min.js"></script>
    <script type="text/javascript">
 
        var app = angular.module( "Demo", [] );
          
        app.controller(
                "AppController",
                function( $scope ) {
                    $scope.friends = [];
                    
                    for ( var i = 1 ; i <= 1000 ; i++ ) {
                        $scope.friends.push({
                            id: i,
                            name: ( "Friend " + i )
                        });
                    }
                }
        );

        app.directive(
                "bnFriend",
                function() {
                    return({
                        link: link,
                        restrict: "A"
                    });
                    
                    function link( $scope, element, attributes ) {
                        element.on(
                                "mouseenter",
                                function handleMouseEnter( event ) {
                                    console.log( "Mousing", angular.element.trim( element.text() ) );
                                }
                        );
                    }
                }
        );
    </script>

</body>
</html>

```

在这个版本里,我们为每个`ngRepeat`渲染的副本绑定`bnFriend`指令
切记这并不是完美的测试,可能因电脑和Chrome产生一定影响.无论如何,事件代理的方式都稍微快一点.1000个元素中事件代理通常需要`225ms`到`260ms`进行渲染.为每个元素绑定通常需要`260ms`到`315ms`进行渲染.
![event-delegation-vs-linking-performance-angularjs](http://www.bennadel.com/resources/uploads/2015/event-delegation-vs-linking-performance-angularjs.png)

也就是说,双方都有波动性.考虑到事件代理增加了一定复杂度,我不确定这点性能是否值得.当然这是独立的测试,一些其他真实环境也会影响性能.(如在`linking function`中大量的方法申明.

我很高兴看到`linking function`在大量`set`操作中工作如此得当.不过我宁愿选择容易思考的方法.此外这给在Angular中用jQuery的应用更多的启发.另一方面我非常喜爱jQuery的特性就是事件代理.但是如果事件代理没有赢得大幅度性能,那可能在Angular中并不是非常适合.

[原文](http://www.bennadel.com/blog/2754-event-delegation-performance-vs-linking-performance-in-angularjs.htm)