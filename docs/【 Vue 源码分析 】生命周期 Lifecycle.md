## 简单聊聊生命周期

对生命周期，相信各位都不陌生。在生命周期钩子函数中，可以做一些数据初始化、DOM操作等等。下面就借用一下官方给出的生命周期图示。

![Vue官方的生命周期图示](https://cn.vuejs.org/images/lifecycle.png)

官方给出的图很好理解，无非就是当程序执行到每一个阶段时，都会调用一个钩子函数，便于处理一些`javascript`业务逻辑。调用顺序如下：

```javascript
// 元素挂载时
beforeCreate --> created --> beforeMount --> mounted

// 元素更新时
beforeUpdate --> updated

// 元素销毁时
beforeDestroy --> destroyed
```



## 从源码的角度进行分析

在开始分析之前，不得不提一下上一章节的[【 Vue 源码分析 】混合 Mixin（上）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/[%20Vue%20源码分析%20]混合%20Mixin(上).md)中提及的生命周期合并。拥有`Mixin`的组件，会先执行`Mixin`中的生命周期钩子函数，最好再执行组件本身的生命周期钩子。

为什么要这样处理？相信很多同学都会都类似的提问，这是因为`Mixin`作为一个组件间的抽象层，一般都会处理一些配置信息、数据初始化等操作。（如果有不了解的同学，我都建议先看看上两篇的关于`Mixin`的讲解）。

其实对于生命周期的讲解，相信同学们看一下源码都会基本知道是怎么一回事，无非就是在一定的阶段直接调用相应的钩子函数即可。（哈哈，是不是很简单 😄。。）下面就进行一个简单讲解。

```javascript
initLifecycle(vm);

function initLifecycle (vm) {
  var options = vm.$options;
  // ...
  vm.$children = []; // 初始化子项
  vm.$refs = {}; // 初始化相关联的DOM元素

  vm._watcher = null; // 初始化Wacther依赖
  vm._inactive = null;
  vm._directInactive = false;
  vm._isMounted = false; // 是否已经执行挂载钩子函数标识
  vm._isDestroyed = false; // 是否执行销毁钩子函数标识
  vm._isBeingDestroyed = false; // 是否正处于销毁中标识
}
```

在初始生命周期上，可以看到先设置一些相关配置项，如子项、依赖、挂载钩子执行标识以及销毁钩子执行标识等等。

也许你会问这些标识有什么用？其实很简单，拥有这些标识就相当于对生命周期进行一个判断，然后才允许去做一系列的逻辑处理，从而避免了生命周期逻辑处理交叉。

既然初始化了生命周期的配置，那么生命周期的钩子函数又是如何调用的呢？

在这里，我就不卖关子了，其实在所有生命周期钩子函数执行前，都会先`Mixin`混合起来，然后在每一个阶段，通过调用`callHook`函数，来执行相应阶段生命周期的钩子函数。那么我们就来看看`callHook`是咋样的 🤔

```javascript
function callHook (vm, hook) {
  pushTarget();
  var handlers = vm.$options[hook]; // 先在实例上拿到对应合并好的生命周期钩子函数
  var info = hook + " hook"; // 标记生命周期阶段信息文案
  if (handlers) { // 当存在合并好的生命周期钩子函数时，就遍历执行
    for (var i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info); // 执行函数并处理错误
    }
  }
  if (vm._hasHookEvent) { // 针对事件的处理，暂不讲解
    vm.$emit('hook:' + hook);
  }
  popTarget();
}

function invokeWithErrorHandling (
 handler,
 context,
 args,
 vm,
 info
) {
  var res;
  try {
    res = args ? handler.apply(context, args) : handler.call(context); // 绑定上下文执行生命周期钩子函数
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(function (e) { return handleError(e, vm, info + " (Promise/async)"); });
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true;
    }
  } catch (e) {
    handleError(e, vm, info);
  }
  return res
}

// 不同阶段调用时，如
callHook(vm, 'beforeCreate');
```

从上面可以看到，通过一个`callHook`函数来执行合并好的生命周期函数，并处理相应的错误。

总结：生命周期在理解上并不难，主要是搞清楚其怎么初始化、怎么调用即可。





































