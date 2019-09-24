本篇主要接着上一篇中提及的关键点，对`strats`中原先定义好的合并规则的探究。我们知道，Mixin 会对不同的场景做不同的合并处理，因此我们接下来也会分情况来进行分析。



## 源码角度分析 Data 合并

先回顾一下对于 Data 合并的优先级处理，如下：

```javascript
组件data > 组件mixin > 组件mixin的mixin > ... > 全局mixin
```

我们现在就来看看源码是如何实现`strats`对象对 Data 合并的定义。

```javascript
strats.data = function (
 parentVal,
 childVal,
 vm
) {
	// ...
  return mergeDataOrFn(parentVal, childVal, vm) // 开始合并Vue实例上data与Mixin上的data
}

/**
  * Data
  */
function mergeDataOrFn (
 parentVal,
 childVal,
 vm
) {
  // ...
  return function mergedInstanceDataFn () {
    // instance merge
    var instanceData = typeof childVal === 'function' // 判断Mixin中定义的data是否为函数
    	? childVal.call(vm, vm) // 若为函数，则直接调用并返回
    	: childVal; // 否则直接返回data对象或为空
    var defaultData = typeof parentVal === 'function' // 判断Vue实例上中定义的data是否为函数
    	? parentVal.call(vm, vm) // 若为函数，则直接调用并返回
     	: parentVal; // 否则直接返回data对象或为空
    if (instanceData) {
      return mergeData(instanceData, defaultData) // 正式合并处理
    } else {
      return defaultData
    }
  }
}

/**
  * Helper that recursively merges two data objects together.
  */
function mergeData (to, from) {
  if (!from) { return to } // 一旦不存在Mixin时，直接返回Vue实例即可
  var key, toVal, fromVal;

  var keys = hasSymbol // 先获取Mixin中data的每一项属性Key值所组成的数组
  ? Reflect.ownKeys(from)
  : Object.keys(from);

  for (var i = 0; i < keys.length; i++) { 
    key = keys[i];
    // in case the object is already observed...
    if (key === '__ob__') { continue } // 跳过key值为__ob__的选项（该选项存储的是依赖）
    toVal = to[key];
    fromVal = from[key];
    if (!hasOwn(to, key)) { // 当遍历data属性的key值在Vue实例data上是不存在时，则直接赋值到Vue实例data上
      set(to, key, fromVal);
    } else if (
      toVal !== fromVal &&
      isPlainObject(toVal) &&
      isPlainObject(fromVal) // 当遍历data属性值是一个对象时，则继续递归遍历下去与Vue实例data对象中同名属性值进行合并处理
    ) {
      mergeData(toVal, fromVal); // 递归遍历复杂数据类型进行合并
    }
  }
  return to // 最后处理合并好的data对象并返回
}

/**
  * Set a property on an object. Adds the new property and
  * triggers change notification if the property doesn't
  * already exist.
  */
function set (target, key, val) {
  // ...
  var ob = (target).__ob__;
  // ...
  defineReactive$$1(ob.value, key, val); // 在对应Vue实例data中设置相应的值，并赋予响应式处理
  ob.dep.notify(); // 并更新对应回调
  return val
}
```

代码看上去复杂了许多，但当你细心斟酌，总能体会到其中意思。

总结一下：**Data 对于重名的合并处理，不属于该 Vue 实例 Data 上定义的才会去赋值或响应式处理，属于该 Vue 实例 Data 上定义的，若是基本数据类型则会直接取该值，若是复杂数据类型则会继续递归遍历其里面的属性值，而对于不重名的合并处理，则是优先级高会覆盖优先级低的**。



## 源码角度分析 Props \ Methods \ Inject \ Computed 合并

Props \ Methods \ Inject \ Computed 这四者的合并会很好理解，遇到同名的属性，优先级高的就会直接取替到优先级低的。我们先来回顾一下该类的优先级：

```javascript
组件 > 组件mixin > 组件mixin的mixin > ... > 全局mixin
```

其实可以看到的是，处理上感觉跟 Data 上一致，但事实并不是如此。Props \ Methods \ Inject \ Computed 对于同名属性会直接覆盖处理，相当于 Data 上对于不重名属性的合并处理。我们看看源码咋实现的

