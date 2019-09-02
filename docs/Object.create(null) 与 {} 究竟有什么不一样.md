## 创建对象

我们都知道，创建一个 JavaScript 对象，可以有如下几种方法，分别是：

- 通过`Object`实例化创建；

  ```javascript
  var obj = new Object()
  obj.name = 'Andraw'
  obj.getName = function() {
    return this.name
  }
  ```

- 使用对象子面量方法；

  ```javascript
  var obj = {
    name: 'Andraw',
    getName: function() {
      return this.name
    }
  }
  ```

也许会有同学会提出，使用`Object.create`方法同样也可以创建对象。没错，该方法的确可以用于创建一个对象，但为啥我并没有放到上面进行提及呢？

原因很简单，就是`Object.create`方法会破坏对象的原型对象，也即相当于继承于某个指定的对象，我们来看看

```javascript
function Person(name) {
  this.name = name
}
var person = new Person('Andraw')
var newPerson = Object.create(person)
console.log(newPerson.__proto__ === person) // true
```

好明显，变量 newPerson 的`__proto__`原型指针直接指向了变量 person。与上面两种不一样，上面两种中的变量 obj 的`__proto__`原型指针直接指向顶层的`Object.prototype`。



## 对比 Object.create(null) 与 {}

从上述的阐述，可以了解到，`Object.create`就是继承了某个指定的对象。

在`Object.create(null)`中，明显就是创建一个继承`null`的对象出来。而对于`{}`，则是顶层`Object`直接所创建的实例。

接下来就看看两者的结构有什么不一样

```javascript
console.log(Object.create(null));
// {}
// No properties

console.log({})
// {}
// __proto__:
// 		constructor: ƒ Object()
// 		hasOwnProperty: ƒ hasOwnProperty()
//		isPrototypeOf: ƒ isPrototypeOf()
//		propertyIsEnumerable: ƒ propertyIsEnumerable()
//		toLocaleString: ƒ toLocaleString()
//		toString: ƒ toString()
//		valueOf: ƒ valueOf()
//		__defineGetter__: ƒ __defineGetter__()
//		__defineSetter__: ƒ __defineSetter__()
//		__lookupGetter__: ƒ __lookupGetter__()
//		__lookupSetter__: ƒ __lookupSetter__()
//		get __proto__: ƒ __proto__()
//		set __proto__: ƒ __proto__()
```

从代码上可以看到，**`Object.create(null)`创建出来后就是一个没任何属性的纯对象，而`{}`则会拥有一个`__proto__`原型指针，用于指向`Object.prototype`，因此可以通过`__proto__`直接访问`Object.prototype`中的属性**。

在应用场景上，使用`Object.create(null)`可以创建一个无任何依赖的纯对象，以便对对象的维护以及处理。而`{}`由于拥有顶层`Object`，因此可以对其属性直接根据原型链来访问一些常用方法，如`toString`、`valueOf`等方法。







































