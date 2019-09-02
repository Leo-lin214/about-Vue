## 回顾侦听器 Watch

相信用过 Vue 侦听器的 Watch 的同学们，都应该比较熟悉，现在就来简单回顾一下，这对后面理解源码起来会简单好多。当然如果你觉得没必要，可以直接跳过去到下面的源码分析哈 🤔 。

侦听器 Watch 接收类型，可以是`String、Function、Object、Array`，那么这些类型你都没有用过呢，别急，下面就会挪用官方文档进行简单讲解一下。

- String

  对于使用字符串类型的情况不多见，但可以体现封装性

  ```javascript
  watch: {
    a: 'setA'
  },
  methods: {
    setA: function() {
      // ...
    }
  }
  ```

  使用字符串时，**字符串的值就是函数名**，并且必须要在`mehods`中有定义。

- Function

  函数类型可以说是我们最常用的监听方式了，但需要注意其参数的表示

  ```javascript
  watch: {
    a: function(newValue, oldValue) {
      // ...
    }
  }
  ```

  **第一个参数就是更新后的值，而第二个则是更新前的值**。

- Object

  用于监听深层次对象中的值变化，至于其他三种方式则无法监听对象中属性变化

  ```javascript
  data: {
    a: {
      b: 1
    }
  },
  watch: {
    a: {
      deep: true,
      handle: function(newValue, oldValue) {
        // ...
      }
    }
  }
  ```

  由于监听的是一个对象，因此当使用其他三种方式时，当改变`this.a.b`后是无法监听其变化的，为什么？后面会单独提出来讲解。使用`deep`属性即可监听深层次的变化了。

- Array

  用于监听多个回调函数时，按序进行

  ```javascript
  watch: {
    a: [
      'set1',
      function set2(newValue, oldValue) {
        // ...
      }
    ]
  }
  ```

  最终会按序执行其中定义的回调函数。



好了，回到刚刚讨论的问题，为什么当我更改`this.a.b`时，使用除了`Object`类型以外的方式无法监听到`a`的变化呢？

原因很简单，那就是在初始化 Watch 并建立依赖收集时，只会对第一层的属性进行收集，即定义 a 的 Watch 会被收集到响应式属性 a 的依赖中。只有当发现`Object`类型中拥有`deep: true`时，才会开始对 a 进行递归遍历，然后将 a 中所有深层响应式属性中都能收集到定义 a 的 Watch 。

当然，除了`Object`类型中`deep`属性，还有`immediate`和`handler`，现在就来讲解一下

- immediate：用于通知定义的 Watch 回调函数立即调用；
- handler：用于定义 Watch 的回调函数；

说了那么多，我始终都是不推荐使用 Watch 的。因为 Watch 真的不好管理，对于最终 Bug 的过程中难以找到准确的触发点。当然官方也并不是反对你使用，但是依然推崇还是计算属性来处理大部分情况，并且还拥有一个强大的缓存功能。

接下来就从源码角度进行分析，如果看过前面篇章的同学们，其实很好理解，当然没看过的同学也不要紧，我会一个一个很好滴解析哈 😄 。



##从源码角度进行分析 

现在就来看看 Vue 在初始化 Vue 实例时是如何处理侦听器的。

```javascript
// Firefox has a "watch" function on Object.prototype...
var nativeWatch = ({}).watch;

function initState (vm) {
  // ...
  if (opts.watch && opts.watch !== nativeWatch) { // 判断是否存在 watch 选项以及避免与火狐浏览器中对象原型中 watch 产生冲突
    initWatch(vm, opts.watch); // 初始化 watch 选项
  }
}

function initWatch (vm, watch) {
  for (var key in watch) { // 遍历 watch 选项中的值
    var handler = watch[key]; // 获取每一项定义的 watch
    if (Array.isArray(handler)) { // 若遍历到的属性为数组类型，则为数组中每一项值进行创建 Watch
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}

function createWatcher (
    vm,
    expOrFn,
    handler,
    options
  ) {
    if (isPlainObject(handler)) {
      options = handler; // 一旦遍历到的值是一个对象时，就直接拿该项定义的 watch 作为选项配置
      handler = handler.handler; // 获取 watch 中定义的回调函数
    }
    if (typeof handler === 'string') { // 当遍历到的值是一个字符串时，则直接根据该字符串从实例上获取该项方法出来
      handler = vm[handler];
    }
    return vm.$watch(expOrFn, handler, options) // 创建 watch
  }

Vue.prototype.$watch = function (
  expOrFn,
  cb,
  options
) {
  var vm = this;
  if (isPlainObject(cb)) { // 当获取到 handler 依然还是一个对象时，则继续递归调用 createWatcher 方法
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {};
  options.user = true;
  var watcher = new Watcher(vm, expOrFn, cb, options); // 建立与响应式属性间的依赖收集
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value);
    } catch (error) {
      handleError(error, vm, ("callback for immediate watcher \"" + (watcher.expression) + "\""));
    }
  }
  return function unwatchFn () { // 需要注意的地方，当使用全局 $watch 时，返回的是一个可取消监听方法
    watcher.teardown(); // 取消监听
  }
};
```

从上面代码可以看到，涉及的知识点并不是很多，也很好理解。现在来总结一下

