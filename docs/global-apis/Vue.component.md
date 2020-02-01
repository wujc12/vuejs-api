# Vue.component(id, [definition])

## Vue.component() 官方API文档

点击[官方API文档](https://cn.vuejs.org/v2/api/#Vue-component)查看官方文档。

- 参数：
    - ``id: String``
    - ``definition: Function | Object``
- 用法：
注册或获取全局组件。注册还会自动使用给定的id设置组件的名称。
```js
// 注册组件，传入一个扩展过的构造器
Vue.component('my-component', Vue.extend({ /* ... */ }))

// 注册组件，传入一个选项对象 (自动调用 Vue.extend)
Vue.component('my-component', { /* ... */ })

// 获取注册的组件 (始终返回构造器)
var MyComponent = Vue.component('my-component')
```