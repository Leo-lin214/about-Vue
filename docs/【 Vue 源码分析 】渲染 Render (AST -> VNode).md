## 渲染 Render 功能

在不了解`Render`实现情况下，很容易会认为它的功能就是直接将`AST`通过执行渲染方法从而得到真实 DOM（当初我也是这样认为的 😅）。

既然`Render`不具备上述的功能，那么它的功能又是什么呢？

看到标题，你应该也知道了答案，那就是**将`AST`转换为`VNode`节点**。那么转化为真实 DOM 又是发生在哪个步骤呢？不瞒你说，那就是**在最后一步`Diff`过程所得到两个`VNode`节点差异后，才会将差异渲染到真实环境中形成视图**。

接下来，我们就来从源码角度探究一下，`Render`是如何将`AST`转换为`VNode`的。😼



## 从源码角度进行分析

相信看过[【从 Template 到 DOM 过程是怎样的】](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E4%BB%8E%20Template%20%E5%88%B0%20DOM%20%E8%BF%87%E7%A8%8B%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84.md)的话，都应该比较清楚。再编译`compile`后，就会直接调用`$mount`方法，我们再来简单回顾一下。

```javascript
Vue.prototype.$mount = function (
 el,
 hydrating
) {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating) // 加载元素或组件方法
}

var mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
 el,
 hydrating
) {
  // ...
  var ref = compileToFunctions(template, { // 经过编译Compile后得到最终渲染方法相关对象
    outputSourceRange: "development" !== 'production',
    shouldDecodeNewlines: shouldDecodeNewlines,
    shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
    delimiters: options.delimiters,
    comments: options.comments
  }, this)
  var render = ref.render;
  var staticRenderFns = ref.staticRenderFns;
  options.render = render; // 将渲染方法挂载到实例选项中
  options.staticRenderFns = staticRenderFns; // 将收集到的静态树挂载到选项中
  // ...
  return mount.call(this, el, hydrating) // 正式开始加载元素
}
```

从流程上可以看到，通过编译`Compile`后，就会将相应的渲染方法以及收集到的静态树统一挂载到选项信息中，接着正式开始加载元素。

在加载元素里，最主要就是`mountComponent`方法，我们再来看看是如何加载的 🤔。

```javascript
var createEmptyVNode = function (text) { // 创建空VNode节点方法
  if ( text === void 0 ) text = '';

  var node = new VNode();
  node.text = text;
  node.isComment = true;
  return node
}

function mountComponent (
 vm,
 el,
 hydrating
) {
   vm.$el = el
   if (!vm.$options.render) { // 若不存在渲染方法，则作为一个创建空VNode节点来处理
     vm.$options.render = createEmptyVNode;
     // ...
   }
   callHook(vm, 'beforeMount') // 开始执行beforeMount生命周期回调
   var updateComponent // 转化AST为VNode节点外层方法
   // ...
   updateComponent = function () {
     // ...
     var vnode = vm._render() // 最主要的！！！直接转化AST为VNode节点
     // ...
     vm._update(vnode, hydrating) // 将转换好的VNode放到更新函数中使用patch进行比对
     // ...
   }
   new Watcher(vm, updateComponent, noop, { // 创建模板依赖Watcher，其中的更新函数即为updateComponent
     before: function before () {
       if (vm._isMounted && !vm._isDestroyed) {
         callHook(vm, 'beforeUpdate');
       }
     }
   }, true /* isRenderWatcher */)
 }
```

`mountComponent`方法会先判断实例选项是否有渲染方法，若无则直接赋值为空VNode节点。接着使用变量`updateComponent`作为暴露转化`AST`为`VNode`外层方法，并在创建模板依赖时将其作为更新回调函数，这样一来，在每次更新时，都会使用`updateComponent`方法执行转化`AST`为`VNode`节点。

在上述代码中，其中`_render`方法是转化`AST`为`VNode`核心方法，而另一个`_update`方法则是最后用于`patch`比对用到的（`_update`方法暂时不讲，会在下一章`Diff`说法中提及）。现在就来继续看看`_render`方法是如何将`AST`转化为`VNode`节点的。

```javascript
Vue.prototype._render = function () {
  var vm = this;
  var ref = vm.$options;
  var render = ref.render; // 获取实例选项中保存的渲染方法
  // ...
  var vnode; // VNode节点变量
  // ...
  vnode = render.call(vm._renderProxy, vm.$createElement) // 根据渲染方法执行相应的_c
  // ...
  return vnode
}
```

可以看到，直接调用了编译`Compile`出来的渲染方法。

首先，我们来回顾一下渲染方法是咋样的？看看下面这个栗子呀。

```javascript
<span>Hello Word...</span>
// 渲染方法如下
with(this) {
  _c('span', [_v('Hello Word...')])
}
```

终究回到了上一章中留下来的问题，究竟`_c`和`_v`是干嘛用的？🤔

