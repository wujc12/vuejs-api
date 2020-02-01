# Vue.extend( options )

## Vue.extend(options) 官方文档
可通过[官方API文档链接](https://cn.vuejs.org/v2/api/#Vue-extend)访问官方API文档。

- 参数：``options: Object``
- 用法：使用基础的Vue构造函数，创建一个“子类”。参数是一个包含组件选项的对象，需要注意的是，组件选项中的``data``选项必须为函数。
- 示例：
```html
<div id="mount-point"></div>
```
```js
let ChildComponent = Vue.extend({
    name: 'ChildComponent',
    template: `<p>{{msg}}</p>`,
    data () {
        return {
            msg: 'Hello World from Child Component!'
        }
    }
})

new ChildComponent().$mount('#mount-point')
```
返回结果：

```html
<p>Hello World from Child Component!</p>
```

## 源码解析
下面是``Vue.extend()``的源码
```js
// Vue 根组件（通过new Vue()）定义的cid为0
Vue.cid = 0;
// 全局变量，当前构造函数的编号
let cid = 1;

Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {};
  const Super = this;
  const SuperId = Super.cid;
  // 如果
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }

  const name = extendOptions.name || Super.options.name;
  if (process.env.NODE_ENV !== 'production' && name) {
    validateComponentName(name)
  }

  // 定义VueComponent构造器
  const Sub = function VueComponent (options) {
    this._init(options);
  };

  // 由于最开始必然是从Vue继承的，因此Sub.prototype = Super.prototype = Vue.prototype
  // 因此继承了一系列预先定义的实例方法和实例的属性
  Sub.prototype = Object.create(Super.prototype);
  Sub.prototype.constructor = Sub;
  // 将全局的cid赋值给新的构造器后全局cid自增
  Sub.cid = cid++;
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  );
  Sub['super'] = Super;

  // For props and computed properties, we define the proxy getters on
  // the Vue instances at extension time, on the extended prototype. This
  // avoids Object.defineProperty calls for each instance created.
  if (Sub.options.props) {
    initProps(Sub)
  }
  if (Sub.options.computed) {
    initComputed(Sub)
  }

  // allow further extension/mixin/plugin usage
  Sub.extend = Super.extend;
  Sub.mixin = Super.mixin;
  Sub.use = Super.use;

  // create asset registers, so extended classes
  // can have their private assets too.
  ASSET_TYPES.forEach(function (type) {
    Sub[type] = Super[type]
  });
  // enable recursive self-lookup
  if (name) {
    Sub.options.components[name] = Sub
  }

  // keep a reference to the super options at extension time.
  // later at instantiation we can check if Super's options have
  // been updated.
  Sub.superOptions = Super.options;
  Sub.extendOptions = extendOptions;
  Sub.sealedOptions = extend({}, Sub.options);

  // cache constructor
  cachedCtors[SuperId] = Sub;
  return Sub
}

function initProps (Comp) {
  const props = Comp.options.props;
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed (Comp) {
  const computed = Comp.options.computed;
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}
```
``Vue.extend()``函数接收的参数为创建组件用的``options``对象，并返回一个组件构造器。

首先，在``Vue.extend``函数之外，定义了``cid``全局变量，并且将Vue根构造函数的``cid``值设置为 0；此外，之后每创建一个组件构造器，构造器会持有一个不断增加``cid``，从而保证了每个组件构造器具有唯一的``cid``值。

我们在顺序阅读代码之前，先注意观察到父构造器的全局方法``extend``、``mixin``和``use``方法会被赋给子构造器，以下面的代码为例：
```js
let ChildComponent = Vue.extend(childOptions);
let GrandSonComponent = ChildComponent.extend(grandSonOptions);
console.log(ChildComponent.extend === Vue.extend);  // true
console.log(GrandSonComponent.extend === Vue.extend);  // true
```

因此实际上，调用``extend``方法的可能是``Vue``构造器，同样也可能是其他的子构造器（如上面代码的``ChildComponent``）！

由于一个组件可能在项目的多个地方复用，为了防止重复生成组件构造器，可以看到构造器在创建过程中，会给**首次**输入的``extendOptions``对象创建一个``_Ctor``属性，并指向缓存对象``cachedCtors``，该对象具有以下“键值对”形式：
```js
cachedCtors = {
    SuperId1: VueComponent1
    SuperId2: VueComponent2
}
```
其中，``SuperId``指的是继承自的父组件的``cid``，``cachedCtors``缓存不同的``SuperId``是因为同一份用于创建组件的``options``可能会被不同组件构造器调用``extend``方法生成不同的组件构造器！之后，如果查询到``SuperId``在``cachedCtors``已有定义后，直接返回已经定义好的构造器即可，避免了重复创建构造器带来的性能开销。

如果是新建的组件构造器，给该构造器添加``cid``并使``cid``增加，再通过``Object.create()``方法，以父构造器的``prototype``为原型创建当前构造器的``prototype``，从而使子构造器生成的实例具有和父构造器生成实例具有一样的实例属性及实例方法。

后续则是``options``的合并，会将父构造器的全局``options``与输入的``extendOptions``合并并赋值给当前子构造器的``options``全局对象。

完成选项合并后，会将``extendOptions``的``props``属性代理到子组件的``prototype``上，这也是为什么能在组件内通过``this.propKey``访问key值为``propKey``的属性的原因。之后``initComputed``会给每个计算属性创建一个``watcher``。

``props``属性和计算属性初始化后，对父构造器的``ASSET_TYPES``（``components, filters, directives``）属性赋值给子构造器。此外，如果定义了组件名称``name``。

最后，完成了多个``options``的备份后，会将当前创建好的子构造器缓存到``cachedCtors``中，并返回新创建好的子构造器，自此``extend``方法结束。