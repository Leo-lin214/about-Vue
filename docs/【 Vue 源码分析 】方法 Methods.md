## 先聊聊call、apply和bind

相信同学们对于 call 和 apply 的区别应该不陌生，毕竟面试时常常问到的问题。那么我们现在就再来巩固一下，我们先来看个简单的栗子🌰：

```javascript
var obj = {
  num: 0,
  addNum(arg) {
    return this.num + arg
  }
}
var newObj = {
  num: 10
}
console.log(obj.addNum(1)) // 1
console.log(obj.addNum.call(newObj, 1)) // 11
console.log(obj.addNum.apply(newObj, [1])) // 11
```

代码很好理解，我们都知道，函数内部存在一个执行上下文，当我们直接执行`obj.addNum`时，this 指向的就是 obj 对象，因此输出也就是 1 。

**call、apply 和 bind 一个共同点就是改变函数的执行上下文到指定的对象中**。

上述，执行`obj.call(newObj, 1)`和`obj.apply(newObj, [1])`，相当于将方法`addNum`中的 this 直接指向了 newObj 对象，因此都会输出 11。另外，你会发现，call 和 apply 方法的第二个参数会有所不同，其中 **call 是一个一个参数传递，而 apply 则是使用数组进行传递**。因此当有多个参数需要传递时，可直接使用 apply 或者 在 call 上使用扩展元算符，如下：

```javascript
var arr = [1, 2, 3, 4, 5, 6]
obj.addNum.call(newObj, ...arr)
// 相当于
obj.addNum.apply(newObj, arr)
```

因此，**call 和 apply 的区别就是，call 传递参数时是一个一个地进行传递，而 apply 则是使用数组方式进行传递。**

既然，call 和 apply 都能改变执行上下文，那么看看下面一个栗子🌰：

```javascript
var num = 100
var obj = {
  num: 0,
  addNum(arg) {
    return this.num + arg
  }
}
console.log(obj.addNum.call(null, 1))
console.log(obj.addNum.apply(undefined, 1))
```

相信很多同学都见过，在使用 call 和 apply 时，第一个参数使用 null 或者 undefined，执行上下文会指向哪里？

在这里我就不卖关子了，**当使用 call 和 apply 时，第一个参数使用 null 或者 undefined，执行上下文会直接指向 window 对象。**因此，最后输出的结果都是 101。

看到这里，或许你会迷惑为什么就没提到 bind ？

虽然 bind 同样也是改变函数的执行上下文，但它却和 call 、apply 不一样，那就是绑定好执行上下文后，重新生成一个新的函数，而这个函数中执行上下文已经绑定好了并不会变。我们来看看：

```javascript
var obj = {
  num: 0,
  addNum(arg) {
    return this.num + arg
  }
}
var newObj = {
  num: 10
}
var newObjAddNum = obj.addNum.bind(newObj)
console.log(newObjAddNum(1)) // 11
```

可以看到的是，使用 bind 后，相当于帮你准备好目标对象了，即绑定好执行上下文到指定对象中去，并最终返回一个新的函数，对原来的函数并不造成任何的影响，而这也符合了纯函数思想。

现在就来总结一下：

- bind 和 call 、apply 之间的区别：

  bind 是改变执行上下文并返回一个新的函数，而 call 和 apply 则是改变执行上下文并立即执行。另外，**bind 会有浏览器的兼容问题，它不兼容 IE6~8。**

- call 和 apply 之间的区别：

  call 传递参数时是一个一个地进行传递，而 apply 则是使用数组方式进行传递。



既然聊到 call、apply 和 bind ，想必跟本主题是有很大的关系。没错，Vue 源码中绑定方法到实例上时，使用的就是 call、apply 以及 bind。让我们接着下去看看。



## 从源码的角度进行分析

在 Vue 源码中，实现方法挂载到 Vue 实例上，使用的是 bind 方法，但我们都知道，bind 方法会有兼容性问题，为避免这个问题，不得不使用 call 和 apply 来实现补全功能。让我们看看源码是如何实现的 🤔：

```javascript
initState(vm);
function initState (vm) {
  // ...
  if (opts.methods) { initMethods(vm, opts.methods); } // 初始化方法
  // ...
}

function initMethods (vm, methods) {
  var props = vm.$options.props;
  for (var key in methods) {
    {
      if (typeof methods[key] !== 'function') { // 判断方法名是否定义为函数类型
        warn(
          "Method \"" + key + "\" has type \"" + (typeof methods[key]) + "\" in the component definition. " +
          "Did you reference the function correctly?",
          vm
        );
      }
      if (props && hasOwn(props, key)) { // 判断方法名是否已经个props中有同名冲突
        warn(
          ("Method \"" + key + "\" has already been defined as a prop."),
          vm
        );
      }
      if ((key in vm) && isReserved(key)) { // 判断方法名是否与Vue实例中有同名冲突
        warn(
          "Method \"" + key + "\" conflicts with an existing Vue instance method. " +
          "Avoid defining component methods that start with _ or $."
        );
      }
    }
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm); // bind 函数就是关键绑定函数
  }
}
```

上面代码中，可以看到方法`initMethods`其实就是在判断定义的方法是否为函数，以及是否和 props 、Vue 实例上会有同名冲突，一旦存在，就会抛出警告，但是你会发现程序还是得会走下去。

另外，当定义的方法是不同名并为函数时，那么就会执行 bind 方法来将方法挂载到 Vue 实例上，让我们来看看该方法是如何操作的：

```javascript
function polyfillBind (fn, ctx) {
  function boundFn (a) {
    var l = arguments.length;
    return l
      ? l > 1
      ? fn.apply(ctx, arguments)
    : fn.call(ctx, a)
    : fn.call(ctx)
  }

  boundFn._length = fn.length;
  return boundFn
}

function nativeBind (fn, ctx) {
  return fn.bind(ctx)
}

var bind = Function.prototype.bind
  ? nativeBind // 对于支持 bind 方法的环境，直接使用原生的 bind 方法
  : polyfillBind; // 对于不支持 bind 方法的环境，则使用兼容方法来进行兼容

```

上面我们提及过，bind 方法会永久地绑定好执行上下文后，并返回一个新的方法。这样会有一个好处，那就是在方法里就不再需要管理执行上下文究竟指向哪一个对象，同时也方便后期的管理。

另外，对于不支持原生 bind 方法的环境，使用方法`polyfillBind`会根据传递进来的参数长度来决定使用 call 和 apply，这是为什么？

原因就是，call 方法需要一个参数一个参数地进行传递，而 apply 则可以直接使用数组形式进行传递。最后返回一个新的函数，这样一来就能兼容到所有不支持 bind 方法的环境。在 Vue 源码实现 Methods 中，上述的绑定代码其实很好理解，也是最简单的实现方式。

总结一下：

- Vue 源码初始化方法 Methods 中，**使用 bind 方法来永久绑定执行上下文并返回一个新的函数到 Vue 实例中**，而**对于不支持 bind 原生方法的环境，则直接使用 call 或 apply 方法来进行兼容**。





















