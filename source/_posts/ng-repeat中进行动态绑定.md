title: ng-repeat中的性能优化
date: 2015-05-16 20:45:20
tags: [angular.js]
---

一般情况下Angular.js的性能都能满足基本需求,但是随着双向绑定的数量上升,会逐渐产生性能问题.
但是在项目中使用`ng-repeat`经常遇到一些不可控的情况发生,使得双向绑定呈数量级上升.
这是因为Angular的双向绑定机制主要依靠digest进行脏数据检查,所以watcher超过一定数量,其对性能的影响便逐渐显现,在其文档中也对绑定数量过多产生会产生性能问题进行了提示(一般把watcher的数量控制在2000个以内比较适合).

<!-- more -->

稍微看看`Angular.js`相关源码

```js

      $digest: function() {
        var watch, value, last,
            watchers,
            length,
            dirty, ttl = TTL,
            next, current, target = this,
            watchLog = [],
            logIdx, logMsg, asyncTask;

        beginPhase('$digest');
        // Check for changes to browser url that happened in sync before the call to $digest
        $browser.$$checkUrlChange();

        if (this === $rootScope && applyAsyncId !== null) {
          // If this is the root scope, and $applyAsync has scheduled a deferred $apply(), then
          // cancel the scheduled $apply and flush the queue of expressions to be evaluated.
          $browser.defer.cancel(applyAsyncId);
          flushApplyAsync();
        }

        lastDirtyWatch = null;

        do { // "while dirty" loop
          dirty = false;
          current = target;

          while (asyncQueue.length) {
            try {
              asyncTask = asyncQueue.shift();
              asyncTask.scope.$eval(asyncTask.expression, asyncTask.locals);
            } catch (e) {
              $exceptionHandler(e);
            }
            lastDirtyWatch = null;
          }

          traverseScopesLoop:
          do { // "traverse the scopes" loop
            if ((watchers = current.$$watchers)) {
              // process our watches
              length = watchers.length;
              while (length--) {
                try {
                  watch = watchers[length];
                  // Most common watches are on primitives, in which case we can short
                  // circuit it with === operator, only when === fails do we use .equals
                  if (watch) {
                    if ((value = watch.get(current)) !== (last = watch.last) &&
                        !(watch.eq
                            ? equals(value, last)
                            : (typeof value === 'number' && typeof last === 'number'
                               && isNaN(value) && isNaN(last)))) {
                      dirty = true;
                      lastDirtyWatch = watch;
                      watch.last = watch.eq ? copy(value, null) : value;
                      watch.fn(value, ((last === initWatchVal) ? value : last), current);
                      if (ttl < 5) {
                        logIdx = 4 - ttl;
                        if (!watchLog[logIdx]) watchLog[logIdx] = [];
                        watchLog[logIdx].push({
                          msg: isFunction(watch.exp) ? 'fn: ' + (watch.exp.name || watch.exp.toString()) : watch.exp,
                          newVal: value,
                          oldVal: last
                        });
                      }
                    } else if (watch === lastDirtyWatch) {
                      // If the most recently dirty watcher is now clean, short circuit since the remaining watchers
                      // have already been tested.
                      dirty = false;
                      break traverseScopesLoop;
                    }
                  }
                } catch (e) {
                  $exceptionHandler(e);
                }
              }
            }

            // Insanity Warning: scope depth-first traversal
            // yes, this code is a bit crazy, but it works and we have tests to prove it!
            // this piece should be kept in sync with the traversal in $broadcast
            if (!(next = ((current.$$watchersCount && current.$$childHead) ||
                (current !== target && current.$$nextSibling)))) {
              while (current !== target && !(next = current.$$nextSibling)) {
                current = current.$parent;
              }
            }
          } while ((current = next));

          // `break traverseScopesLoop;` takes us to here

          if ((dirty || asyncQueue.length) && !(ttl--)) {
            clearPhase();
            throw $rootScopeMinErr('infdig',
                '{0} $digest() iterations reached. Aborting!\n' +
                'Watchers fired in the last 5 iterations: {1}',
                TTL, watchLog);
          }

        } while (dirty || asyncQueue.length);

        clearPhase();

        while (postDigestQueue.length) {
          try {
            postDigestQueue.shift()();
          } catch (e) {
            $exceptionHandler(e);
          }
        }
      },

```

