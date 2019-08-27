## Object.defineProperty 的缺陷

相信同学们都知道原生 JS 提供的`Object.defineProperty`方法存在着缺陷，在某些场景下，对对象的操作是无法触发到其 Setter 方法的，先看个栗子 🌰 体验一下：

```javascript
var book = {
  _config: {}
}
Object.defineProperty(book, 'config', {
  get() {
    return book._config
  },
  set(val) {
  	book._config = val
	}
})

book.config = { name: 'haha' }
console.log(book.config) // { name: 'haha' }
book.config.year = '2019'
console.log(book.config) // { name: 'haha', year: '2019' }
```

代码其实很好理解，由于属性 _config 被设置为响应式，当我们直接初始化其值时，会直接调用到 Setter 方法来进行赋值。但是对于新增属性时，会发现并没有触发到 Setter 方法，为什么呢？如果是这样的话，那为何打印出来的 book.config 中还会出现 year 属性呢？

属性 config 在一开始进行响应式时，是不会对其内部属性进行统一的响应式处理的，因此对 _config 属性内部新增一个新的属性时，很显然是不会直接触发到 Setter 函数，这样一来，当 template 模板上使用该新增的属性时，同样也无法触发到 Getter 函数。

为此，既然新增属性无法进行响应式处理，那如何去收集其依赖呢？

这里就不卖关子了，**Vue 直接在父级响应式对象里添加一个不可枚举属性`__ob__`，专门用来存储该响应式的相关的 Watcher 依赖的**。由于`__ob__`存储着相对应的 Watcher 依赖，官方文档就提供了全局`$set`方法来更新，而该方法就是利用属性`__ob__`去寻找对应的依赖的，并进行更新。

另外，除了新增属性会出现这种情况外，官方文档还提及到：

1. Vue 不能检测对象属性的添加或删除；
2. Vue 不能检测以下数组的变动：
   - 当你利用索引直接设置一个数组项时，例如：`vm.items[indexOfItem] = newValue`;
   - 当你修改数组的长度时，例如：`vm.items.length = newLength`;

接下来，我们就来看看是如何通过`__ob__`属性进行存储响应的依赖的。



##  从源码进行分析

Vue 在初始化响应式 $data 时，就已经开始在每个复杂数据类型下定义一个不可枚举属性`__ob__`，再结合递归操作，就能保证所有复杂数据类型下都会有一个`__ob__`依赖项。需要注意的是，Vue 对于对象和数组的处理会有不一样的处理，我会分开来探讨一下，先看看初始化时是如何进行定义的：

```javascript
function initData (vm) {
  var data = vm.$options.data;
  // ...
  // observe data
  observe(data, true /* asRootData */); // 调用响应式处理方法
}

function observe (value, asRootData) {
  // ...
  var ob;
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) { // 关键点，递归过程中，一旦有复杂数据类型存在__ob__属性，就立马返回
    ob = value.__ob__;
  } else if (...) {
		ob = new Observer(value);
  }
	// ...
	return ob
}

var Observer = function Observer (value) {
  this.value = value;
  this.dep = new Dep(); // 用来存储依赖所用
  // ...
  def(value, '__ob__', this); // 关键点，在响应式复杂类型上定义一个__ob__属性，并且配置为不可枚举
  if (Array.isArray(value)) {
    // .......................待续（数组的处理，接下去会讲到）
  } else {
    this.walk(value);
  }
}

/**
 * Define a property.
 */
function def (obj, key, val, enumerable) { // 定义属性并且配置为不可枚举
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  });
}

Observer.prototype.walk = function walk (obj) { // 逐个遍历响应式复杂数据类型中属性，并且进行响应式处理
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive$$1(obj, keys[i]);
  }
};

function defineReactive$$1 (
	obj,
 	key,
 	val,
 	customSetter,
 	shallow
) {
    // ...
    var childOb = !shallow && observe(val); // 递归处理，最终返回子组件的__ob__，用来存储父级组件的依赖用的
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,
      get: function reactiveGetter () {
        // ...
        if (Dep.target) {
          dep.depend();
          if (childOb) {
            childOb.dep.depend(); // 子组件存储父组件的依赖
            // .............................待续（数组处理，接下来会讲到）
          }
        }
      },
      // ...
    })
  }

```

