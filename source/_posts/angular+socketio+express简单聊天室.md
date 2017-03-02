title: angular+socketio+express简单聊天室
date: 2014-08-11 17:59:23
tags: [node.js,io.js,angular.js,socket.io]
---

运用angular+socketio+express搭建的简单聊天室
<!-- more -->

# 简单Demo

[github源代码](https://github.com/daysv/Chat-room)

# 部分代码
## package.json	

```json
	{
	 "name": "chat",
	 "version": "0.0.1",
 	 "description": "angular+express+socketio demo",
 	 "main": "web.js",
 	 "scripts": {
  	  "start": "node ./web.js"
 	 },
 	 "dependencies": {
 	  "express": "^4.8.3",
 	  "socket.io": "^1.0.6",
  	  "static-favicon": "^2.0.0-alpha"
  	},
 	 "devDependencies": {},
 	 "author": "days_v",
	  "license": "MIT"
	}
```

# web.js
用express+socket.io搭建websocket服务
```js
	var express= require('express');
	var path = require('path');
	var favicon = require('static-favicon');
	var app=express();
	var server = require('http').Server(app);
	var io = require('socket.io')(server);
	app.use(express.static(path.join(__dirname, 'public')));

	var userList=[];

	io.on('connection', function(socket){
  	  var newUser='匿名';
 	   var time=new Date();
  	  var nowTime=time.getHours()+":"+time.getMinutes()+":"+time.getSeconds();
  	  io.sockets.emit('messageAdded',{data:'新用户进入',time:nowTime,user:''});
  	  userList.push(newUser);
  	  io.sockets.emit('newUser',userList);
   	 socket.on('changeName',function(newName){
   	     userList.splice(userList.indexOf(newUser),1);
  	     userList.push(newName);
   	     io.sockets.emit('newUser',userList);
   	     io.sockets.emit('messageAdded',{data:newUser+'改名为'+newName,time:nowTime,user:''});
         newUser=newName;
  	  });
  	  socket.on('putMessage',function(message){
   	     time=new Date();
   	     nowTime=time.getHours()+":"+time.getMinutes()+":"+time.getSeconds();
   	     io.sockets.emit('messageAdded',{data:message,time:nowTime,user:newUser});
   	 });
   	 socket.on('disconnect', function () {
   	     userList.splice(userList.indexOf(newUser),1);
    	    var time=new Date();
     	   var nowTime=time.getHours()+":"+time.getMinutes()+":"+time.getSeconds();
     	   io.sockets.emit('messageAdded',{data:newUser+'已离开',time:nowTime,user:''});
     	   io.sockets.emit('newUser',userList);
   	 });
	});
	var port = process.env.PORT || 5000;
	server.listen(port);
	console.log('success');
````
# index.html

尝试用angularjs+bootstrap构建前端

```html
    <!DOCTYPE html>
    <html>
    <head>
    <title>days_v聊天室</title>
    <link href="http://cdn.bootcss.com/bootstrap/3.2.0/css/bootstrap.css" rel="stylesheet">
    <script src="http://cdn.bootcss.com/jquery/2.1.1/jquery.min.js"></script>
    <script src="http://cdn.bootcss.com/bootstrap/3.2.0/js/bootstrap.min.js"></script>
    <script src="https://cdn.socket.io/socket.io-1.0.6.js"></script>
    <script src="http://cdn.bootcss.com/angular.js/1.3.0-beta.8/angular.min.js"></script>
    </head>
    <body ng-app="chat" style="padding-left: 5%;padding-right: 5%">
    <div class="row">
        <div class="col-sm-8">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <span class="glyphicon glyphicon-edit"></span>
                    聊天内容
                </div>
                <ul class="list-group"  ng-controller="messageCtrl" style="height: 550px;overflow-y: auto">
                    <li class="list-group-item" ng-repeat="message in messages">
                        <span class="badge">{{message.time}}</span>
                        <span class="label label-info">{{message.user}}</span>
                        {{message.data}}
                    </li>
                </ul>
            </div>
            <div class="input-group input-group-lg" ng-controller="newMessage">
                <input class="form-control" id="input-edit" placeholder="请输入聊天内容" 
                type="text" ng-model="oneMessage" ng-keypress="enter($event)">
                <span class="input-group-btn form-group">
                    <button class="btn btn-default" type="button" ng-click="newMsg()">
                        发送
                    </button>
                </span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <span class="glyphicon glyphicon-user"></span>
                    &nbsp;个人信息
                </div>
                <div class="panel-body" ng-controller="changeName">
                    <div class="row">
                        <div class="col-sm-2">姓名</div>
                        <div class="col-sm-10">
                            <input class="form-control"  ng-model="name">
                            <button type='button' class="btn btn-default" ng-click="reName()">确认</button>
                        </div>
                    </div>
                </div>
            </div>
            <div class="panel panel-default" ng-controller="nowUser">
                <div class="panel-heading">
                    <span class="glyphicon glyphicon-list"></span>
                    &nbsp;在线名单
                </div>
                <div class="panel-body list-body" style="padding: 0;height: 400px;overflow-y: auto">
                    <table class="table table-hover list-table" >
                        <tbody>
                        <tr  ng-repeat="user in users track by $id($index)"><td>{{user}}</td></tr>
                        </tbody>
                    </table>
                </div>
                <div class="panel-footer" id="list-count">当前在线：{{number}}人</div>
            </div>
        </div>
    </div>
    <script>
        angular.module('chat',[]);
        angular.module('chat').factory('socket',function($rootScope){
            var socket=io.connect('/');
            return{
                on: function (eventName, callback) {
                    socket.on(eventName, function () {
                        var args = arguments;
                        $rootScope.$apply(function () {
                            callback.apply(socket, args)
                        })
                    })
                },
                emit: function (eventName, data, callback) {
                    socket.emit(eventName, data, function () {
                        var args = arguments;
                        $rootScope.$apply(function () {
                            if (callback) {
                                callback.apply(socket, args);
                            }
                        })
                    })
                }
            }
        });
        angular.module('chat').controller('messageCtrl',function($scope,socket){
            $scope.messages=[];
            socket.on('messageAdded',function(msg){
                $scope.messages.push(msg);
            });
        });
        angular.module('chat').controller('newMessage',function($scope,socket){
            $scope.enter = function(ev) {
                if (ev.keyCode !== 13) return;
                if($scope.oneMessage!='') {
                    socket.emit('putMessage',$scope.oneMessage);
                    $scope.oneMessage='';
                }
                else{
                    return;
                }
            }
            $scope.newMsg=function(){
                if($scope.oneMessage!='') {
                    socket.emit('putMessage',$scope.oneMessage);
                    $scope.oneMessage='';
                }
                else{
                    return;
                }
            }
        });
        angular.module('chat').controller('changeName',function($scope,socket){
            $scope.reName=function(){
                if($scope.name!=''&&typeof ($scope.name)!="undefined") {
                    socket.emit('changeName',$scope.name);
                }
                else{
                    return;
                }
            }
        });
        angular.module('chat').controller('nowUser',function($scope,socket){
            $scope.users=[];
            $scope.number=0;
            socket.on('newUser',function(user){
                $scope.users=user;
                $scope.number=$scope.users.length;
            });
        });
    </script>
    </body>
    </html>
    
```

简单的代码就能马上实现聊天室功能,angular与平常与jquery的思维不同,需要提前做好框架减少DOM的操作.