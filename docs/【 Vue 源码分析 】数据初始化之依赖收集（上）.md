## 先聊聊闭包

说起闭包，可以说是大前端里老生常谈的话题了。那为啥要在这里说起它呢，自然肯定会有地方是有用到它才会拿出来进行讨论和回顾，废话不说，直奔主题。

那究竟什么是闭包？

按照官方的定义，**闭包是指有权访问另一个函数作用域中变量的函数**（其实我一直觉得理解官方的字面意思好别扭，而且最主要的，就是不好理解啊！！！🤮，纯属个人意见）。来个直白点的理解，就是在一个函数内部创建一个函数，而这也是创建闭包最常见的方式，看🌰：

```javascript
function sum() {
  var a = 0
  return function(b) {
    return a += b
  }
}
var sumFun = sum()
sumFun(1) // 1
sumFun(1) // 2
```

从代码层面上很好理解，在 sum 函数内返回一个匿名函数，有一个点需要重点注意的是，匿名函数是可以直接访问到外层 sum 函数作用域中的 a 变量，如果你对这个有疑惑，那就要额外补补 js 的作用域链，当然有空我也可以补上来。话不多说，直接上一个该闭包中所涉及到的作用域链，如下：



![scope-chain](https://github.com/Andraw-lin/about-Vue/blob/master/images/scope-chain.jpg)



相信同学们对闭包的一个很重要的特点很熟悉，就是**闭包所涉及到的变量会一直存留在内存中，而不被垃圾回收机制回收**。这也就能说明，为什么执行两次 sumFun 函数后，输出的是 2 而不是 1 ，因为变量 a 会驻留在内存中，因此第二次执行后仍然可以读到前一次所处理的变量 a 的值。

既然闭包中的变量会长期驻留在内存，那会不会导致内存泄漏？

答案是肯定的，不当的闭包处理会让变量无法销毁释放，必要时需要进行手动释放。但是闭包也可以在一些场景下发挥着很好的作用，例如 Memoization 缓存机制。对于一些反复计算，闭包存储的值就可以立马返回，从而避免了反复的操作。

之所以提到闭包中这个缓存功能，是因为今天要探讨的依赖收集功能恰恰就是利用了这一功能进行了实现。



##理解依赖收集 

提起依赖收集，也许会有同学会感到疑惑，依赖收集究竟是一个什么东西？又有什么用处？🤔 

简单点来说，依赖收集就是与之相关联的事物都收集起来。在生活里，其实有很多与该概念相似的场景，比如一台即将发布的新手机，用户若想购买需提前填写手机号码进行预约，当填写完后，手机号就会被收集起来，因此该台新手机就会收集很多与之预约的手机号，等到发布当天，就会逐个通知相关手机号的用户进行购买。在这里面，预约的手机号就是一个依赖，最终会被收集起来。

说了含义，那就得提提用处。同学们应该也比较清楚，既然依赖收集起来，肯定是为了日后有需要时候就拿出来使用。好比如刚刚所说的，如果不收集预约的手机号的话，一旦手机发布后，是无法追踪到相关预约手机号进行通知的。而这就是接下去的章节要提到的依赖更新，总结一下就是，**依赖收集就是为了依赖更新而服务的**。



## 从源码角度进行探究

在上一篇递归处理响应式中有提及过，依赖收集是在 getter 中进行，而依赖更新则是在 setter 中进行。

我们现在来回顾一下`defineReactive$$1`函数，看看依赖收集是如何实现的。

```javascript
function defineReactive$$1 (
 obj,
 key,
 val,
 customSetter,
 shallow
) {
    var dep = new Dep();
   	// ...
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get: function reactiveGetter () {
        // ...
        if (Dep.target) { // 判断是否存在依赖目标
          dep.depend(); // 依赖收集
					// ...
        }
      }
    })
  }
```

由代码可以看到，`defineReactive$$1`函数内实现了闭包，每次属性被使用时，都会调用 getter 函数，而 getter 函数恰恰就是利用了外层函数`defineReactive$$1`中的 dep 变量（也许你会问，dep 是用来干嘛的？稍等一下哈，接下去会详细描述），而该 dep 变量就是用来缓存收集该属性相关依赖的（说白点，就是哪里用到该属性的地方就收集起来 😄 ）。所以每次使用到该属性时，都会先判断与该属性依赖的目标是否存在，一旦存在就立马收集起来。

既然 dep 变量是一个 Dep 对象的实例，那么它究竟是一个什么样的结构的？我们再来看看

```javascript
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
var Dep = function Dep () {
  this.id = uid++;
  this.subs = [];
};

Dep.prototype.addSub = function addSub (sub) {
  this.subs.push(sub);
};

Dep.prototype.depend = function depend () {
  if (Dep.target) {
    Dep.target.addDep(this);
  }
};
```

好明显，Dep 对象中存储有两个属性，其中一个是 id，用来标识用的，而另一个就是数组 subs，该属性就是用来收集依赖用的，其中在 Dep 对象在原型上定义两个方法，分别是 addSub 、depend，都是用来添加依赖用的（当然还定义有其他方法，有兴趣的可以自行研究一下）。

现在来总结一下：

- 当存在 Dep.target 依赖目标时，就会调用 Dep.depend 方法来进行收集，其中 Dep.depend 方法实质就是数组的 push 推进；
- 由于`defineReactive$$1`函数实现的是一个闭包，因此 dep 变量会一直存储在内存中，当有新的地方用到该属性时，就会继续 push 进去，而每次的 push 操作都会保留在 dep 中，直到该属性销毁为止；



## 依赖收集的疑惑

通过上述的描述，我们都大概清楚了只有存在 Dep.target 依赖目标时，才会去进行收集。既然 Dep.target 是用来存储依赖目标的，那么 Dep.target 又是从何而来的呢？接下来我们就来探究一下 Vue 是如何进行处理的。

其实 Vue 从一开始挂载元素时，就已经开始赋值 Dep.target 了，我们来看看是如何实现的：

```javascript
function initMixin(Vue) {
  Vue.prototype._init = function (options) {
    var vm = this;
    // ...
    if (vm.$options.el) { // 判断是否存在挂载元素
      vm.$mount(vm.$options.el); // 若存在就开始进行挂载处理
    }
  };
}

initMixin(Vue)

Vue.prototype.$mount = function (
 el,
 hydrating
) {
  // ...
  return mountComponent(this, el, hydrating) // 直接调用挂载组件方法
};

function mountComponent (
 vm,
 el,
 hydrating
) {
  var updateComponent;
  // ...
  updateComponent = function () {
    vm._update(vm._render(), hydrating); // 更新组件，并进行重新渲染（与依赖收集相关的关键处理）
  };
	// ...
  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, { // 创建一个 Watcher 实例（即依赖）
    before: function before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate');
      }
    }
  }, true /* isRenderWatcher */);
}
```

 从代码上可以看到，元素是在一开始挂载过程中，创建一个 Watcher 实例，也就是一个相关依赖。需注意的是，该依赖是以 template 为单位的（即多个属性都会可能收集同一个 Watcher，形成多对一的关系）。那既然挂载组件的过程中会创建相应的依赖，那么它是如何赋予到 Dep 对象上的？接着看下去：

```javascript
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options,
  isRenderWatcher
) {
  // options
  if (options) {
		// ...
    this.lazy = !!options.lazy; // lazy 用于判断组件是否延迟加载
    // ...
  }
  // ...
	this.value = this.lazy
    ? undefined
  	: this.get(); // get 方法是建立和 Dep 对象的桥梁
}

Watcher.prototype.get = function get () {
  pushTarget(this); // 直接讲 Watcher 赋值到 Dep.target 上
  var value;
  var vm = this.vm;
  try { 
    value = this.getter.call(vm, vm); // 赋值后，开始调用 updateComponent 方法进行渲染并进行依赖收集
  } catch {
    // ...
  } finally {
    // ...
    popTarget(); // 渲染结束并收集依赖后，就重置 Dep.target
    // ...
  }
}

var targetStack = []; // 用来记录保存创建好的 Watcher 实例的栈
function pushTarget(target) {
  targetStack.push(target);
  Dep.target = target; // 赋值到当前 Dep.target 上
}

function popTarget () {
  targetStack.pop(); // 移除当前的 Watcher
  Dep.target = targetStack[targetStack.length - 1]; // Dep.target 永远指向 targetStack 的最后一项
}
```

至此，你应该也能大概了解创建依赖 Watcher 的整个过程，在创建 Watcher 实例的同时并记录下来，然后进行渲染和依赖收集，到最后推出与此相关的 Watcher。

需要注意的是，建立好 Watcher 和 Dep 桥梁后，还需要将 Dep.target 推进相对应响应式属性的 subs 数组中，其中 updateComponent 起到了重要作用，它就是直接触发到 getter 函数进而可以推进到 subs 数组中。

现在再来总结一下：

- 组件在挂载过程中，会创建相对应的 Watcher 实例，并将 Watcher 保存在 Dep.target 中，最好保存到响应式属性的闭包当中，完成依赖收集的过程；
-  通过组件的渲染和依赖收集，将 Dep.target 保存在响应式属性的 subs 数组中，完成后从 targetStack 记录中推出与此相关的 Watcher ；























