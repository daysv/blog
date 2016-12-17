title: 感受下React
date: 2016-07-17 19:27:25
tags: [react.js]
---

依据业务需要用React写了个简单的树组件,写起来还是比较顺手的,以后再根据业务需求慢慢扩展吧

<!-- more -->

## index.js
```js
    import React, {Component, PropTypes, Perf} from 'react'
    import {VelocityTransitionGroup} from 'velocity-react'
    import css from './styles.css'

    class STree extends Component {
        constructor(props) {
            super(props)
        }

        render() {
            const ul = (this.props.hide || !this.props.data.length) ? '' :
                <ul className={ (this.props.isChild ? '' : css.root + ' ') + css.nodes }>
                    {
                        this.props.data.map((v, i)=> {
                            const id = this.props.id.concat(i)
                            return <Item id={id} key={id.join('.')} setting={this.props.setting} value={v}
                                         children={this.props.children}/>
                        })
                    }
                </ul>
            return (
                <VelocityTransitionGroup enter={{animation: "slideDown", duration: 250}}
                                         leave={{animation: "slideUp", duration: 250}}>
                    {ul}
                </VelocityTransitionGroup>
            )
        }
    }
    STree.defaultProps = {
        id: [],
        data: [],
        hide: false,
        setting: {
            onClick: ()=> {
            }
        }
    }
    STree.propTypes = {
        id: React.PropTypes.array,
        data: React.PropTypes.array.isRequired,
        hide: React.PropTypes.bool,
        isChild: React.PropTypes.bool,
        setting: React.PropTypes.shape({
            onClick: React.PropTypes.func.isRequired
        })
    }

    class Item extends Component {
        constructor(props) {
            super(props)
            this.state = {
                hide: this.props.value.hasOwnProperty('hide') ? this.props.value.hide : true,
                nodes: this.props.value.nodes
            }
        }

        getTree() {
            return Object.assign({}, this.props.value, {
                _id: this.props.id, _nodes: this.state.nodes
            })
        }

        setChildNodes(nodes, callback) {
            this.setState({
                nodes
            }, callback)
        }

        onClick() {
            this.props.setting.onClick.call(null, this.getTree(), this.setChildNodes.bind(this))
            this.setState({
                hide: !this.state.hide
            })
        }

        render() {
            const children = React.Children.map(this.props.children, child => {
                return React.cloneElement(child, {
                    onClick: child.props.onClick.bind(null, this.getTree(), this.setChildNodes.bind(this))
                })
            })

            return (
                <li className={css.item}>
                    <div>
                        { !this.state.nodes ?
                            <img onClick={this.onClick.bind(this)} src={this.props.value.img}/> :
                            <icon onClick={this.onClick.bind(this)} className={(this.state.hide? '' : css.open)}/> }
                        <span onClick={this.onClick.bind(this)}>{this.props.value.name}</span>
                        { children }
                    </div>
                    <STree hide={this.state.hide} id={this.props.id} setting={this.props.setting}
                           data={this.state.nodes} isChild={true} children={this.props.children}/>
                </li>
            )
        }
    }

    Item.propTypes = {
        id: React.PropTypes.array.isRequired,
        value: React.PropTypes.shape({
            hide: React.PropTypes.bool,
            img: React.PropTypes.string,
            nodes: React.PropTypes.array
        }),
        setting: React.PropTypes.shape({
            onClick: React.PropTypes.func.isRequired
        })
    }

    export default STree

```

## style.css
```css
:local .root {
    padding: 4px !important;
    font-family: Microsoft YaHei, Verdana, Arial, Helvetica, AppleGothic, sans-serif;
    font-size: 1em;
    -webkit-user-select: none;
    color: #333;
}

:local .nodes {
    list-style-type: none;
    margin: 0;
    padding: 0 0 0 18px;
}

:local .nodes > li {
    list-style-type: none;
    margin: 0;
    padding: 0;
}

:local .nodes > li > div {
    display: inline-flex;
    line-height: 32px;
}

:local .nodes > li > div > img {
    width: 25px;
    height: 25px;
    border-radius: 3px;
    vertical-align: middle;
    margin: auto;
    cursor: pointer;
}

:local .nodes > li > div > span {
    flex: 1;
    padding: 3px;
    margin: 0 3px;
    cursor: pointer;
}

:local .nodes > li > div > span:hover {
    background-color: rgb(245, 245, 245);
}

:local .nodes > li > div > icon {
    cursor: pointer;
    display: inline-block;
    border-top: 5px solid transparent;
    border-left: 8px solid rgb(167, 179, 191);
    border-bottom: 5px solid transparent;
    border-radius: 4px;
    width: 0;
    height: 0;
    margin: auto;
}

:local .open {
    transform: rotate(90deg);
}
```

## 使用
```js
import STree from './index'
import ReactDOM from 'react-dom'
import React, {Component, PropTypes} from 'react'

var Example = React.createClass({
    getInitialState: function () {
        var nodes = []

        for (var i = 0; i < 5; i++) {
            nodes.push({id: i, name: "父节点" + i, nodes: []})
        }

        var arr = []
        for (var j = 0; j < 10; j++) {
            if (j < 5) {
                arr.push({name: 'child', img: '//www.baidu.com/img/bd_logo1.png'})
            } else {
                arr.push({name: 'child', nodes: []})
            }
        }

        const setting = {
            onClick: (obj, setChildNodes)=> {
            // obj 只可获取一层子节点
                if (!!obj.nodes) {
                    var d = new Date()
                    setChildNodes(arr, ()=> {
                        console.log(new Date() - d + 'ms')
                    })
                }
            }
        };
        return {
            nodes,
            setting
        }
    },

    onClick: function (obj, setState) {
        console.log(obj)
    },

    render: function () {
        return (
            <STree setting={this.state.setting} data={this.state.nodes}>
                <button onClick={this.onClick}>click</button>
            </STree>
        );
    }
});


ReactDOM.render(<Example />, document.getElementById('root'));
```