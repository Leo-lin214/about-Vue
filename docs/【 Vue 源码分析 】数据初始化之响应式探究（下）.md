## 复杂数据类型初始化时的响应式

接着上一篇的内容，由于我们定义数据时都是以一个基本数据类型来进行讲解的，如下：

```javascript
data: {
  text: 'hello world'
}
```

通过`Object.defineProperty`方法，将 text 直接挂载在 Vue 实例上，读取和修改都会通知到 data 对象同名属性值。当然这样基本数据类型是不会有问题的，但是一旦遇到复杂数据类型会不会是一样？如下：

```javascript
data: {
  text: {
    value: 'hello world'
  }
}
```

很显然，通过`this.text.value`或`app.text.value`是可以直接读取到的，但是对于设置值时，如下：

```javascript
app.text.value = 'haha'
```

这样设置值后，使用`this.text.value`或`app.text.value`依然是可以能拿到最新的值，因为`this.text`或`app.text`指向的地址没变。我就不卖关子了，**接下来要讨论的依赖收集和依赖更新是分别在 get 和 set 中进行的**，同学们都可以提前知道一下。

上述的设置值是无法触发到 set 方法的，进而无法通知依赖进行相对应的更新。上一篇我也说过，**使用`Object.defineProperty`实现响应式系统，并将 data 对象中第一层数据挂载在 Vue 实例上**。其中第一层数据指的就是 text 对象挂载在 Vue 实例上，而对于第二层数据 value 则没有挂载（也就是无法响应式的）。这也就是我为什么分上下两篇进行讲解的原因，第一层数据若是基本数据类型时，则可以直接挂载在 Vue 实例上进行响应式，若是复杂数据类型时，则无法对其第二层进行响应式处理。

既然第二层数据无法响应式，那么重新让它响应式不就得了？答案是对的，那么该如何处理？Vue 中使用的方法是**递归**，通过递归来一层一层地进行响应式处理。



## 递归响应式处理

接着上一篇未完待续中的源码分析，让我们看看源码中是如何进行递归实现的。（同样地，我只会贴出相关主题的代码，其他代码同学们有空就可以进行探究 🤔 ）：

```javascript
function initData(vm) {
  var data = vm.$options.data
  // ...挂载 data 中元素值到 Vue 实例中
  observe(data, true) // 递归 data 中数据实现响应式处理
}

function observe(value, asRootData) { // asRootData 用于判断是否为根数据
  if (!isObject(value) || value instanceof VNode) { // 可用于递归到最后一层数据时，当不再是 Object 时，就直接返回
    return
  }
  // ...
  ob = new Observer(value); // 基于 Observer 构造函数实现响应式
}

var Observer = function Observer (value) {
  this.value = value;
  // ...
  if (Array.isArray(value)) { // 判断是否为数组，若是则做进一步的处理
    // ...
  } else {
    this.walk(value); // 将 data 中的属性转化为 getter 或 setter
  }
}

Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) { // 对 data 中的值进行遍历，然后作进一步的处理
    defineReactive$$1(obj, keys[i]);
  }
}

function defineReactive$$1 ( // 将对象上的属性定义为响应式
	obj,
	key,
	val,
	customSetter,
	shallow
) {
	// ...
  var childOb = !shallow && observe(val); // 递归的关键（即判断该属性是否为对象，若是则继续递归下去，直到遍历到的属性不再是 object 为止）
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 待续.................................（不卖关子，依赖收集所用）
      return value
    },
    set: function reactiveSetter (newVal) {
      // 待续.................................（依赖更新所用）
    }
  })
}
```

在代码上你可以看到我的标注，在最终的将对象上属性定义为响应式 defineReactive$$1 方法上，再一次对传递进来的属性进行 observe 递归，由于在 observe 方法上会判断该属性是否为对象，若是，则继续赋予响应式，直到传递进 observe 方法中的属性不再是一个对象为止。

从源代码层面其实很好理解，无非就是遍历 data 中的对象，从第一层属性开始，若是基本数据类型，则直接通过 Object.defineProperty 方法来为其赋予响应式，若是复杂数据类型，则再次递归它，直到递归到的属性不再是复杂数据类型为止。

这篇其实内容不多，也挺好理解。讲了那么多，还是得总结一下 🌚 ：

- 依然使用`Object.defineProperty`方法来进行响应式处理，对于嵌套的对象，会使用递归的方式进行处理，直到递归到的数据类型为基本数据类型为止；





