- 对定义的 watch 类型进行判断，若是数组类型时，则会遍历出来为每一项值创建相应的 watch；
- 若 watch 定义时是一个对象类型，则直接作为依赖的 options 选项；
- 当存在 immediate 属性时，会立即调用 watch 的回调函数；
- 调用全局 $watch 时，都会返回一个可取消监听的方法（即监听的属性改变时，不会调用 watch 的回调函数）；



接下来我们再看看，类 Watcher 是如何为 watch 选项建立与响应式属性的依赖关系的。

```javascript
var bailRE = new RegExp(("[^" + (unicodeRegExp.source) + ".$_\\d]"));
function parsePath (path) {
  if (bailRE.test(path)) {
    return
  }
  var segments = path.split('.');
  return function (obj) {
    for (var i = 0; i < segments.length; i++) {
      if (!obj) { return }
      obj = obj[segments[i]];
    }
    return obj
  }
}

var Watcher = function Watcher (
    vm,
    expOrFn,
    cb,
    options,
    isRenderWatcher
  ) {
      // ...
      // options
      if (options) {
        this.deep = !!options.deep; // 针对有定义 deep 属性，会赋予 watcher 中内在的 deep
        // ...
      } else {
        this.deep = this.user = this.lazy = this.sync = false;
      }
      this.cb = cb; // 获取 watch 的回调函数
      // ...
      if (typeof expOrFn === 'function') {
        this.getter = expOrFn;
      } else { // 定义的 expOrFn 并不是一个函数类型，因此最后会使用 parsePath 去解析
        this.getter = parsePath(expOrFn);
        if (!this.getter) {
          this.getter = noop;
          warn(
            "Failed watching path: \"" + expOrFn + "\" " +
            'Watcher only accepts simple dot-delimited paths. ' +
            'For full control, use a function instead.',
            vm
          );
        }
      }
      this.value = this.lazy // 开始执行计算 watch 
        ? undefined
        : this.get();
    }

Watcher.prototype.get = function get () {
  pushTarget(this);
  var value;
  var vm = this.vm;
  try { 
    value = this.getter.call(vm, vm); // 关键点，由于引用到响应式属性，因此最终会在响应式 Getter 中收集该定义的 watch 依赖
  } catch (e) {
    if (this.user) {
      handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) { // deep 属性的关键，就是递归遍历，为每一个响应式属性收集当前创建的 Watcher
      traverse(value);
    }
    popTarget();
    this.cleanupDeps();
  }
  return value
};
```

代码中可以很好理解，前面我已经说了，在创建 Watcher 依赖时，需要传更新回调函数`expOrFn`进来以便进行计算。但是 watch 会稍微一点点不一样，`expOrFn`传递的却是一个属性名，而`cb`才是更新回调函数。

既然`expOrFn`不是一个函数类型，因此就会调用`parsePath`方法进行解析，返回一个解析回调。在最后`this.get()`方法中，会调用到`this.getter`中的解析回调，并传入当前 vue 实例。最终从 vue 实例上解析出来一个同名响应式属性，以至于触发了响应式属性的 Getter 函数，这样一来，响应式属性就收集了当前创建 Watcher。

另一方面，也可以看到，在`this.get`方法中，当发现`this.deep`存在时，会调用`traverse`方法来为每一个深层响应式属性收集当前创建的 Watcher 依赖，让我们再来看看是如何处理的。

```javascript
var seenObjects = new _Set(); // 创建一个集合

function traverse (val) {
  _traverse(val, seenObjects); // 开始遍历操作
  seenObjects.clear();
}

function _traverse (val, seen) {
  var i, keys;
  var isA = Array.isArray(val);
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (val.__ob__) {
    var depId = val.__ob__.dep.id;
    if (seen.has(depId)) {
      return
    }
    seen.add(depId); // 当 __ob__ 中不存在当前 Watcher 时，就开始收集
  }
  if (isA) {
    i = val.length;
    while (i--) { _traverse(val[i], seen); } // 如果遍历到的是数组，那么就会递归遍历其中每一项值，并建立依赖关系
  } else {
    keys = Object.keys(val);
    i = keys.length;
    while (i--) { _traverse(val[keys[i]], seen); } // 当遍历到的是对象时，同样也会递归遍历其中每一项值，并建立依赖关系
  }
}
```

我们可以看到，当存在 deep 属性时，就证明该监听的对象为一个复杂数据类型。前面我也提到过，对于监听复杂数据类型时，使用原有的`Object.defineProprtty`方法是无法实现完整的监听，而是使用一个变量`__ob__`进行存储。

因此，上面的方法`_traverse`就是遍历复杂数据类型，然后为每一个复杂数据类型的`__ob__`收集当前创建的依赖 Watcher。

那么，如果是更新呢？过程又是如何的？

其实 watch 的更新过程和前面所说的更新过程都是一样，都是调用`queueWatcher`方法，按照队列的形式进行。虽然 Watch 涉及的东西不多，但是要重点注意其中`deep	`和`immediate`两个属性，由于这两个属性的存在，可以很好处理一些日常开发难题，但最后，我还是建议大家多使用计算属性来处理问题，毕竟滥用 watch 的话，到最后是一个难以维护的事实。



























