其实它们都是尤大大在内部封装好的各种渲染方法，不妨我们就来看看还有哪些。

```javascript
function installRenderHelpers (target) { // 针对各种场景封装好的渲染方法
  target._o = markOnce; // 处理v-once指令的渲染方法
  target._n = toNumber; // 将输入值转化为数值，若转化失败则直接使用原始字符串
  target._s = toString; // 将输入值转化为字符串
  target._l = renderList; // 处理v-for指令的渲染方法
  target._t = renderSlot; // 处理v-slot指令的渲染方法
  target._q = looseEqual; // 判断两个值是否相等
  target._i = looseIndexOf; // 判断数组中是否存在与输入值相等的项
  target._m = renderStatic; // 处理静态树的渲染方法
  target._f = resolveFilter; // 处理选项信息中filters项
  target._k = checkKeyCodes; // 判断配置中是否拥有相应的eventKeyCode
  target._b = bindObjectProps; // 将v-bind指令绑定到相应的VNode
  target._v = createTextVNode; // 处理文本节点的渲染方法
  target._e = createEmptyVNode; // 创建空节点的渲染方法
  target._u = resolveScopedSlots; // 处理ScopedSlots
  target._g = bindObjectListeners; // 处理对象监听
  target._d = bindDynamicKeys; // 处理v-bind:key渲染方法
  target._p = prependModifier; // 判断类型是否为唯一字符串
}

vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false) // 创建元素VNode节点
```

上述代码，可以很仔细地展示了每一种场景的处理方案。由于时间有限，我就不会将每个场景的处理源码都提出来讲，后面有时间会回来再进行相应的讲解。

目前也大概知道每一个封装的渲染方法的含义，回到上面的主题，调用编译`Compile`出来的渲染方法，，其实就是调用`_c`创建元素`VNode`节点。

接下来我们就来看看`_c`内部是如何实现的。

```javascript
function createElement (
 context,
 tag,
 data,
 children,
 normalizationType,
 alwaysNormalize
) {
   if (Array.isArray(data) || isPrimitive(data)) { // 判断数据是否为数组或原始数据类型
     normalizationType = children;
     children = data;
     data = undefined;
   }
   // ...
   return _createElement(context, tag, data, children, normalizationType)
 }

function _createElement (
 context,
 tag,
 data,
 children,
 normalizationType
) {
   if (isDef(data) && isDef((data).__ob__)) { // 判断数据不为空并且无依赖时，则作为一个空VNode节点进行处理
     warn(
        "Avoid using observed data object as vnode data: " + (JSON.stringify(data)) + "\n" +
        'Always create fresh vnode data objects in each render!',
        context
      ); // 避免使用非响应式数据创建VNode节点，不然导致每次渲染都包含其中
      return createEmptyVNode() // 返回创建的空VNode节点
   }
   // ...
   var vnode
   if (typeof tag === 'string') {
     // ...
     if (config.isReservedTag(tag)) { // 判断标签名是否为HTML标签
       vnode = new VNode( // 创建元素VNode节点
         config.parsePlatformTagName(tag), data, children,
         undefined, undefined, context
       )
     } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) { // 判断是否为组件
       vnode = createComponent(Ctor, data, context, children, tag) // 创建组件VNode节点
     } else { // 其他一律当未知类型创建VNode节点
       vnode = new VNode(
         tag, data, children,
         undefined, undefined, context
       )
     }
   } else { // 若标签名不是字符串，则作为组件选项/构造函数创建组件
     vnode = createComponent(tag, data, context, children)
   }
   if (Array.isArray(vnode)) { // 判断生成的vnode是否为数组类型
     return vnode
   } else if (isDef(vnode)) { // 判断vnode是否为空
     if (isDef(ns)) { applyNS(vnode, ns); } // 定义当前节点的命名空间
     if (isDef(data)) { registerDeepBindings(data); } // 绑定动态的style、class
     return vnode
   } else { // 若为空，则直接返回一个空VNode节点
     return createEmptyVNode()
   }
 }
```

可以看到，在执行渲染方法`_c`时，最终都是返回了创建好的`VNode`节点。

代码上可能还有些方法没有细讲，如`createComponent`，其实底层都是经过一系列的处理后得到相应的`VNode`节点。另外，我个人认为创建好的`VNode`节点终究就是一个对象形式，不会存在数组形式（若有错误，希望指出哈）。

创建好的`VNode`节点最终经过绑定当前节点命名空间以及动态的`style`、`class`，最终将节点返回。

最后的最后，渲染`Render`最终实现的功能就是**将转化好的`AST`渲染方法，直接通过`VNode`构造函数构建相应的`VNode`节点**。在这个过程中，会建立一个模板依赖 Watcher，并且将转化过程作为一个更新回调方法保存，**每当响应式数据更新时，都会触发该过程，先渲染出相应的`VNode`节点，再进行`patch`比对，最后将比对的差异直接渲染成相应的真实`DOM`**。



























