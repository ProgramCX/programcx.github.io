---
title: Uncaught TypeError:xxx.component is not a function
description: Vue 3 报错Uncaught TypeError:xxx.component is not a function
date: 2024-10-15
image: vue.svg
categories:
    - 前端
---

# Vue 3 报错Uncaught TypeError:xxx.component is not a function

**记录学习Vue 3过程中踩的坑**
## 一、错误代码
有以下代码：

```javascript
const vueWatcher = {
    methods: {
        click(index) {
            if (index == 0) {
                alert("外层");
            } else if (index == 1) {
                alert("中层");
            } else if (index == 2) {
                alert("内层");
            }
        },
        KeyDownCtrl() {
            alert("Ctrl Pressed");
        }
    }
}
const app = Vue.createApp(vueWatcher).mount("#vue-watcher");
const alertComponent = {
    data() {
        return {
            msg: "警告",
        }
    },
    methods: {
        click() {
            alert(this.msg);
        }
    },
    template: '<div><button @click="click">按钮</button></div>'
}
app.component("alert-component", alertComponent);
```
我们是想要给 ``app``对象添加一个标准的Vue组件``alert-component``，但是浏览器报错``Uncaught TypeError:app.component is not a function``。
## 二、分析原因
其实这是一个添加子组件顺序的问题。上述错误代码中，先创建一个名为``app``的Vue对象，接着挂载到``vue-watcher``上，最后添加子组件``alert-componet``。
这种顺序是错误的，正确的顺序应该是：

 1. 创建一个名为``app``的Vue对象；
 

```javascript
const app = Vue.createApp(vueWatcher);
```

 3. 添加子组件``alert-componet``；
 

```javascript
app.component("alert-component", alertComponent);
```

 5. 挂载到``vue-watcher``上。

```javascript
app.mount("#vue-watcher");
```

 
 ## 三、正确代码

	

```javascript
const vueWatcher = {
    methods: {
        click(index) {
            if (index == 0) {
                alert("外层");
            } else if (index == 1) {
                alert("中层");
            } else if (index == 2) {
                alert("内层");
            }
        },
        KeyDownCtrl() {
            alert("Ctrl Pressed");
        }
    }
}
const app = Vue.createApp(vueWatcher);
const alertComponent = {
    data() {
        return {
            msg: "警告",
        }
    },
    methods: {
        click() {
            alert(this.msg);
        }
    },
    template: '<div><button @click="click">按钮</button></div>'
}
app.component("my", alertComponent);
app.mount("#vue-watcher");
```
*欢迎指正错误！*