很明显,`$digest`是用了个三重大循环深度优先遍历来实现整个`Angular`的核心脏数据检查,以至于源码里都自带了吐槽. `yes, this code is a bit crazy, but it works and we have tests to prove it!`
那么为了解决性能问题最好的入手办法无外乎就是减少双向绑定的数量.

在Angular的1.3版本中引入了一次性数据绑定`one-time bindings`来解决双向绑定过多的性能问题,使得我们可以通过`::`来对数据进行一次性的填充,对于不会改变的数据我们可以统统运用::来进行绑定.

但在项目中经常会遇到需要列表进行动态排序并且列数量巨大,这时候的思路是通过将不显示的元素不进行渲染来大大减少双向绑定的数量,同时依据滚动条来进行动态增删保证数据绑定连贯性.
下面是简单的思路实现列表动态加载

```js

angular.module('Scroll', [])
    .controller('MainController', function ($scope) {
        $scope.list = []
        for (var i = 0; i < 100; i++) {
            $scope.list.push(i)
        }
    })
    .directive('repeat', function () {
        return {
            restrict: "A",
            link: function (scope, element, attrs) {
                var scrollTop, index;
                scope.first = 0;
                scope.last = 30;
                // 监听滚动事件动态限制渲染
                element.on('scroll', function () {
                    scrollTop = element[0].scrollTop;
                    index = scrollTop / 20;
                    scope.first = index - 10 > 0 ? index - 10 : 0;
                    scope.last = index + 30 < scope.list.length ? index + 30 : scope.list.length;
                    scope.$digest()
                })
            },
            template: ' <li ng-repeat="item in list"><span ng-if="$index <= last && $index >=first">{{item}}</span></li>'
        }
    })

```

```html

<!DOCTYPE html>
<html ng-app="Scroll">
<head lang="en">
    <meta charset="UTF-8">
    <title></title>
    <script src="//cdn.bootcss.com/angular.js/1.4.2/angular.js"></script>
    <script src="main.js"></script>
    <style>
        .repeat-list{
            height: 400px;
            overflow-y:auto
        }
        .repeat-list li{
            height: 20px;;
        }
    </style>
</head>
<body ng-controller="MainController">
    <ul repeat class="repeat-list" >
    </ul>
</body>
</html>

```

不过这样做有弊端,其中的`<li>`依然会进行渲染.
所以最好进行数据分离,可以为`repeat`的第一个元素和最后一个元素设置动态的高度,用于滚动条的控制.
最后再监听`scroll`通过双向绑定的方式控制显示的内容

github中已有相似库 https://github.com/kamilkp/angular-vs-repeat 能够快速解决因过多的ng-repeat 造成的性能问题

使用`angular-vs-repeat` 十分简单,引入类库并在angular中注册,在`ng-repeat`外面嵌套一层`div`并设置` vs-repeat`参数即可
```
<div vs-repeat>
    <div ng-repeat="item in someArray">
        <!-- content -->
    </div>
</div>

```

## 注意
使用`angular-vs-repeat` 时,数据必须为数组
所有元素必须已知有相同高度或宽度
还有使用时候注意css可能会影响指令生效


## 在React中的简单实现
在React内通过几行代码就可以轻松实现渲染所需列表数据

```js
import React, { Component } from 'react'
import ReactDOM from 'react-dom'

class ListRepeat extends Component {
    constructor(props) {
        super(props);
        this.state = {
            scrollTop: 0
        }
    }

    scrolling() {
        this.setState({scrollTop: ReactDOM.findDOMNode(this).scrollTop})
    }

    render() {
        let start = ~~(this.state.scrollTop / 30);
        const createItem = (item, index) => {
            if (start <= index && index <= start + 10) {
                return <li style={{height:'30px'}} key={index}>{item}</li>
            } else {
                return false;
            }
        }
        const style = {
            height: this.props.items.length * 30 - this.state.scrollTop + this.state.scrollTop % 30,
            paddingTop: start * 30
        }
        const warpStyle = {
            height: '300px',
            overflow: 'auto',
            margin: 0
        }
        return <div style={warpStyle} onScroll={this.scrolling.bind(this)}>
            <ul style={style}>
                {this.props.items.map(createItem)}
            </ul>
        </div>
    }
}

class TodoApp extends Component {
    constructor(props) {
        super(props);
        var arr = []
        for (let i = 0; i < 1000; i++) {
            arr.push(i)
        }
        this.state = {items: arr}
    }

    render() {
        return <ListRepeat items={this.state.items}/>
    }
}

ReactDOM.render(
    <TodoApp />,
    document.getElementById('example')
);

```