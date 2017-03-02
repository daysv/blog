title: Angular性能实践 - 通过延迟执行Transclusion来延迟DOM树绑定
date: 2015-10-16 20:00:51
tags: [angular.js]
---
当你开始使用AngularJS时,你会发现这东西真TM屌.但是当应用足够复杂,用户界面需要更多的层级,Angular的视图渲染速度开始下降.一旦你的绑定数量接近2000+的临界值时,处理的时间就能被明显感知.所以我通过不断实验寻找让处理时间在可控范围内的方法.我最后一次实验表明当不需要时候则不绑定的方式延迟绑定较少使用的DOM树能有效控制Angular的性能

<!-- more -->

[github demo](http://bennadel.github.io/JavaScript-Demos/demos/defer-dom-subtree-angularjs/)

AnguarJS 已经有方式来排除一些不需要的部分(如ngSwitch, ngSwitchWhen, ngIf 等都能延迟链接已有的DOM树).但是他们都需要`$watch()`绑定来确保DOM树可被操作.(也可以使用一次性绑定来解决一定问题)

我的实验方式是简单的用强迫方式关联已知的上下文来实现,并非创建AngularJS的绑定来监视`$scope`.我使用jQuery的事件代理执行仅仅一次DOM树的`transclusion`

这种方式类似不监视`$scope`实现注入及操作DOM树, 而是当DOM树首次被需要时才永久注入.他只是在相应时间结束而不会管DOM树是否需要被排除(注入后可重复使用).

我在demo中使用大量带有删除确认覆盖的的friends对象来进行验证.
在编译阶段删除并获取`delete confirmation`元素,然后在需要的时候进行注入

```html
<html ng-app="Demo" ng-controller="AppController">
<head>
    <meta charset="utf-8"/>
    <title>
        通过延迟执行Transclusion来延迟DOM树绑定
    </title>

    <link rel="stylesheet" type="text/css" href="demo.css" />
</head>
<body>

    <h1>
        通过延迟执行Transclusion来延迟DOM树绑定
    </h1>

    <ul bn-list class="items">

        <li ng-repeat="friend in friends" class="item">
            <span class="name">{{ friend.name }}</span>
            <a ng-click="showConfirmation( friend )" class="delete">Delete</a>
            <div ng-show="friend.isShowingConfirmation"  class="deleteConfirmation">
                <span class="confirmation">
                    <span class="intent">Delete</span>
                    <span class="target">{{ friend.name }}</span>
                </span>

                <a ng-click="hideConfirmation( friend )" class="action">Delete</a>
                <a ng-click="hideConfirmation( friend )" class="action">Cancel</a>
            </div>
        </li>

    </ul>


    <!-- Load jQuery and AngularJS. -->
    <script type="text/javascript" src="jquery-2.0.3.min.js"></script>
    <script type="text/javascript" src="angular-1.2.min.js"></script>
    <script type="text/javascript">
        var app = angular.module("Demo", []);

        app.controller(
                "AppController",
                function ($scope) {
                    /* 构建friends大对象. */
                    $scope.friends = buildFriends(1000);

                    $scope.showConfirmation = function (friend) {
                        friend.name = friend.name.toUpperCase();
                        friend.isShowingConfirmation = true;
                    };

                    $scope.hideConfirmation = function (friend) {
                        friend.isShowingConfirmation = false;
                    };

                    function buildFriends(count) {
                        var names = ["Sarah", "Joanna", "Tricia"];
                        var friends = [];
                        for (var i = 0; i < count; i++) {
                            friends.push({
                                id: i,
                                name: names[i % 3],
                                isShowingConfirmation: false
                            });

                        }
                        return ( friends );
                    }
                }
        );

        /* 帮助渲染DOM树
            可以延迟一部分不常用的DOM树链接
        */
        app.directive(
                "bnList",
                function ($compile) {
                    /* 编译DOM模板 */
                    function compile(tElement, tAttributes) {
                        /* 当模板准备就绪,希望获取到不常用的deleteConfirmation元素 */
                        var tOverlay = tElement.find("div.deleteConfirmation")
                                        .remove()
                                ;
                        /* 现在我们提取到并需要分别编译以至能分别transcluded链接 */
                        var transcludeOverlay = $compile(tOverlay);
                        /* 在scope中绑定UI */
                        function link($scope, element, attributes) {

                            /* 这个demo中,当用户点击删除按钮时候,需要触发删除确认.
                                当用户点击starts时,我们能注入删除确认元素
                            */
                            element.on(
                                    "mousedown",
                                    "li a.delete",
                                    function (event) {
                                        var item = $(this).closest("li");
                                        var localScope = item.scope();
                                        /* 判断是否已经注入 */
                                        if (localScope.hasOwnProperty("__injected")) {
                                            return;
                                        }
                                        /* 为删除确认执行Transclude和链接DOM树 */
                                        transcludeOverlay(
                                                localScope,
                                                function (overlayClone, $scope) {
                                                    item.append(overlayClone);
                                                    $scope.__injected = true;
                                                }
                                        );
                                        /* 触发$apply 刷新界面 */
                                        localScope.$apply();

                                    }
                            );
                        }

                        return ( link );
                    }

                    return ({
                        compile: compile
                    });
                }
        );
    </script>
</body>
</html>
```

编译阶段移除`delete confirmation`元素,该元素不会被封装.此外,没有一个ng-repeat实例为删除确认进行双相绑定
当需要删除确认时,删除确认DOM对象会被引入friend对象内,在demo中`mousedown`事件能触发注入
显然这种方法前提是紧密结合的一个给定的上下文中的相应触发器。这就是说这种方法是一个优化（或优化的尝试），必然被绑定到给定的上下文。
当然，我想我能想出一些办法让它稍微通用点(可能效果不大)。
我认为这是一个非常有趣的实验。能教会我更多有关transclusion如何在AngularJS工作。

[原文](http://www.bennadel.com/blog/2754-event-delegation-performance-vs-linking-performance-in-angularjs.htm)






