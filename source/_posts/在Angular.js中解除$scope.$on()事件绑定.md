title: 在Angular.js中解除$scope.$on()事件绑定
date: 2015-05-04 19:45:20
tags: [angular.js]
---

现从事于nw.js (node-webkit)开发,桌面客户端与浏览器客户端开发还是有较大的区别,所以常会遇到一些小麻烦
大多数时候我们不需要手动解除`Scope.$on()` 事件,因为当Scope销毁时Angular会隐性解除事件.但是在大的项目中,过多的事件监听对性能产生影响时,我们需要手动解除事件绑定.
在jQuery中我们是用.off()来对事件进行解绑.而Angular中则不能直接解除.下面是一个简单实例展示如何控制$scope.$on

<!-- more -->

父级appcontroller中对scope进行事件`ping`进行`$broadcast`广播,页面两侧均为子控制器EventController对`ping`事件进行监听,通过每次事件触发增加数值.
toggleListener方法对事件进行手动绑定及解绑
可以看出通过对变量 unbindHandler 进行赋值,使得操作unbindHandler对事件进行管理
其中主要通过 `unbindHandler()` 和 `unbindHandler = null` 进行解绑.

```html
<!doctype html>
<html ng-app="Demo">
<head>
    <meta charset="utf-8"/>

    <title>
        Unbinding Scope.$on() Event Handlers In AngularJS
    </title>


</head>
<body ng-controller="AppController">

<h1>
    Unbinding Scope.$on() Event Handlers In AngularJS
</h1>

<div
        ng-controller="EventController"
        ng-click="toggleListener()"
        class="event-target left"
        ng-class="{ active: isWatchingEvent }">

    {{ eventCount }}

</div>

<div
        ng-controller="EventController"
        ng-click="toggleListener()"
        class="event-target right"
        ng-class="{ active: isWatchingEvent }">

    {{ eventCount }}

</div>


<!-- Load scripts. -->
<script type="text/javascript" src="http://cdn.bootcss.com/jquery/2.1.4/jquery.min.js"></script>
<script type="text/javascript" src="http://cdn.bootcss.com/angular.js/1.3.15/angular.js"></script>
<script type="text/javascript">


    var app = angular.module("Demo", []);

 
    app.controller(
            "AppController",
            function ($scope, $interval) {

                $interval(
                        function handleInterval() {
                            $scope.$broadcast("ping");
                        },
                        200
                );
            }
    );

 
    app.controller(
            "EventController",
            function ($scope) {
                $scope.eventCount = 0;

                $scope.isWatchingEvent = false;

                var unbindHandler = null;

                startWatchingEvent();
 
                $scope.toggleListener = function () {

                    unbindHandler
                            ? stopWatchingEvent()
                            : startWatchingEvent()
                    ;

                };

 

                function handlePingEvent(event) {
                    $scope.eventCount++;
                }

                function startWatchingEvent() {
                    unbindHandler = $scope.$on("ping", handlePingEvent);
                    $scope.isWatchingEvent = true;
                }

                function stopWatchingEvent() {
                    unbindHandler();
                    unbindHandler = null;
                    $scope.isWatchingEvent = false;

                }

            }
    );

</script>
<style>

    div.event-target {
        background-color: #FAFAFA;
        border: 1px solid #CCCCCC;
        border-radius: 4px 4px 4px 4px;
        color: #CCCCCC;
        cursor: pointer;
        font-size: 36px;
        font-weight: bold;
        height: 100px;
        line-height: 100px;
        text-align: center;
        width: 48%;
    }

    div.event-target.active {
        border-color: #FF33CC;
        color: #333333;
    }

    div.event-target.left {
        float: left;
    }

    div.event-target.right {
        float: right;
    }

</style>
</body>
</html>

```

