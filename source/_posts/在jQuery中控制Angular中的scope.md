title: 在jQuery中控制Angular中的scope
date: 2015-08-01 11:50:27
tags: [angular.js,jQuery.js]
---


由于大部分Angular项目中还是使用jQuery作为补充，jQuery作为最流行的类库在其基础上开发了众多的组件。
在Angular架构之外经常会伴随着其他模块，这些模块时常需要获取或控制Angular中看控制器的域。
在angular前引用jQuery，其会替换Angular中自带的jQlite，我们可以在jQuery中通过.scope()方法获取当前选择器内容里继承的域。

<!-- more -->

下面是简单的例子，通过在外部获取Angular控制器中的域，并执行相关方法。

```html

<!DOCTYPE html>
<html lang="en" ng-app="app">
<head>
    <meta charset="UTF-8">
    <title>Get angular's scope in jQuery</title>
    <script src="http://cdn.bootcss.com/jquery/2.1.4/jquery.js"></script>
    <script src="http://cdn.bootcss.com/angular.js/1.4.3/angular.js"></script>
    <script>
        angular.module('app',[])
                .controller('listController',['$scope', function ($scope) {
                    $scope.list = [1,2,3,4,5];
                    $scope.test = function () {
                        console.log('test');
                    }
                }])
    </script>
    <script>
        $(document).on('ready', function () {
            var controllerScope = $('div[ng-controller="listController"]').scope();  // Get controller's scope
            controllerScope.test(); // log 'test'
            console.log(controllerScope.list); // log [1,2,3,4,5]
            $('button').click(function (e) {
                var scope = $(e.target).scope();
                console.log(scope.item) // log item number
                scope.test(); // log 'test'
            })
        })
    </script>
</head>
<body>
<div ng-controller="listController">
    <ul>
        <li ng-repeat="item in list"><button>Select {{item}}</button></li>
    </ul>
</div>
</body>
</html>

```