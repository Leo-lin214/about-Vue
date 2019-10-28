## 数据响应式的疑惑

先看下 Vue 中编写一个 hello word 的代码： 

```javascript
<div id="app">{{ text }}</div>

<script src="./vue.js"></script>
<script>
	const app = new Vue({
    el: '#app',
    data: {
      text: 'hello world'
    }
  })
</script>
```

上述是一个很简单的栗子🌰，打开网页就可以看到一个简单的 hello word 文案，很明显，`{{ text }}`被解析到下面数据初始化时定义的 text 值。当我们作如下处理时：

```javascript
app.text = 'haha'
```

会发现，模板上的`{{ text }}`值同样被更新为 haha 值，官方文档其实说的很清楚，**当一个 Vue 实例被创建时，它将 data 对象中的所有属性加入到 Vue 的响应式系统中**。这句话理解起来其实很简单，就是定义在 data 对象总的数据都会是响应式的，这也就能够说明，为什么直接操作`app.text = 'haha'`时，模板上的相对应的响应值会跟着变化。

当然，上述只是一个宏观解析，抱有好奇心的你们当然不能满足，更期待的是一个微观解析。那官方所说的响应式系统，在 Vue 实例被创建和初始化数据时是如何定义成响应式的呢？？下面就会进入我们的🐷题。



## 从源码角度探究

下面先看下 Vue 源码是如何实现（下方只会编写出与该主题相关代码，当然还有很多代码没贴出，各位有空可以去探究一下 😄 ）：

```javascript
function initMixin(Vue) {
  // ...
  initState(vm); // 初始化状态
}

initMixin(Vue)

function initState(vm) {
  // ..
  var opts = vm.$options
  if (opts.data) { // 判断 data 字段中是否有值，一旦有值初始化数据
    initData(vm)
  }
}

function initData(vm) {
  var data = vm.$options.data
  data = vm._data = typeof data === 'function' ? getData(data, vm) : data || {} // 旧版 data 可赋值为一个返回对象的函数，新版则直接支持了对象赋值，一旦 data 不存在时，直接初始化为一个 {} 空对象
  if (!isPlainObject(data)) { // 判断定义的 data 不是对象时会抛出警告⚠️，并同时初始化为一个 {} 空对象
    data = {};
    warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }
  var keys = Object.keys(data);
  var i = keys.length;
  while (i--) {
    var key = keys[i]
    // ...
    } else if (!isReserved(key)) {
      proxy(vm, "_data", key);
    }
  }
	// 待续............................（一会就待续 🤔 ）
}

function isReserved (str) { // 判断字符串的第一个字符是否为 $ 或 _ 
  var c = (str + '').charCodeAt(0);
  return c === 0x24 || c === 0x5F
}

var sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
};

function proxy (target, sourceKey, key) { // 将 data 中属性挂载在 Vue 实例上
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  };
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

上述代码希望各位都可以花点时间仔细看下（因为上述代码可以很好解析在 Vue 实例上的数据是响应式的），看似很多代码，却是很好理解的，并且我也在上述贴出了相对应的注释。接下来就针对上述代码层面，分析关键点：

1. `vm.$options.data`合法性判断

   初始化数据时，定义的 data 可以是函数，也可以是一个对象。一旦不是函数或对象时，都会抛出一个警告⚠️，同时初始化 data 为一个空对象。

   ```javascript
   // 合法定义
   data() {
     return {
       text: 'hello world'
     }
   }
   
   data: {
     text: 'hello world'
   }
   
   // 不合法定义
   data: 'hello world'
   ```

   当 data 定义为函数时，会直接调用该函数并返回一个对象挂载在实例上的，这就比直接定义为对象多了一步执行函数而已。

   

2. isReserved

   上面注释写的很清楚，该方法就是**用于判断字符串的第一个字符是否为 $ 或 _** 。也许有同学会问，这方法判断的意义在何处？

   其实在[官方文档](https://cn.vuejs.org/v2/api/#data)都写的很清楚，**以 `_` 或 `$` 开头的属性 不会 被 Vue 实例代理，因为它们可能和 Vue 内置的属性、API 方法冲突**。

   这也就能够解析为啥要加上该方法判断了 🤔 ，就是避免和内置属性产生冲突。

   

3. proxy

   实现 Vue 实例上访问数据的响应式系统的关键函数，**使用`Object.defineProperty` ES5 方法来实现访问器属性的响应式**。

   我们都知道，对象属性分为两种，分别是数据属性和访问器属性。其中数据属性和我们平时直接操作的变量没什么区别，而访问器属性则是相当于中间加了一层代理，当访问该属性时，需经过代理 getter 获取（此时可以进行依赖的收集，后面章节会提到），当改变该属性，也需经过代理 setter 进行设置。

   说了那么多，到底怎么用啊？？别的不说 🙊，直接来一波 🌰 体验一下：

   ```javascript
    var book = {
      year: 2008
    }
    Object.defineProperty(book, '_year', {
      get() {
        return book.year
      },
      set(val) {
        book.year = val
      }
    })
    console.log(book._year) // 2008
    book._year = 2019
    console.log(book.year) // 2019
   ```

   可以看到，每次访问`book.year`时会经过`get`方法获取真正`_year`，而每次更改`book.year`时都会经过`set`方法来操作真正的`_year`。

   回到正题，proxy 方法就是将 data 中每一个定义的属性挂载在 Vue 实例上，这样每次访问`app.text`时其实就是直接读取了 data 对象中属性，同样地，每次更改`app.text`时就相当于直接更改了 data 对象中定义的属性。

   因此，proxy 方法也就可以说明为什么在实例上可以直接访问到 data 对象中的数据属性以及更改其值。（当然也包括了为什么 this.text 也是可以直接访问到 data 对象中的数据属性以及更改其值）。



讲了那么多，来总结下：

- Vue 中 data 的定义既可以是对象，也可以是函数。一旦不是对象或函数，就会抛出警告⚠️，并初始化 data 为一个空对象，代码还是能执行；
- 在 data 对象中定义数据时，变量名绝对不能加上 $ 或 _ 作为开头；
- 使用`Object.defineProperty`实现响应式系统，并将 data 对象中第一层数据挂载在 Vue 实例上，以便通过`this.text`或`app.text`来访问 data 对象中的数据；



讲到这里，对于 Vue 实现数据初始化的响应式，也许你已经有点头绪了 🤔 。既然如此，是不是已经讲完了？NoNo～

对于一开始的疑惑，当我们的直接操作`app.text = 'haha'`时，为什么模板上的相对应的响应值会跟着变化？这个问题还是没回答啊。是的，要讲解这个问题，还得继续看下去，因为其中涉及到了依赖收集以及依赖更新内容。

就好比如，iphone11 快出来了，这时候官网放出了预定功能，消费者可在上面输入手机号进行预订，当iphone11出来后，就会通知该手机号，消费者就可以直接给钱购买了。同样地，直接操作`app.text = 'haha'`时，需通知依赖们，来更新他们订阅的值。

















