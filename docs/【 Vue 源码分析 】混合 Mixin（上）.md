## 回顾一下 Mixin

相信大家对 Mixin 并不陌生，那它究竟是有何用处呢？官方给出了解释：

> 混入 (mixin) 提供了一种非常灵活的方式，来分发 Vue 组件中的可复用功能。

这段话在理解上很简单，无非就是将每个组件中共同功能抽离出来，而所有共同功能所有组成的一个对象就会被作为一个 Mixin 处理。我们看下官方给出的简单栗子🌰，这样会比较好滴理解上述的解释：

```javascript
// 定义一个混入对象
var myMixin = {
  created: function () {
    this.hello()
  },
  methods: {
    hello: function () {
      console.log('hello from mixin!')
    }
  }
}
// 定义一个使用混入对象的组件
var Component = Vue.extend({
  mixins: [myMixin]
})
var component = new Component() // => "hello from mixin!"
```

当然抽离复用的功能是 Mixin 的关注的重点，但它却不是 Mixin 的处理核心。而 **Mixin 的处理核心就是选项合并。**选项合并如何理解？

简单来讲，那就是将重名冲突的属性或方法进行处理，是选择合并起来还是进行覆盖。官方文档也有提及，针对不同的情况进行不同的处理，主要有以下方面：

- Data

  新版 Vue 支持了 data 中直接赋于对象形式，若 data 为 Function 类型，则会先调用该函数，再进行重名合并，若 data 为 Obejct 类型，则会直接进行重名合并。如何进行重名合并？优先级如下：

  ```javascript
  组件data > 组件mixin > 组件mixin的mixin > ... > 全局mixin
  ```

- Props \ Methods \ Inject \ Computed

  处理上跟 data 为对象的形式类似，优先级如下：

  ```javascript
  组件 > 组件mixin > 组件mixin的mixin > ... > 全局mixin
  ```

- Watch

  监听在处理上并不是简单滴将同名属性监听进行覆盖，而是将整个 watch 对象收集起来存到一个数组中，当一个监听的属性发生变更时，调用的优先级如下：

  ```javascript
  全局mixin > ... > 组件mixin的mixin > 组件mixin > 组件
  ```

- Lifecycle

  生命周期在处理上跟 watch 很类似，不过它是针对每个生命周期函数进行收集并存放到数组中。同名的生命周期函数调用顺序如下：

  ```javascript
  全局mixin > ... > 组件mixin的mixin > 组件mixin > 组件
  ```

- Component \ Directive \ Filter

  该三项的处理有点特殊，利用了继承的方式来实现合并，使用原型链访问的形式，当在一个组件中找不到 component、diirective、fiter 时，会沿着原型链向上寻找，优先级如下：

  ```javascript
  组件 > 组件mixin > 组件mixin的mixin > ... > 全局mixin
  ```

对于混合中要合并的选项上面都已经进行简单的阐述，可以看到的是，针对不同情况，都会使用不同的方式进行实现，特别是使用原型合并的方式用的最为精妙，利用原型链访问的原理进行查询，而这也为平时的开发中增加了一种新的认识。那么接下来我们就来看看源码对于上面不同的情况是如何实现的。🤔



## 先从源码角度进行分析 Mixin 的入口

相信我们都知道有`Vue.mixin`全局定义混合这个 API，**一般情况下，全局定义 mixin （即`Vue.mixin`）必须要在初始化 Vue 之前**。既然要用到这个 API，那也得暴露出来给我们使用对吧？好，那么就来看看是如何暴露的。

```javascript
function initGlobalAPI (Vue) {
  // ...
  initMixin$1(Vue);
  // ...
}
initGlobalAPI(Vue)

function initMixin$1 (Vue) { // 初始化mixin的API
  Vue.mixin = function (mixin) {
    this.options = mergeOptions(this.options, mixin); // 开始合并选项
    return this
  };
}
```

到这里，代码从理解上很简单，就是直接在 Vue 上定义一个 mixin 的全局方法。至于如何进行合并选项，下面会提及。而这就是 mixin 方法的全局入口，其实它还有一个局部入口，即直接在 Vue 实例上进行初始化定义 mixin。我们接着看看局部入口是怎样的：

```javascript
Vue.prototype._init = function (options) {
  // ...
  vm.$options = mergeOptions( // 开始合并选项
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  );
  // ...
}
```

可以看到，在初始化阶段，同样使用了方法`mergeOptions`来进行合并选项，那么这个方法究竟是怎样合并的？我们接着看下去：

```javascript
/**
  * Merge two option objects into a new one.
  * Core utility used in both instantiation and inheritance.
  */
function mergeOptions (
 parent, // Vue 实例本身
 child, // 要合并的选项
 vm
) {
  // ...
  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) { // 针对子组件中扩展，就开始递归其扩展直到找到最后一层的mixins出来进行合并选项
      parent = mergeOptions(parent, child.extends, vm);
    }
    if (child.mixins) { // 当子组件中存在mixins选项时，就开始递归寻找最后一层的mixins出来进行合并选项
      for (var i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm);
      }
    }
  }

  var options = {};
  var key;
  for (key in parent) { // 将Vue实例上的属性与要合并的选项值中的属性进行合并
    mergeField(key);
  }
  for (key in child) {
    if (!hasOwn(parent, key)) { // 遍历要合并的选项，若遍历到的属性是Vue实例不存在的，也要进行合并处理
      mergeField(key);
    }
  }
  function mergeField (key) { // 真正的合并选项
    var strat = strats[key] || defaultStrat;
    options[key] = strat(parent[key], child[key], vm, key);
  }
  return options
}
```

方法`mergeOptions`从原理的角度分析并不难，就是**递归寻找要合并选项中优先级最低的 mixin 取出来**，在遍历Vue实例上的属性进行合并处理时，其实也包含了与 Mixin 中的属性合并。然后再遍历一层一层递归回去取出来的 mixin ，若是 Vue 实例不存在的属性也进行合并处理。

那么问题来了，真正的合并选项方法中的`strats`和`defaultStrat`究竟是用来干嘛的？我们先来看看它们的结构

```javascript
var strats = config.optionMergeStrategies;

var config = ({
  // ...
  optionMergeStrategies: Object.create(null),
  // ...
})

var defaultStrat = function (parentVal, childVal) {
  return childVal === undefined
    ? parentVal
  	: childVal
}
```

`strats`本质上就是一个空对象，那么为什么在合并选项时，还要传递对应选项 Key 值进去？好明显，在你传递 Key 值进去之前，别人就已经实现好对应选项合并的方法在里面了。

再来看看`defaultStrat`，理解上应该没什么问题，就是判断 Mixin 是否存在，一旦为 undefined 时就直接取 Vue 实例，否则直接取 Mixin 中对应的选项值。（其实这里我刚开始感觉特别绕，后面想了想也能思考出来，就是凡是上面提及的定义项外的属性，都是直接取 Mixin 中即可，因为它们已经超出了定义的范围）

看到这里，是否会感觉有点难以理解？慢慢来，总能理解到作者这样写的用意的 😄。另外你应该也知道，Mixin 中最大的 Boss 也就是`strats`了。由于篇幅缘故，对于它的源码，我们会在下一篇讲解，欢迎收看。

现在再来总结一下：

- Mixin 针对不同的情景做了不同的合并处理，详细可以看上面优先级；
- 合并选项使用方式是递归遍历，一直寻找到最后一项的 Mixin，再去进行合并。只有在 Vue 规定内的属性才会进行合并选项，否则直接拿 Mixin 中的选项；







































