*今天接着上一篇内容，主要来对编译的三大阶段 parse、optimise、generate 进行一个简单讲解，由于每一个阶段都涉及到很长的代码，所以我都会简单引入一些代码来说说，至少是知道每一个阶段是如何实现的，至于细微的细节方面，后面有时间的话我也会进行讲解。 🙈*



## 语法解析 Parse

想到语法解析，就不得不提起 AST。那到底什么是 AST 呢？

AST ( abstract syntax tree )是抽象语法树，用来描述树形结构的。 其实在前面讲解从 template 到 DOM 时也提到过，现在再次拿来举个例子，这样才会比较直白点。

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

上述使用 VNode 表示了 AST，每一个子项都会塞进 children 属性中，一层嵌套一层，最终将以一棵树形式表示了真实的 DOM 结构。

说了那么多，是时候回到源码层面了，看看源码中是如何实现。

```javascript
/**
  * Convert HTML string to AST.
  */
function parse (
 template,
 options
) {
   // ...
   var stack = []  // 使用栈数据结构保存每个DOM节点的AST
   // ...
   var root // template中根元素
   var currentParent // 当前节点的父元素节点
   // ...
   parseHTML(template, { // 正式开始对模板进行解析执行
     // ...
     
   })
 }
```

对于`parseHTML`方法，内部其实就是使用了**正则表达式对模板进行了解析匹配**。现在我们就来看看`parseHTML`方法是如何实现的额（当然我也会稍微缩略了...）

```javascript
var isPlainTextElement = makeMap('script,style,textarea', true)

function parseHTML (html, options) {
  var stack = [] // 用来暂时性地保存父子节点间关系
  // ...
  var last, lastTag // 分别用来表示当前模板，是否是最后一个标签
  while(html) {
    last = html // 获取当前剩余模板
    if (!lastTag || !isPlainTextElement(lastTag)) { // 判断是最后一个标签，或最后一个标签是为特殊标签名
      var textEnd = html.indexof('<') // 获取<标志符在模板的索引号
      if (textEnd === 0) { // 当索引号为0时，肯定是一个首标签符
        if (comment.test(html)) { // 首标签符是否是注释
          // ...
          continue
        }
        var doctypeMatch = html.match(doctype) // 首标签符是否为doctype
        if (doctypeMatch) {
          // ...
          continue
        }
        var endTagMatch = html.match(endTag) // 首标签符是否为最后一个尾标签
        if (endTagMatch) {
          // ...
          parseEndTag(endTagMatch[1], curIndex, index) // 处理尾标签符
          continue
        }
        var startTagMatch = parseStartTag() // 前面都不通过后，直接作为一个首标签符进行处理
        if (startTagMatch) {
          handleStartTag(startTagMatch) // 正式处理首标签符
					continue
        }
      }
      var text // 模板中的文本
      if (textEnd >= 0) { // 当索引号大于或等于0时，在索引为0到符号为<为止，都是作为一个文本进行处理
        // ...
        text = html.substring(0, textEnd) // 截取获得文本
      }
      if (textEnd < 0) { // 当索引号为小于0时，那么肯定不再存在标签，直接作为文本进行处理
        text = html
      }
      if (options.chars && text) { // 获取相对应的文本后，就进行文本处理，如文本中有表达式或静态文案等
        options.chars(text, index - text.length, index)
      }
    } else { // 最后一个标签或者是特殊标签名的处理
      // ...
      parseEndTag(stackedTag, index - endTagLength, index)
    }
  }
}
```

看完上述的代码，其实可以很简单滴来总结一下`parseHTML`的流程主要是做了哪些工作。

- 遍历模板内容，先匹配标志符`<`索引号。若匹配成功，则作为首标签处理，判断是否是注释、doctype等特殊处理，然后**再判断若为尾标签则直接作为尾标签处理，若不为尾标签则直接作为首标签处理**。
- 若匹配到标志符`<`索引号是大于0，那么在索引号为0到标志符`<`索引号为止，都是作为一个文本进行处理，因此会对模板进行截取。若匹配到标志符`<`索引号是小于0，那么整个模板都是一个文本，直接作为文本处理。
- 当前面两个步骤处理好的`lastTag`是尾标签后，直接作为一个尾标签使用`parseEndTag`进行处理。

那么问题来了，看了半天上述代码，我都找不出任何有关 AST 方面的东西啊？

其实在处理首标签符方法`handleStartTag`中，就是会创建一个 AST，我们来看看 😄。

