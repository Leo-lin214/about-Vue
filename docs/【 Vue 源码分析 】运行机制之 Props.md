## 先聊聊 with

相信很多同学面对 with 语句时，都会相对陌生（因为平时实际开发中很少会用到）。那它究竟是用来干嘛的？官方文档给出的解释如下：

> **with语句** 扩展一个语句的作用域链

这句话如何理解？简单滴说，改变作用域链中优先读取的变量值。说白点，相当于在一个作用域中访问一个变量时，会先判断改变量是否存在于`with`指定的对象中，若存在则直接读取，否则会沿着作用域链一直向上找。

我们先来看一个栗子🌰简单了解一下：

```javascript
function add() {
	var a = 1
  with(this) {
    return a + 1
  }
}

console.log(add())
console.log(add.call({}))
console.log(add.call({ a: 2 }))
```

上面的代码很好理解，那究竟三个输出会得到什么呢？现在就来讲解一下

1. 第一个输出为 2。道理很简单，当执行单纯`add`方法时，执行上下文 this 指向的是 window 对象。因此在 with 语句中，访问 a 变量时，会先从 this （即 window 对象）中进行寻找，一旦不存在，就会沿着作用域链向上找，即会到函数作用域 add 中寻找，从而得到了结果 2；
2. 第二个输出为 2。同样地，因为一个空对象中 {} 并木有 a 属性的值，因此也是直接去到函数作用域中进行寻找，从而得到 2；
3. 第三个输出为 3。好明显，在对象 { a: 2 } 中有一个属性 a 为 2，因此直接得到结果为 3；

通过上面的解析，可以看到，其实原理上跟我们上一节中的使用`call和apply`是一个道理，都是改变执行上下文，但是`with`语句却有着与众不同的地方，那就是找不到值了，不会报错而是会继续往作用域链向上进行寻找，直到找到为止。

对于应用场景来说，针对访问多层属性的值特别有用，举个例子：

```javascript
var obj = {
  item1: {
    item2: {
      a: 1,
      b: 2,
      c: 3
    }
  }
}

console.log(obj.item1.item2.a)	// 1
console.log(obj.item1.item2.b)	// 2
console.log(obj.item1.item2.b)	// 3

// 相当于
with(obj.item1.item2) {
  console.log(a)	// 1
  console.log(b)	// 2
  console.log(c)	// 3
}
```

可以看到，使用`with`语句可以很好滴简化了我们平时频繁访问多层对象中属性的问题。那么，`with语句`就这么好使，没弊端吗？

事实上并不是的，在官方文档中并不推荐我们使用`with语句`。原因有几个：

- 由于`with`语句都会使得作用域先从指定的对象中进行寻找，对于那些本来就不是这个对象的属性，查找起来将会特别慢；

- `with`语句并不利于阅读；

- `with`语句无法向前兼容，举个例子：

  ```javascript
  function f(foo, values) {
    with (foo) {
      console.log(values)
    }
  }
  ```

  我们都知道，foo 为数组时，在 ES5 中都会返回正确的结果，而由于在 ES6 中，数组中已经拓展了一个`Array.prototype.values`方法，因此在 ES6 环境中就无法得到正确的结果了；

以上的弊端均来自[官方文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/with)，有兴趣的同学都可以去了解一下。

