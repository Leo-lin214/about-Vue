*在观看这篇时，还是建议童鞋们先睇睇上一篇的从 template 到 DOM 大概流程，因为对于接下来的章节都会有很大帮助的... 😂*

## Compile 主体流程

在上一个章节中，我曾留下来一个函数`compileToFunctions`，这个方法到底是用来干嘛？

相信看过上一个篇节的童鞋都会清楚，它就是用来将 template 模板内容进行语法**解析`parse`、优化`optimise`、生成渲染函数`generate`**，按三个步骤实现从模板 template 到渲染函数的。

既然是按三个步骤，我们接下来肯定会讲源码在这三个步骤中到底是如何实现的。在这之前，我还是先会给大家大概讲一下`compile`执行过程到底是如何的，因为里面会融入了多层函数嵌套，当然也少不了使用闭包形式。接下来就来看看啦 🤔

```javascript
var ref$1 = createCompiler(baseOptions);
var compile = ref$1.compile;
var compileToFunctions = ref$1.compileToFunctions
```

千辛万苦之下，终于找到了`compileToFunctions`的来源地。由代码可以看到，通过`createCompiler`方法解释一个全局配置后得到了`compileToFuntions`，接下来就看看`createCompiler`到底干了什么东西。

```javascript
var createCompiler = createCompilerCreator(function baseCompile (
  template,
  options
) {
  var ast = parse(template.trim(), options); // 语法解析模板 template
  if (options.optimize !== false) { 
    optimize(ast, options); // 优化抽象语法树
  }
  var code = generate(ast, options); // 生成最终的渲染函数相关对象
  return {
    ast: ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

`createCompiler`方法逻辑不多，就是执行方法`createCompilerCreator`，并传递方法`baseCompile`作为参数。可以预料到，方法`createCompilerCreator`实质上会返回一个函数，而**方法`baseCompile`的作用也很明显，就是实实在在的`compile`功能，包含了parse、optimise、generate三大阶段，并最终返回一个渲染函数以及抽象语法树**。接下来继续探究方法`createCompilerCreator`。

```javascript
function createCompilerCreator (baseCompile) { // 将baseCompile作为了私有方法
  return function createCompiler (baseOptions) { // 使用闭包实现创建渲染方法
    function compile ( // 编译方法，返回编译后的信息如渲染方法、错误信息等
     template,
     options
    ) {
      var finalOptions = Object.create(baseOptions); // 使用继承方式，浅复制baseOptions全局基础配置
      var errors = []; // 用于存储错误
      var tips = [];

      var warn = function (msg, range, tip) {
        (tip ? tips : errors).push(msg);
      };

      if (options) {
        // ...
        // 已省略，选项信息处理
      }

      var compiled = baseCompile(template.trim(), finalOptions); // 正式开始对模板进行编译（包括parse、optimise、generate三个阶段）
      {
        detectErrors(compiled.ast, warn);
      }
      // 开始赋予处理后的错误以及提示
      compiled.errors = errors;
      compiled.tips = tips;
      return compiled
    }

    return {
      compile: compile, // 返回编译方法
      compileToFunctions: createCompileToFunctionFn(compile) // what？干么用的
    }
  }
}
```

可以看到，`createCompilerCreator`方法使用闭包实现了编译方法`compile`，将真正的`baseCompile`作为了私有方法进行处理。

`compile`方法只做两件事情，分别是

- 处理`options`选项信息，如错误、提示、指令等。
- 使用`baseCompile`方法直接编译模板（包含了parse、optimise、generate三个阶段），最终获得渲染方法，并返回。

到这里，也许你会觉得流程已经结束了，但是尤大大不甘心，总觉得每次编译时，不变化的地方是不是就不应该再重新编译？那么有没有一些好的方法可以处理这些场景呢？答案就是缓存。

不跟你兜圈，上述代码中留下来的`compileToFunctions`就是`comile`方法的加强版 😄。我们接着看他是如何实现的。

```javascript
function createFunction (code, errors) { // 将函数字符串转化为函数
  try {
    return new Function(code) // 根据传递过来函数字符串重新转化为函数
  } catch (err) {
    errors.push({ err: err, code: code });
    return noop
  }
}

function createCompileToFunctionFn (compile) {
  var cache = Object.create(null); // 用纯对象作为映射表，私有变量，作为缓存用的

  return function compileToFunctions ( // 闭包实现缓存功能的compile
   template,
   options,
   vm
  ) {
    // ...

    // check cache
    var key = options.delimiters
    ? String(options.delimiters) + template
    : template; // 缓存中直接是根据options.delimiters
    if (cache[key]) { // 判断映射表中是否有对应编译结果，若有直接返回，不再进行编译
      return cache[key]
    }

    // compile
    var compiled = compile(template, options); // 若缓存中无对应的编译结果，则开始编译

    // check compilation errors/tips（检查对比错误以及提示是否处理完毕）
  	// ...

    // turn code into functions（转化函数字符串为函数形式）
    var res = {};
    var fnGenErrors = [];
    res.render = createFunction(compiled.render, fnGenErrors);
    res.staticRenderFns = compiled.staticRenderFns.map(function (code) {
      return createFunction(code, fnGenErrors)
    });

   	// ...
    // 处理函数编译过程的错误

    return (cache[key] = res) // 缓存编译结果，并返回
  }
}
```

`compileToFunctions`方法的工作很简单，就是使用闭包形式缓存了编译结果，极大地提升编译性能。

在这里，我先不讨论`render`方法和`staticRenderFns`方法到底有何区别，因为这属于下一节的内容。🤔

看到这里，是不是会有点晕晕的哈哈，函数层层嵌套，**使用闭包实现了函数的私有性以及可缓存性**。虽然看起来复杂，但是实现的核心我们还是能好好使用的。