```javascript
/**
  * Other object hashes.
  */
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
 parentVal,
 childVal,
 vm,
 key
) {
	// ...
  if (!parentVal) { return childVal } // 当Vue实例上没有该选项（即Props\Methods\Inject\Computed）时，直接取Mixin中的
  var ret = Object.create(null); // 创建一个空对象
  extend(ret, parentVal); // 将对应选项中属性值赋值到新创建的对象中
  if (childVal) { extend(ret, childVal); } // 当存在子实例时，会进行覆盖处理
  return ret
}

/**
  * Mix properties into target object.
  */
function extend (to, _from) { // 覆盖处理
  for (var key in _from) {
    to[key] = _from[key];
  }
  return to
}
```

看完上述的代码，我感觉完全懵了，为什么？你会发现使用`extend`方法进行覆盖处理时候，它会拿最后一层的 Mixin 中 Props \ Methods \ Inject \ Computed 的直接覆盖了 Vue 实例上已有的，然后逐层递归回来覆盖，那么问题来了，虽然说优先级高的会覆盖优先级低的，那么这样逐层递归回去时，到了第一层时是不是就是组件的 Mixin？

一开始我也是错误滴认为递归回到第一层时刚好就是组件的 Mixin 😂，但后面细心看了看代码，就发现了其实第一层应该就是 Child，而这个 Child 刚好就是`this.$options`，为此`this.$options`指向的就是 Vue 实例上的选项了。因此最后还是能够遵循优先级高的覆盖优先级低的。

总结一下： **Props \ Methods \ Inject \ Computed 对于重名的属性会取优先级高的，即直接按优先级高的覆盖优先级低的。对于只有优先级低的拥有，而优先级高的没有的属性，则会直接赋值处理**。



## 源码角度分析 Watch 合并

监听的合并处理的不是 Watch 选项中同名的属性，而是整个 Watch 选项，将组件的 Watch 选项和 Mixin 中的 Watch 选项组合成一个数组，优先级高的放在第一位，以此类推。在上一篇提及过，Watch 调用的优先级如下：

```javascript
全局mixin > ... > 组件mixin的mixin > 组件mixin > 组件
```

现在我们就来从源码的角度进行分析一下。

```javascript
/**
  * Watchers.
  *
  * Watchers hashes should not overwrite one
  * another, so we merge them as arrays.
  */
strats.watch = function (
 parentVal,
 childVal,
 vm,
 key
) {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) { parentVal = undefined; } // 判断是否为原生对象的Watch
  if (childVal === nativeWatch) { childVal = undefined; } // 判断是否为原生对象的Watch
  /* istanbul ignore if */
  if (!childVal) { return Object.create(parentVal || null) } // 当Mixin不存在时，重新创一个新的对象
  // ...
  if (!parentVal) { return childVal } // 当Mixin存在而Vue实例中不存在时，则直接取Mixin中的
  var ret = {};
  extend(ret, parentVal); // 先将Vue实例中的选项添加到新对象中
  for (var key$1 in childVal) { // 从最后一层Mixin递归遍历开始，直到this.$options
    var parent = ret[key$1];
    var child = childVal[key$1];
    if (parent && !Array.isArray(parent)) { // 一旦Vue实例中存在Watch选项不是数组时，则赋予一个数组形式
      parent = [parent];
    }
    ret[key$1] = parent
      ? parent.concat(child) // 合并数组处理
    	: Array.isArray(child) ? child : [child]; // 最后一层Mixin时，由于parent不会存在Watch选项，因此会赋予一个数组形式
  }
  return ret
}
```

经过上一次清晰滴知道 Child 最终会递归到`this.$options`时，就会相对好理解上面代码。原理上就是从最后一层的 Mixin 逐层递归回来时，进行数组合并，明显滴全局的 Mixin 会被放到数组的第一位，需要知道的是，**一开始 Vue 实例上是不会有 Watch 选项的**。

