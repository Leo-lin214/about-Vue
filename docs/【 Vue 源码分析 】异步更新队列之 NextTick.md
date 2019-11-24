## 所谓的宏任务和微任务

*提到宏任务和微任务，相信对于部分同学来说并不陌生，但还是得简单讲讲（当然关心源码的童鞋们可以直接拉到下面哈 😅）。*

JavaScript 是一个单线程、单进程语言，那么它在运行中的执行过程又是如何的呢？

1. 首先，JavaScript 主进程会作为一个宏任务开始执行，此时，主进程代码就会从上到下开始解释执行。
2. 在解释执行的过程中，当遇到宏任务时，就会将其放入到宏任务队列中，当遇到微任务时，则将其放入到微任务队列中。宏任务队列和微任务队列都是采用队列的数据结构形式进行存储和读取。
3. 当主进程中的代码执行完毕后，会优先检查微任务队列中是否有任务，若有则会按队列的先进先出的原则执行相应代码。
4. 微任务队列执行完毕后，又会开始下一轮的循环，检查宏任务队列，执行下一个宏任务，执行完毕后接着又会检查微任务队列。如此循环，直到宏任务队列以及微任务队列都为空为止。
5. 微任务和宏任务的执行过程就组成了所谓的`Event Loop`。

看完上面，是否会有点懵逼？😅 所以究竟什么是宏任务？什么又是微任务啊？

甭急。接下来就给你看看。

> 宏任务

|         类型          | 浏览器 | Node |
| :-------------------: | :----: | :--: |
|     js 主进程代码     |   ✅    |  ✅   |
|          I/O          |   ✅    |  ✅   |
|       setTimout       |   ✅    |  ✅   |
|      setInterval      |   ✅    |  ✅   |
|     setImmediate      |   ❌    |  ✅   |
| requestAnimationFrame |   ✅    |  ❌   |