```javascript
function handleStartTag (match) {
  // ...
  // 首标签中属性、指令等处理
  if (options.start) { // 该选项上面我并没有指出，其实就是真正生成ast方法
    options.start(tagName, attrs, unary, match.start, match.end);
  }
}

parseHTML(template, {
  // ...
  start: function start (tag, attrs, unary, start$1, end) {
    // ...
    var element = createASTElement(tag, attrs, currentParent) // 根据匹配到的首标签生成相应的ast
    // ...
  }
})

function createASTElement ( // 生成ast
 tag,
 attrs,
 parent
) {
  return {
    type: 1,
    tag: tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    rawAttrsMap: {},
    parent: parent,
    children: []
  }
}
```

是吧（虽然我省略了很多，但还是知道有那么回事好点），在匹配到首标签时，就会生成一个ast，并对后续的尾标签以及子元素做相应的处理。



## 优化编译 Optimise

上述简单介绍了`parse`阶段，那么到了`optimise`阶段，又是做点什么工作的？当然你看英文意思都很清楚，肯定是优化鸭 🤔。那么问题来了，既然是优化，那么是怎么优化呢？

我们就先来看看源码是如何实现优化编译的。

```javascript
function optimize (root, options) { // 优化编译开始
  if (!root) { return } // 必须是直接拿编译好的ast进行优化功能
  // ...
  // first pass: mark all non-static nodes.
  markStatic$1(root); // 标记所有非静态节点
  // second pass: mark static roots.
  markStaticRoots(root, false); // 标记所有静态树
}
```

代码很简单，无非就是标记非静态节点或静态节点，那么这样做有啥用呢？其实这个函数我没有标出官方注释，在注释中说明的很清楚，现在就来展示一下官方注释是如何解释的。

> Goal of the optimizer: walk the generated template AST tree and detect sub-trees that are purely static, i.e. parts of the DOM that never needs to change.
>
> 优化器的目标：遍历生成的模板AST树，并检测纯静态的子树，即永远不需要更改的DOM。
>
> 1. Hoist them into constants, so that we no longer need to create fresh nodes for them on each re-render;
>
> 将它们提升为常量，这样我们就不再需要在每次重新渲染时为他们创建新的节点；
>
> 2. Completely skip them in the patching process.
>
> 在更改过程中完全跳过它们进行编译。

看了官方解释，是否眼前一亮，无非就是对静态节点范围的标记，然后在下次编译时就会跳过并且在渲染过程中也不会再次渲染该节点，因为**静态节点都是不可变得**。

接下来我们就来看看它们是如何分别标记静态节点以及非静态节点的。

```javascript
function isStatic (node) { // 判断是否为非静态节点
  if (node.type === 2) { // expression（表达式）
    return false
  }
  if (node.type === 3) { // text（文本）
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings（非动态绑定）
    !node.if && !node.for && // not v-if or v-for or v-else（无指令）
    !isBuiltInTag(node.tag) && // not a built-in（节点不在建立过程中）
    isPlatformReservedTag(node.tag) && // not a component（不是一个组件）
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}
function markStatic$1 (node) { // 标记非静态节点处理
  node.static = isStatic(node) // 判断当前节点是否为非静态节点
  if (node.type === 1) { // 作为元素节点处理
    // ...
    for (var i = 0, l = node.children.length; i < l; i++) { // 遍历字节点
      var child = node.children[i];
      markStatic$1(child); // 递归遍历标记下去
      if (!child.static) {
        node.static = false;
      }
    }
  }
}
function markStaticRoots (node, isInFor) { // 标记静态树处理
  if (node.type === 1) {
    // ...
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) { // 要使节点具有静态根功能，那么必须拥有字节点，并且字节点只能是文本
      node.staticRoot = true; // 直接标记为静态树
      return
    } else { // 否则则不是静态树
      node.staticRoot = false
    }
    if (node.children) {
      for (var i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for); // 继续递归处理
      }
    }
    // ...
  }
}
```

看完上述代码，我最直观的感受是，**使用递归形式实现了静态树和非静态节点标记功能**。一旦是静态树，那么就直接标记为静态树，并且在下次编译时就会直接跳过以及在渲染时不会再次重新创建。

另外需要说明的一点是，`node.type`是代表了不同类型。

- 1：表示元素节点。
- 2：表示表达式。
- 3：表示文本。

总结一下：**优化编译`Optimise`功能在于标记非静态节点以及静态树**。



##生成渲染方法 Generate

终于到了生成渲染方法`generate`，相信你们对他功能也有所了解，就是经过优化编译后的 AST，直接转换成渲染方法。那么它究竟又是如何实现的，我们接着看下去。