总结一下：**Watch 选项会使用数组合并来处理，从最后一层的 Mixin 递归回来，逐层数组合并，因此全局 Mixin 会被放到数组的第一位，以此类推，到了`this.$options`后，会将其中的 Watch 选项添加到数组的末尾。而执行时，全局 Mixin 会先执行，以此类推，直到组件中的 Watch 选项最后才执行**。



## 源码角度分析 Lifecycle 合并

生命周期的合并其实在处理上跟监听的合并基本一致，也是通过数组来进行合并的。我们先来看看生命周期合并后调用的优先级，如下：

```
全局mixin > ... > 组件mixin的mixin > 组件mixin > 组件
```

既然也是全局 Mixin 的先执行，那么合并的方式也是一样的，我们就来看看源码里是如何实现的：

```javascript
var LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]

LIFECYCLE_HOOKS.forEach(function (hook) {
  strats[hook] = mergeHook; // 每一种生命周期函数都会拥有一个单独的合并处理函数mergeHook
})

/**
  * Hooks and props are merged as arrays.
  */
function mergeHook (
 parentVal,
 childVal
) {
  var res = childVal
  	? parentVal
  		? parentVal.concat(childVal) // 合并数组处理
  		: Array.isArray(childVal)
  			? childVal
  			: [childVal]
  	: parentVal;
  return res
    ? dedupeHooks(res) // 删除重复的生命周期函数
  	: res
}
```

可以看到，代码在合并的处理上基本跟 Watch 的处理是一致的，都是从最后一层 Mixin 递归遍历回来，直到组件本身的生命周期函数为止。因此在调用顺序方面，都是先执行全局 Mixin 中定义的生命周期函数，以此类推回到组件本身生命周期函数的。

总结一下：**Lifecycle 在合并处理上跟 Watch 是一致的，都是使用数组合并方式，从最后一层的 Mixin （即全局 Mixin ）开始递归回来，直到组件本身，因此调用顺序也会先执行全局 Mixin，最后才执行组件本身的生命周期函数**。



## 源码角度分析 Component \ Directive \ Filter 合并

在上一篇中，我有提及过，这三项的合并处理与其他项的合并处理会很不一样，使用的是**原型链方式**来合并处理。

相信同学们都清楚原型链的访问方式，在访问一个对象中的变量时，会沿着原型链进行查找，直到`Object.prototype`为止。而尤大大恰恰就巧妙滴利用了这一点，使用最简单方式进行了合并处理，不得不服 🤔。

我们先来回顾一下该三项访问的优先级关系：

```javascript
组件 > 组件mixin > 组件mixin的mixin > ... > 全局mixin
```

可以看到的是，先访问组件本身的这三项内容，一旦找不到了才会去找组件 Mixin 的，以此类推，直到全局 Mixin 为止。我们就先来看看源码是如何实现原型链合并的。

```javascript
var ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets;
})

/**
  * Assets
  *
  * When a vm is present (instance creation), we need to do
  * a three-way merge between constructor options, instance
  * options and parent options.
  */
function mergeAssets (
 parentVal,
 childVal,
 vm,
 key
) {
  var res = Object.create(parentVal || null); // 继承上一次递归处理好的对象
  if (childVal) {
    // ...
    return extend(res, childVal) // 将递归到的对象直接进行赋值处理
  } else {
    return res // 否则直接返回该对象
  }
}
```

代码从理解上应该不会有太大的问题，在最后一层的 Mixin （即全局 Mixin）时，会先继承了一个 Vue 实例上的项，其实这时 Vue 实例上的项并没有任何东西。逐层递归回来，每一次都会继承上一次处理好的结果，这样一来，到组件本身时，就会直接将组件本身赋予到原型链的最底层，因此在访问时就会先访问组件本身再去沿着原型链进行向上查找。

总结一下： **Component \ Directive \ Filter 使用原型链的方式来进行合并处理，从最后一层 Mixin （即全局 Mixin）递归回来，每一次合并时都会先继承上一次处理好的对象，以此类推，到组件本身时，就会将组件直接放到了原型链的最底层。因此在访问时，就会先访问组件本身的，再沿着原型链向上询问**。























































