# Vue.component()，Vue.filter()和Vue.directive()

三个全局API的输入参数相同：
- 参数：
    - ``id: String``
    - ``definition: Function | Object``

可以通过访问[官方文档](https://cn.vuejs.org/v2/api/#Vue-directive)查看3个全局API的官方文档。

## Vue.component() 官方API文档
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

## Vue.filter() 官方API文档
- 用法：
注册或获取全局过滤器。
```js
// 注册
Vue.filter('my-filter', function (value) {
  // 返回处理后的值
})
// getter，返回已注册的过滤器
var myFilter = Vue.filter('my-filter')
```

## Vue.directive() 官方API文档
- 用法：
注册或获取全局指令。
```js
// 注册
Vue.directive('my-directive', {
  bind: function () {},
  inserted: function () {},
  update: function () {},
  componentUpdated: function () {},
  unbind: function () {}
})

// 注册 (指令函数)
Vue.directive('my-directive', function () {
  // 这里将会被 `bind` 和 `update` 调用
})

// getter，返回已注册的指令
var myDirective = Vue.directive('my-directive')
```

## Vue.component()，Vue.directive()和Vue.filter() 源码解析

上述全局API的格式一致，且功能类似，实际上他们的源代码一致，定义如下：
```js
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production' && type === 'component') {
          validateComponentName(id)
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id;
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        this.options[type + 's'][id] = definition;
        return definition
      }
    }
  })
}
```

上面代码中，``ASSET_TYPES``的定义为：
```js
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
];
```
因此，上述``type``值可以为``component``，``directive``或``filter``中的任一个。此外，由于函数的调用者为``Vue``构造器，因此**函数中的``this``则为``Vue``构造器**。

函数可以接收两种类型的参数：
1. 只接受``id``作为参数，此时那么返回已经定义好的全局资源（组件，过滤器或指令）；
2. 同时接受``id``和``definition``作为参数，此时会定义一个新的全局资源（组件，过滤器或指令）。

我们首先分析第一种情况。``return this.options[type + 's'][id]``，也就是说如果定义了标识为``id``的资源，则返回已定义值。

下面分析第二种情况。首先如果type值为``component``，也即通过``Vue.component()``定义全局组件，会首先判断输入的``id``是否为合法的组件名。之后，通过``this.options._base.extend(definition)``继承生成组件构造器，需要注意的是，``this.options._base``实际上就是指向``Vue``构造器的引用。``Vue.extend()``的过程在[Vue.extend](./Vue.extend.md)一文有详细的介绍，感兴趣的读者可以阅读。

如果type值为``directive``则是将输入的``definition``进行如下赋值：
```js
definition = { bind: definition, update: definition }
```

最后，所有的资源（component，directive或filter）均会赋值到``this.options[type + 's'][id]``中，因此可以通过``Vue.options.components[componentId]``这样的形式访问已经定义好的子组件构造器！

特别需要注意的是，我们回顾下[Vue.extend](./Vue.extend.md)一文中的合并选项部分：
```js
Sub.options = mergeOptions(
  Super.options,
  extendOptions
);
```
可以发现，基于父构造器创建子构造器的过程中，会将父构造器的``options``属性合并到子构造器的``options``中，而``Vue``构造器是所有组件构造器的父（祖先）构造器，因此通过``Vue.component(), Vue.directive(), 和Vue.filter()``注册后，所有其他组件构造器均能访问到注册后的资源，从而实现了全局注册的效果！
