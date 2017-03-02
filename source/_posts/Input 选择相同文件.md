title: Input 选择相同文件
date: 2017-02-24 00:44:52
tags: [tips]
---

使得`type='file'`的input标签可以多次选择相同文件

<!-- more -->

```js
var target = document.getElementById('files')

target.addEventListener('change', selected)

function selected() {
    // todo

    var obj = new File('', '');
    var list = new FileList();
    list.append(obj);

    target.removeEventListener('change', selected)
    target.files = list
    target.addEventListener('change', selected)
}


```