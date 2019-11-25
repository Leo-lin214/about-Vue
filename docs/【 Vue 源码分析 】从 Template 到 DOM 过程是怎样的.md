## Virtual DOM

虚拟 DOM 技术在前端领域中并不少见，无论是 React 还是 Vue 都已经选择拥抱了它。那么它究竟是什么东西？又有什么好处吸引着别人呢？接下来我就简单滴讲讲。🤔

首先，**虚拟 DOM 其实就是一个 Javascript 数据结构，它可以描述真实的 DOM Tree 结构**。

使用一个 Javascript 数据结构（如对象）实现虚拟 DOM，无疑实现了跨平台的能力，无需考虑兼容性等问题。当然，在更改数据时，更是将虚拟 DOM 与真实 DOM 进行比较，准确滴找出需要修改的地方，最后再一次性滴对真实 DOM 进行更改（侧面也反映了，多次更改只重渲真实 DOM 一次），有效地提高性能。

既然虚拟 DOM 好处多多，那么它的构成又是怎样的呢？在这里我会使用 Vue 中的 VNode 进行讲解。



## VNode

对于 VNode ，相信大家一点都不陌生，用于表示虚拟节点，是实现 Virtual DOM 的一种方式。那么它究竟是怎样的呢？我们就去 Vue 源码里探讨一下。

```javascript
var VNode = function VNode (
 tag,
 data,
 children,
 text,
 elm,
 context,
 componentOptions,
 asyncFactory
) {
  this.tag = tag; // 标签名
  this.data = data; // 关联数据
  this.children = children; // 所有的子元素
  this.text = text; // 静态文案
  this.elm = elm; // DOM 节点
  this.ns = undefined;
  this.context = context; // 执行环境，即执行上下文
  this.fnContext = undefined;
  this.fnOptions = undefined;
  this.fnScopeId = undefined;
  this.key = data && data.key; // 数据相关联标识
  this.componentOptions = componentOptions; // 组件选项
  this.componentInstance = undefined;
  this.parent = undefined; // 父元素
  this.raw = false;
  this.isStatic = false; // 是否为静态节点（即无任何表达式）
  this.isRootInsert = true;
  this.isComment = false; // 是否为注释
  this.isCloned = false;
  this.isOnce = false;
  this.asyncFactory = asyncFactory;
  this.asyncMeta = undefined;
  this.isAsyncPlaceholder = false;
};
```

可以看到，VNode 其实就是一个对象，包含节点中所有信息如标签名、关联数据、父元素以及子元素等等。想象一下，从根节点走一遍下来到最后一个节点，是否就可以构造一个完整的 Virtual DOM 了？😄

我们先来简单滴看一个栗子🌰。

```javascript
<span class="text">Hello World...</span>
// 使用 VNode 表示
{
  tag: 'span',
  isStatic: false,
  text: undefined,
  data: {
    staticClass: 'text'
  },
  children: [{
    tag: undefined,
    text: 'Hello World...'
  }]
}
```

既然目前对 VNode 有了大概的了解，那么当我们开发的时候，编写的 template 是如何变成真实的 DOM 的呢？

不妨我们从源码的角度来探讨一下大致流程。🤔



## 源码分析 template 到 DOM 流程

我们先从在初始化 Vue 实例时来进行一步一步探讨。

```javascript
function Vue (options) {
  // ...
  this._init(options)
}
Vue.prototype._init = function (options) {
  // ...
  if (vm.$options.el) { // 判断绑定元素选项是否存在
    vm.$mount(vm.$options.el); // 根据绑定元素选项开始挂载
  }
}
```

明显滴，会先判断是否绑定有元素，再根据绑定元素选项直接挂载。那么这个挂载方法又是如何的呢？

```javascript
Vue.prototype.$mount = function (
 el,
 hydrating
) {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating) // 挂载组件
}

var mount = Vue.prototype.$mount;
Vue.prototype.$mount = function ( // 挂载执行
 el,
 hydrating
) {
  el = el && query(el); // 获取元素

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) { // 若元素是body或document，则直接跳过
    warn(
      "Do not mount Vue to <html> or <body> - mount to normal elements instead."
    );
    return this
  }

  var options = this.$options; // 获取Vue实例的Options选项
  // resolve template/el and convert to render function
  if (!options.render) { // 判断选项中渲染函数是否存在
    var template = options.template; // 获取template模版
    if (template) {
      if (typeof template === 'string') { // 判断模板是否是字符串
        if (template.charAt(0) === '#') { // 判断模板是否以#开头
          template = idToTemplate(template); // 根据对应的id获取模板内容
          /* istanbul ignore if */
          if (!template) {
            warn(
              ("Template element not found or is empty: " + (options.template)),
              this
            );
          }
        }
      } else if (template.nodeType) { // 判断是否为一个节点类型，若是直接获取内在HTML
        template = template.innerHTML;
      } else {
        {
          warn('invalid template option:' + template, this);
        }
        return this
      }
    } else if (el) { // 获取元素外层父元素
      template = getOuterHTML(el);
    }
    if (template) { // 开始compile
      /* istanbul ignore if */
      if (config.performance && mark) {
        mark('compile');
      }

      var ref = compileToFunctions(template, { // 根据template获得相应的渲染函数（这是重点啊！！！）
        outputSourceRange: "development" !== 'production',
        shouldDecodeNewlines: shouldDecodeNewlines,
        shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render; // 把渲染相关函数赋予选项中
      options.staticRenderFns = staticRenderFns; // 把渲染相关函数赋予选项中

      /* istanbul ignore if */
      if (config.performance && mark) {
        mark('compile end');
        measure(("vue " + (this._name) + " compile"), 'compile', 'compile end');
      }
    }
  }
  return mount.call(this, el, hydrating) // 处理好渲染函数后，正式开始挂载
}
```

由上面的代码可以看到，从 template 到 DOM 过程，需要经过两个阶段，分别是：

- 通过`compileToFunctions`函数对 template 进行 compile，其中包括`parse`、`optimize`、`generate`三大阶段，进而获得渲染函数，接下去章节会分别介绍。
- 通过`mountComponent`函数执行渲染函数，最终得到真实 DOM 结构。



















