而在 Vue 中同样使用了`with`语句进行传递作用域，既然它有那么多的弊端，为何还要用它呢？其实[尤大大也在github上解析了一番](https://github.com/vuejs/vue/issues/4115)，简单来说，就是为了方便管理，才会统一使用`with`语句。



## 父组件是如何通过 Props 传递到子组件？

由于篇幅原因，其实该小节的内容涉及到一个很重要的知识点，那就是`compile编译分析`（会在后面的章节进行讲解）。所以小弟我会先绕过它，去寻找与该节相关的东西。望同学们体谅一下，也是为了方便大家好理解。

在开始之前，建议同学们还是看看[官方文档中渲染函数一章节](https://cn.vuejs.org/v2/guide/components-registration.html)，也是为了方便理解

在组件进行挂载编译时，都会被解析成一个渲染函数，而这个渲染函数中就会使用到上一节所提到`with`语句来进行作用域的传递。我们先从一个栗子🌰开始入手：

```javascript
// html
<div id="app">
	<person :name="name" />
</div>

// js
new Vue({
  el: '#app',
  components: {
    Person: {
      name: 'person',
      props: {
        'name'
      },
      template: '<span>{{ this.name }}</span>'
    }
  },
  data: {
    name: 'andraw'
  }
})
```

上面的代码在经过 Vue 源码挂载时，会得到如何渲染函数：

```javascript
(function anonymous(
) {
with(this){
  return _c(
    'div',
    {attrs:{"id":"app"}},
    [
      _c(
        'person',
        {attrs:{"name":name}}
      )
    ], 1)}
})
```

看到这个渲染函数，是否会被懵逼？现在就来讲解一下

- with：可观看上一节的内容，目的就是为了把当前 vue 实例作用域绑定到 this 中，从而使父组件中传递给子组件的属性，都会先从当前 vue 实例（即父组件）中进行寻找。好明显，此时`person`组件中 attrs 的 name 属性会先从当前 vue 实例（即父组件）中寻找，值也为`andraw`；

- _c：在初始化 Render 的过程中，就已经将其作为`createElement`的通道，源码如下：

  ```javascript
  initRender(vm);
  
  function initRender (vm) {
    // ...
    vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
    // ...
  }
  ```

  因此，可以说**`_c`其实就是指向`createELement`方法**。而对于`_c`方法中的参数，其实可以看下官方文档中的渲染函数章节，里面会有比较详细的讲解。



现在来总结一下：父组件使用`with`语句以及渲染函数`createELement`来实现了将 Props 传递到子组件中。

当然，你也会有疑惑，究竟`createElement`是何方神圣？如果你用过`React`的话，其实会很好理解，当然我也会在后面的`compile编译`章节中提及。我们先跳过，然后接着看下去先哈 😂.....



## 从源码角度进行分析

这一次，通过源码阅读，主要探索的方面包括如何初始化 Props、以及如何进行更新。

```javascript
initState(vm);

function initState(vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.props) { initProps(vm, opts.props); }
  // ...
}

function initProps (vm, propsOptions) {
	var propsData = vm.$options.propsData || {}; // 获取Vue实例选项上的Props
  var props = vm._props = {}; // 获取挂载Vue实例上的_props
  var keys = vm.$options._propKeys = []; // Props的Key值组成的数组
  // ...
	for (var key in propsOptions) loop( key ); // 循环遍历 vue 实例选项中Props，并且执行响应式处理以及挂载在对应实例上
  // ...
}
```

初始化 Props 的关键点就在于`loop`函数，让我们接着该函数做了什么事情。

```javascript
var sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
};
function proxy (target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  };
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}

var loop = function ( key ) {
  keys.push(key); // 每遍历一次Props中的值，都会收集其Key
  // ...
  defineReactive$$1(props, key, value, function () { // 将Vue实例上对象的_props中每一项属性都设置响应式
    if (!isRoot && !isUpdatingChildComponent) { // 配置的Setter函数，避免子组件直接操作Props的警告
      warn(
        "Avoid mutating a prop directly since the value will be " +
        "overwritten whenever the parent component re-renders. " +
        "Instead, use a data or computed property based on the prop's " +
        "value. Prop being mutated: \"" + key + "\"",
        vm
      );
    }
  });
  // static props are already proxied on the component's prototype
  // during Vue.extend(). We only need to proxy props defined at
  // instantiation here.
  if (!(key in vm)) { // 遍历时若发现新属性时，就将新属性重新挂载到Vue实例的_props中
    proxy(vm, "_props", key);
  }
};
```

理解上应该不会有太大的问题，`loop`函数中主要做了以下几件事：

- defineReactive$$1

  相信看过前面章节的童鞋们对这个函数应该都不会陌生，其实就是对Vue实例上的 _props 对象中每一个属性都配置成响应式的，这样一来，当父组件中传递进来的 Props 变化时，则会通知相应的子组件中更新函数进行更新；

- 对相应的子组件配置Props属性的Setter警告函数

  用过Vue的童鞋们应该都会遇到直接更改一个 Props 中的属性时，会抛出一个警告。而这个警告就是在 Vue 遍历 _props 对象中的值时，都会默认配置一个警告 Setter 函数；

- proxy

  在遍历的过程中，一旦发现有新的属性时，都会将新属性重新挂载到 Vue 实例的 _props 中。这里有一个很重要的知识点，**当我们直接访问一个 Props 中的属性时，即上面栗子中`this.name`，其实是直接访问了 Vue 实例的 _props 对象中值而已**。

至此，我们也知道 Vue 源码是如何实现初始化 Props 的了，那么，究竟是父组件是如何通知更新 Props 的呢？我们接着看下去。

由于父组件在更新的过程中，会通知子组件也进行更新，这时候就会调取一个方法`updateChildComponent`，而这个方法里就会对 Props 进行更新。我们就来看看是如何处理的：

```javascript
updateChildComponent(
  child,
  options.propsData, // updated props
  options.listeners, // updated listeners
  vnode, // new parent vnode
  options.children // new children
);

function updateChildComponent (
 vm,
 propsData,
 listeners,
 parentVnode,
 renderChildren
) {
   // ...
   // update props
   if (propsData && vm.$options.props) {
     toggleObserving(false); // 关闭依赖监听
     var props = vm._props; // 获取Vue实例上_props对象
     var propKeys = vm.$options._propKeys || []; // 获取保留在Vue实例上的props key值
     for (var i = 0; i < propKeys.length; i++) { // 循环遍历props key
       var key = propKeys[i];
       var propOptions = vm.$options.props; // wtf flow?
       props[key] = validateProp(key, propOptions, propsData, vm); // 先校验Props中定义的数据类型是否符合，符合的话就直接返回，并且直接赋值给Vue实例上_props对象中相应的属性中
     }
     toggleObserving(true); // 打开依赖监听
     // keep a copy of raw propsData
     vm.$options.propsData = propsData; // 新的PropsData直接取替掉选项中旧的PropsData
   }
   // ...
 }
```

一开始看到这里代码时，我是懵逼状态的，因为很容易绕不出来 😂。。在这里面会有几个问题，分别是：

- validateProp 作用究竟是什么？

  相信用过 Props 的同学都清楚，在传递给子组件时，子组件中是有权限决定传递的值类型的，大大提高传递的规范，举个例子：

  ```javascript
  props: {
    name: {
      type: String,
      required: true
    }
  }
  ```

  代码很好理解，就是规定 name 属性的类型以及是否必传。而**方法`validateProp`作用就是校验父组件传递给子组件的值是否按要求传递，若正确传递则直接赋值给 _props 对象上相应的属性**。

- 校验通过后，直接赋值给 _props 对象上相应的属性的用意何在？

  上面提到过，_props 对象上的每一个属性都会使用 proxy 方法进行响应式挂载。那么当我直接赋值到 _props 对象上相应的属性时，就会触发到其 Setter 函数进行相应的依赖更新。因此，当父组件更新一个传递到子组件的属性时，首先会触发其 Setter 函数通知父组件进行更新，然后通过渲染函数传递到子组件后，更新子组件中的 Props。这时候，由于此时的 Props 对象中的属性收集到了子组件的依赖，更改后会通知相应的依赖进行更新。

- toggleObserving 究竟是干嘛用的？

  要理解该方法的用处，必须先从前面响应式探究章节有了解过方法`observe`，为什么要提到它？

  首先它是一个递归遍历方法，**Props 在通知子组件依赖更新时，必须搞清楚的一点，就是是整个值的变化来进行通知**。如何理解？简单滴说，对于属性值为基本数据类型的，当值改变时，是可以直接通知子组件进行更新的，而对于复杂数据类型来说，前面章节我们说到过，在更新时，会递归遍历其对象内部的属性来通知相应的依赖进行更新。

  那么**当调用方法`toggleObserving`为 false 时，对于基础数据类型来说，当其值变化时则直接通知子组件更新，而对于其复杂数据类型来说，则不会递归下去，而只会监听整个复杂数据类型替换时，才会去通知子组件进行更新。因此在 Props 中所有属性通知完后，又会重新调用方法`toggleObserving`为 true 来打开递归开关**。（真的不得不服尤大大啊，这么好的优化思路都能想出来，牛人 👍）



至此，你大概也知道整个更新流程了，但是我当时还是存在疑惑的，既然基础数据类型值更改或复杂数据类型整个值更改，可以直接通知到子组件进行更新，那么是否会有一种情况就是，复杂数据类型中属性更改时，又是如何通知子组件更新的呢？？🤔

对于这个问题，当时的我也是想到想发烂咋。。。最后还是看代码后才能一步一步滴理解其深奥之处。

首先，我们一开始已经忽略一个方法，那就是`defineReactive$$1`，这个方法真的用的秒，可以看看上面的代码，在初始化 Props 时候，会对 Props 每一项的属性进行使用该方法进行响应式的处理，包括了复杂数据类型的中属性，此时该属性不但收集了父组件依赖，还收集了子组件的依赖，这样一来，当复杂数据类型中属性变化时，会先通知父组件更新，再通知子组件进行更新。（这时候我真的不得不服到五体投地。。。）

总结一下：

- 当 Props 中属性为基础数据类型值更改或复杂数据类型替换时，会通过 Setter 函数通知父组件进行更新，然后通过渲染函数，传递到子组件中更新其 Props 中对象相应的值，这时候就会触发到相应值的 Setter 来通知子组件进行更新；
- 当 Props 中属性为复杂数据类型的属性更改时，由于使用`defineReactive$$1`方法收集到了父组件依赖以及子组件的依赖，这时候会先通知父组件进行更新，再通知子组件进行更新；

































