看完会不会觉得有点懵？现在我来简单总结一下，初始化时，从`$data`开始，由于`$data`对象本身就是一个复杂数据类型，因此会先新增一个`__ob__`属性，紧接着就开始递归遍历`$data`中属性，一旦遇到复杂数据类型就会立马添加一个`__ob__`属性，其中`__ob__`属性中的 dep 变量就是用来存储依赖 Watcher 用的。

我们都知道一开始添加的`__ob__`属性中的变量 dep 存储的依赖 Watcher 依赖肯定是空的，那么依赖会从哪里开始存储，答案就是在`Object.defineProperty`方法中的 Getter 函数，当每次递归回来的变量`childOb`有值时，即相当于返回了子组件中的`__ob__`属性，这时候除了缓存一份到自身的 dep 中外，还会把其依赖 Watcher 直接存储到子组件的`__ob__`属性的 dep 变量中去。

这样一来，就能说明为何子组件更新时，也会触发到父组件的更新。这就是因为子组件一直存储有父组件的 Watcher 依赖。

1. **Obejct**

   其实上述讲述的，就是以一个 Object 对象来讲解的，先递归遍历添加`__ob__`属性，然后递归返回时在`__ob__`属性中存储父组件的相关依赖。

2. **Array**

   上述代码中我已经屏蔽了数组的处理，意在避免和对象的处理混淆。接下来我们来看看数组会是如何处理的：

   ```javascript
   var Observer = function Observer (value) {
     def(value, '__ob__', this); // 关键点，在响应式复杂类型上定义一个__ob__属性，并且配置为不可枚举
     if (Array.isArray(value)) {
       // ...
       this.observeArray(value);
     }
     // ...
   }
   
   /**
    * Observe a list of Array items.
    */
   Observer.prototype.observeArray = function observeArray (items) {
     for (var i = 0, l = items.length; i < l; i++) {
       observe(items[i]);
     }
   };
   
   ```

   可以看到，对于数组类型的属性，会调用方法`observeArray`来对每一项的值进行响应式处理，包括遇到复杂数据类型时就添加一个`__ob__`属性，这跟对象递归遍历其实是很类似的。再看看在 Getter 的处理：

   ```javascript
   function defineReactive$$1 (
   	obj,
    	key,
    	val,
    	customSetter,
    	shallow
   ) {
       // ...
       var childOb = !shallow && observe(val); // 递归处理，最终返回子组件的__ob__，用来存储父级组件的依赖用的
       Object.defineProperty(obj, key, {
         enumerable: true,
         configurable: true,
         get: function reactiveGetter () {
           // ...
           if (Dep.target) {
             dep.depend();
             if (childOb) {
               childOb.dep.depend(); // 子组件存储父组件的依赖
               if (Array.isArray(value)) {
                 dependArray(value);
               }
             }
           }
         },
         // ...
       })
     }
   
   function dependArray (value) {
     for (var e = (void 0), i = 0, l = value.length; i < l; i++) {
       e = value[i];
       e && e.__ob__ && e.__ob__.dep.depend();
       if (Array.isArray(e)) {
         dependArray(e);
       }
     }
   }
   ```

   同样地，当遇到数组类型时，会调用方法`dependArray`来进行递归处理，向数组中每一个复杂数据类型添加父层数据的依赖 Watcher 。而遇到基础数据类型时，就会直接跳过。

   

好了，现在我们来总结一下：

- 对于 Vue 不能监测的行为，都会利用`__ob__`来进行存储依赖；
- 官方提供的全局`$set`方法，其实就是利用`__ob__`中依赖进行设置和更新的（下一篇依赖更新中会提及）；
- 针对对象和数组类型，默认情况都会是直接遍历对象中的属性进行响应式处理，而数组则需要提前遍历，遇到数组则继续遍历，遇到对象则直接递归处理其属性，遇到基础数据类型时就会直接跳过；