注：`requestAnimationFrame` 是要求浏览器在下次重绘之前调用指定的回调函数更新动画，该方法要传入一个回调函数作为参数。[详见MDN关于requestAnimationFrame讲解](https://developer.mozilla.org/zh-CN/docs/Web/API/window/requestAnimationFrame)

> 微任务

|              类型               | 浏览器 | Node |
| :-----------------------------: | :----: | :--: |
| Promise中的then、catch、finally |   ✅    |  ✅   |
|        process.nextTick         |   ❌    |  ✅   |
|          async / await          |   ✅    |  ✅   |
|        MutationObserver         |   ✅    |  ❌   |

注：`MutationObserver` 接口提供了监视DOM树所做更改的能力，属于DOM3 Events规范的一部分。[详见 MDN 关于MutationObserver讲解](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)

为啥要提及宏任务和微任务呢？很简单，因为今天的主角`nextTick`就是基于上述提及的知识点进行实现的。



## NextTick 究竟是啥？又有啥用？

`nextTick`从字面上可理解为下一个钩子，再结合参数为一个回调函数，那么就可以理解**在某个时间段内，触发当前的钩子函数**。

问题来了，某个时间段是在什么时候？

在这里就不卖关子，这个时间段就是使用上述提及的微任务进行实现的，简单地说就是等到主进程的 js 代码执行完毕后，才会去执行该回调函数。

这样子设计有什么好处呢？甭急，先看看官方文档的一个解释。

> Vue 在更新DOM时时异步执行的，只要侦听到数据变化，Vue将开启一个队列，并缓冲在同一事件循环中的所有数据变更，若同一个watcher被多次触发时，只会被推入到队列中一次。

接下来我们就来简单滴解释一下这句话中到底蕴含了多少个知识点哈...🤔

- Vue 在更新DOM时时异步执行的。

  既然 Vue 在更新DOM时是异步的，若此时想获取DOM的某些数据，按同步方式获取肯定是无法获取的。而`nextTick`却是弥补了这一缺陷，开启一个队列，等待宏任务执行完毕后，就会执行该队列中函数（即传进`nextTick`中回调函数）。

  ```javascript
  <div id="app">{{ message }}</div>
  
  const app = new Vue({
    el: '#app',
    data: {
      message: ''
    }
  })
  app.message = 'haha'
  console.log(app.$el.textContent) // ''
  app.nextTick(() => {
    console.log(app.$el.textContent) // 'haha'
  })
  ```

- 多次操作一个数据，DOM 渲染只会执行一次。

  由于 Vue 使用`Object.defineProperty`实现数据响应式的，那么当多次操作一个响应式数据时，就不可避免滴多次执行其`setter`方法。

  尤大大也想到这一点，使用`nextTick`另起一个队列，当多次操作同一个数据时，就根据其`Watcher id`是否已经存在于该队列中，若存在则直接丢掉。这样一来，就可以保证一个数据的多次更改回调只会出现在队列中一次。这样就会有效地提升整个数据响应性能。

  **结合`nextTick`和`Watcher id`就实现了一个多次操作一个数据只会渲染一次的性能提升。🐂**

好啦，既然对`nextTick`有了一个大概的了解，那么我们就来向源码进发。



## 从源码的角度进行探讨

在官方文档，已经提及过`nextTick`的实现，我们来看看。

> Vue 在内部对异步队列尝试使用原生的`Promise.then`、`MutationObserver`、`setImmediate`，如果执行环境不支持，则会采用`setTimeout(fn, 0)`代替。

接下来我们就从源码层面进行探究～🤔

```javascript
renderMixin(Vue)
function renderMixin (Vue) {
  // ...
  Vue.prototype.$nextTick = function (fn) {
    return nextTick(fn, this)
  }
  // ...
}
```

在页面渲染初始化之际，会暴露出`$nextTick API`出来，以方便开发者能够在`nextTick`中获得异步更新的DOM。

上面也提到过，当响应式数据多次更新时，会直接调用`nextTick`来作为一个队列。

```javascript
Watcher.prototype.update = function update () { // 依赖更新操作
  /* istanbul ignore else */
  if (this.lazy) { // 用于计算属性，前面篇节有提及
    this.dirty = true;
  } else if (this.sync) { // 用于判断是否为 watcher，前面篇节也有提及 
    this.run();
  } else { // 除了上述情况外，最后只剩下DOM更新操作了
    queueWatcher(this); // 将依赖推进队列处理函数
  }
}
var queue = [] // 全局DOM依赖Watcher队列
var has = {} // 全局DOM依赖Watcher映射表
var waiting = false // 是否需要等待上一个异步任务执行完毕的标志
function queueWatcher (watcher) {
  var id = watcher.id; // 获取对应依赖的唯一标识id
  if (has[id] == null) { // 判断映射表has中是否存在该id，若有就直接跳走，避免了频繁操作导致不停滴调用回调函数
    has[id] = true;
    if (!flushing) { // 判断全局的队列DOM依赖队列是否为空
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;

      if (!config.async) { // 由于config.async配置默认都是为true，因此可以忽略
        flushSchedulerQueue();
        return
      }
      nextTick(flushSchedulerQueue); // 把DOM依赖回调函数都放进异步队列中
    }
  }
}
```

上述代码可以看到，使用`queue`存储了所有DOM依赖Watcher，同时**使用`has`对象模拟了所有DOM依赖的Watcher映射表，进而避免了多次操作一个响应式属性而导致页面不停渲染**。

上面的代码重点就在于一个`nextTick`函数。接下来我们就来看看它是做点什么的

```javascript
var callbacks = [] // 用于存储DOM依赖Watcher回调函数任务队列
var pending = false // 用于确定是否已经注册微任务
function nextTick (cb, ctx) {
  var _resolve;
  callbacks.push(function () { // 存储DOM依赖Watcher回调函数
    if (cb) {
      try {
        cb.call(ctx); // 每个回调函数都是执行传进来的callback，这里就是flushSchedulerQueue函数（下面会提到）
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  if (!pending) { // 还没注册微任务，因此开始创建微任务
    pending = true; // 每一次都只允许创建一个微任务的任务队列，避免注册过程中误入注册
    timerFunc(); // 创建一个微任务，便于独立于主进程代码
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(function (resolve) {
      _resolve = resolve;
    })
  }
}
```

可以看到，**nextTick 的工作就是创建存储DOM依赖Watcher回调函数任务队列，接着就会判断是否已经注册了微任务，若无注册，则开始注册微任务，并且每次都只允许创建一个微任务**。

接下来我们就继续看看是如何创建微任务的

```javascript
var timerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) { // 优先判断是否支持Promise
  var p = Promise.resolve();
  timerFunc = function () { // 使用Promise创建一个微任务
    p.then(flushCallbacks);
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) { setTimeout(noop); }
  };
  isUsingMicroTask = true;
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) { // 接着判断是否支持MutationObserver
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  var counter = 1;
  var observer = new MutationObserver(flushCallbacks);
  var textNode = document.createTextNode(String(counter));
  observer.observe(textNode, { // 使用MutationObserver来监听DOM节点变化
    characterData: true
  });
  timerFunc = function () { // 执行微任务则会动态创建一个节点来触发MutationObserver变化来调取回调函数
    counter = (counter + 1) % 2;
    textNode.data = String(counter);
  };
  isUsingMicroTask = true;
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) { // 接着判断是否支持setImmediate
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = function () { // 使用setImmediate来创建微任务
    setImmediate(flushCallbacks);
  };
} else { // 若上述都不支持，最后使用setTimeout创建微任务
  // Fallback to setTimeout.
  timerFunc = function () {
    setTimeout(flushCallbacks, 0);
  };
}
```

创建微任务是很简单理解的，符合官方文档的说法，**先是判断是否支持Promise创建、接着判断是否支持MutationObserver创建、再判断是否支持setImmediate创建、到最后才会使用setTimeout来创建微任务**。

到这里是否有点恍然大悟了？🤣 

还没结束呢。`flushCallbacks`函数是干嘛？相信你都知道的，那就是**遍历DOM依赖Watcher任务队列，按顺序来执行其回调函数**。

```javascript
function flushCallbacks () {
  pending = false; // 遍历回调函数任务队列，意味着可以创建微任务
  var copies = callbacks.slice(0); // 浅复制任务队列
  callbacks.length = 0; // 重置全局任务队列为空
  for (var i = 0; i < copies.length; i++) { // 正式遍历全局任务队列
    copies[i](); // 按顺序滴执行每个任务队列的回调函数
  }
}
```

到这里，也许你会有疑问，究竟每一个回调函数里面又是啥？

首先，这个`copies[i]`回调函数会有两种情况

- **内部更改响应式属性，DOM元素重新渲染**（即`flushSchedulerQueue`函数）。
- **外部调用全局`$nextTick`传递进来的回调函数**。

外部调用全局的`$nextTick`没啥好讲的，就是直接调用自定函数即可。现在就来讲讲`flushSchedulerQueue`函数，我们来看看它是做点什么的。

```javascript
/**
 * Flush both queues and run the watchers.
 * 无非就是将queues中的watcher进行遍历以及执行每个watcher的回调函数
 */
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow(); // 获取当前时间戳
  flushing = true; // 已经在遍历DOM依赖的Watcher队列，不能再添加Watcher进入队列
  var watcher, id;

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort(function (a, b) { return a.id - b.id; }); // 使用快排的方式进行排序Watcher队列

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) { // 开始遍历
    watcher = queue[index];
    if (watcher.before) {
      watcher.before();
    }
    id = watcher.id; // 获取每个Watcher的id
    has[id] = null; // 并设置响应的映射表为空值，允许下一轮的更新
    watcher.run(); // 获取最新的响应式属性值并更新DOM
  }

  // keep copies of post queues before resetting state
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice(); // 浅复制Watcher队列，新建一个更新队列

  resetSchedulerState(); // 重置异步队列状态

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue); // 调用更新钩子函数
  // ...
}

/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () { // 重置包括异步队列、Watcher映射表、异步队列函数都执行完毕标志、异步队列都遍历完毕标志
  index = queue.length = activatedChildren.length = 0;
  has = {};
  {
    circular = {};
  }
  waiting = flushing = false;
}
```

很简单，由于响应式属性的更改，其实是需要重新渲染的，因此使用了一个异步队列遍历后并巧妙结合一个全局映射表来让页面只会渲染一次，避免了多次频繁操作数据而导致的页面不断渲染，有效地将性能得到提升。

现在就来总结一下：

- `nextTick`中回调函数会使用微任务来包裹，独立于主进程运行。
- 使用映射表`has`来结合到异步队列上，有效避免多次频繁操作数据而导致的页面不断渲染，只会执行一次效果。