```javascript
function generate ( // 生成渲染方法
 ast,
 options
) {
  var state = new CodegenState(options); // 初始化配置信息
  var code = ast ? genElement(ast, state) : '_c("div")'; // 根据ast以及配置信息生成渲染方法
  return {
    render: ("with(this){return " + code + "}"), // 渲染方法字符串
    staticRenderFns: state.staticRenderFns // 静态树
  }
}
```

我们就来看看每一个过程大概是怎样的。

```javascript
var CodegenState = function CodegenState (options) {
  this.options = options; // 缓存一份选项信息
  this.warn = options.warn || baseWarn; // 获取选项中警告信息
  this.transforms = pluckModuleFunction(options.modules, 'transformCode');
  this.dataGenFns = pluckModuleFunction(options.modules, 'genData');
  this.directives = extend(extend({}, baseDirectives), options.directives); // 获取选项中指令信息
  var isReservedTag = options.isReservedTag || no;
  this.maybeComponent = function (el) { return !!el.component || !isReservedTag(el.tag); }; // 是否为组件
  this.onceId = 0;
  this.staticRenderFns = []; // 用于保存静态树
  this.pre = false;
}
```

可以看到，`CodegenState`方法主要是针对选项进行配置一些简单信息，其中最为重要的就是使用`staticRenderFns`数组来保存静态树。

接下来，就是`genELement`方法，这个方法很重要，主要是通过 AST 转换成相应的渲染方法。

```javascript
function genElement (el, state) {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre;
  }

  if (el.staticRoot && !el.staticProcessed) { // 判断是否静态树
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) { // 判断是否为v-once指令节点
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) { // 判断是否为v-for指令节点
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) { // 判断是否为v-if指令节点
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) { // 判断是否为模板元素，若是则继续获取子节点
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') { // 判断是否为v-slot指令节点
    return genSlot(el, state)
  } else { // 若以上都不通过，则都作为元素或组件节点来处理
    // component or element
    var code;
    if (el.component) { // 组件节点处理
      code = genComponent(el.component, el, state);
    } else { // 元素节点处理
      var data;
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData$2(el, state); // 获取元素中相应的数据
      }

      var children = el.inlineTemplate ? null : genChildren(el, state, true); // 继续获取子节点
      code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")"; // 最终得到渲染方法字符串
    }
    // module transforms
    // ...
    return code
  }
}
```

可以看到**`genElement`方法就是针对 AST，处理一些静态树、特殊指令节点、组件节点以及元素节点并生成相应的渲染方法字符串**。

由于每一项代码都很长，我就暂时不放出来进行讨论，后面有机会的话我会再回来跟你们一起探讨哈 😂。

现在就拿`genStatic`方法作为栗子举个栗子看看其源码实现。

```javascript
function genStatic (el, state) { // 生成静态树的渲染方法
  // ...
  state.staticRenderFns.push(("with(this){return " + (genElement(el, state)) + "}")); // 将静态树渲染方法指定作用域直接Push进staticRenderFns中进行保存
  state.pre = originalPreState;
  return ("_m(" + (state.staticRenderFns.length - 1) + (el.staticInFor ? ',true' : '') + ")") // 返回使用渲染方法包裹的静态树
}
```

`genStatic`方法工作只有两个，分别是

- 使用`_m`渲染方法包裹该作用域，组装成节点最终的渲染方法。

- 将静态树使用`with`操作符将渲染方法绑定相应作用域，并保存到`staticRenderFns`属性中。

注：`with`作用域在[【 Vue 源码分析 】运行机制之 Props](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%E4%B9%8B%20Props.md) 中提及过，忘记的童鞋们有兴趣还是可以返回去看看哈。

对于上述过程，也许你会有疑惑了，究竟`_m`、`_c`是什么东西，又是能干嘛的？别急，在下一章 Render 中我会慢慢提到的。

看到这里，其实就可以明白，**在`generate`方法中最终处理的其实就是生成渲染方法以及将绑定好作用的静态树渲染方法保存起来**。



🙈 至此，编译阶段基本讲解完啦，可能有些地方讲的还是比较简单，望见谅哈。现在回头来总结一下，`Compile`编译阶段到底做了哪些事情。

- 一个完整编译阶段，必须经过语法解析`parse`、优化编译`optimise`、生成渲染方法`generate`三大阶段。
- 语法解析`parse`主要负责**将`template`模板内容转换成相应的`AST`抽象语法树**。
- 优化编译`optimise`主要负责**遍历转换好的`AST`抽象语法树，分别标记非静态节点和静态树**。
- 生成渲染方法`generate`主要负责**对优化好的`AST`抽象语法树生成相应的渲染方法，并将绑定好作用域的静态树渲染方法保存起来**。