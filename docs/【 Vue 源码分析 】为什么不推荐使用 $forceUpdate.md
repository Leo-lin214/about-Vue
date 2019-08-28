## 从响应式对象新增属性说起

我们先看一个日常的栗子 🌰 ：

```javascript
// html
<div id="app">
	<span v-show="message.isShow">{{ message.value }}</span>
	<button @click="showTest">Show Message...</button>
	<button @click="hideTest">Hide Message...</button>
</div>

// js
const app = new Vue({
  el: '#app',
  data: {
    message: {
      value: 'Display a message...'
    }
  },
  methods: {
    showTest() {
      this.message.isShow = true
    },
    hideTest() {
      this.message.isShow = false
    }
  }
})
```

我们都知道，当我们点击展示信息的按钮时，会发现信息组件是不会展示出来。官方文档也有说过，**Vue 不能检测对象属性的添加或删除**。

那么，若我想点击按钮时候，界面上信息组件能按要求展示与否时，该如何处理呢？

这里就不卖关子，其实可以有三种处理方案：

- 设置响应式对象中`isShow`属性为响应式
- 全局`$set`方法
- `$forceUpdate`强制更新

官方中提及过，前两种方式是极其推荐的方式，而最后一种方法则是不推荐的，让我们来看看官方对`$forceUpdate`的评价：

> 如果你发现你自己需要在 Vue 中做一次强制更新，99.9% 的情况，是你在某个地方做错了事。

😂，官方说的真的很直接，除非万不得已的地步，都是永远不推荐使用`$forceUpdate`方法，那到底是为什么呢？接下来让我们从源码中去看看。



## 从源码的角度分析

```javascript
Vue.prototype.$forceUpdate = function () {
  var vm = this; // 获取当前 Vue 实例
  if (vm._watcher) {
    vm._watcher.update(); // 只对当前的 Vue 实例进行更新操作
  }
}
```

如果看过我上一篇对于依赖更新讲解的同学，应该就很好理解这部分代码。调用`$forceUpdate`方法时，会先获取当前 Vue 实例，然后判断当前实例上是否存在 Watcher 依赖，若存在则直接调用其 update 方法来通知每一个依赖执行相应的更新回调函数。

那么问题来了，Vue 实例上的`_watcher`是在什么时候设置的？

其实在组件挂载并创建相应的 Watcher 依赖时，就已经将当前依赖挂载到当前 Vue 实例上，让我们看看源码的实现：

```javascript
function mountComponent (
	vm,
 	el,
 	hydrating
) {
    // ...
    new Watcher(vm, updateComponent, noop, {
      before: function before () {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate');
        }
      }
    }, true /* isRenderWatcher */);
    // ...
  }

var Watcher = function Watcher (
	vm,
 	expOrFn,
 	cb,
 	options,
 	isRenderWatcher
) {
  	this.vm = vm;
    if (isRenderWatcher) { // 挂载组件时，默认都会为 true
      vm._watcher = this; // 将当前 Watcher 依赖设置到 Vue 实例上
    }
    // ...
  }
    
```

至此，一个`$forceUpdate`的实现思想你应该也有了印象。那么回到正题，你是否能看到官方还是不建议使用`$forceUpdate`方法么？

按我的理解，对比上一篇提及的全局`$set`方法，它和`$forceUpdate`有一个共同点，就是通知依赖进行更新。但是你有没有发现全局`$set`方法有一个点是`$forceUpdate`方法没有的么？

没错，那就是**设置新增的属性为响应式** 🤔 。从`$forceUpdate`方法实现上，你会发现，由此至终它只做一件事情，那就是通知 Watcher 依赖进行更新。如果对于新增属性的处理时，由于它不会将新增的属性重新设置为响应式，因此下一次若想操作新增属性时，还会需要重新调用一次`$forceUpdate`方法。另一方面，对于这类非响应式的属性的后期管理也是个大问题，当数据变得臃肿时，对于哪些是响应式属性，哪些是非响应式属性，以及是否需要调用`$forceUpdate`方法都会是一个考验开发者的难题。

直到现在，我依然觉得`$forceUpdate`方法更像我们生活里的一次性工具，用完就完事了，不需要考虑其他事情，当下次再需要做事时，再用一次新的一次性工具，用完即丢掉。但是你有没有想过，一次性工具是没有循环利用，只会导致资源的浪费～～😼。与此相反的，就是响应式，不管如何使用都会是同一样东西，刚好就符合了循环利用的目的。

好了，现在我们来总结一下不推荐使用`$forceUpdate`方法原因：

- `$forceUpdate`方法无法设置新增属性为响应式；
- `$forceUpdate`方法不利于后期的变量维护，对于项目中是否需要调用是不可控的；

































