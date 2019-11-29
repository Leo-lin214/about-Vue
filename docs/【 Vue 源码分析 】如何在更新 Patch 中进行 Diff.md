## 何为 Patch?

相信看过[【 Vue 源码分析 】渲染 Render (AST -> VNode)](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%B8%B2%E6%9F%93%20Render%20(AST%20-%3E%20VNode).md) 的童鞋们都还有点印象，就是我在最后执行渲染方法后，留下一个方法没有讲到，就是`_update`方法。忘记的童鞋也没关系，我简单滴截取一下代码过来，如下：

```javascript
updateComponent = function () { // 模板依赖Watcher更新回调方法
  // ...
  var vnode = vm._render() // 最主要的！！！直接转化AST为VNode节点
  // ...
  vm._update(vnode, hydrating) // 将转换好的VNode放到更新函数中使用patch进行比对
  // ...
}
```

通过执行`_c`渲染方法将相应的`AST`转化为`VNode`节点后，再将生成的`VNode`树作为参数传进`_update`方法中进行相应的`patch`。

好了，问题来了，那么究竟什么是`patch`呢？

所谓`patch`，按字面就是补丁，而在`Vue`中则是比较的意思。怎么理解呢？

**当响应式数据更改时，都会触发模板的`updateComponent`方法，生成新的`VNode`树，由于旧的`VNode`树会直接映射到真实`DOM`中的每一个节点，因此通过对新旧`VNode`树进行`patch`比较后，得出两个`VNode`树最小的差异，再将这些差异渲染到视图上**。

文字看的不是很清楚？🤔 甭急，下面就列出图来就会好理解了。

![vnode-patch](https://raw.githubusercontent.com/Andraw-lin/about-Vue/master/images/vnode-patch.jpg)



有上图在，是否会豁然开朗点哈哈。但是从图中，我们又发现了另外一个知识点，`Diff`出`VNode`树间的差异，那么`Diff`又是啥？

其实`Diff`就是一种实现`patch`的比较算法，可以很高效地处理两个新旧`VNode`树间的差异。接下来我们就来介绍一下这个算法。🤔



## Diff 算法

话不多说，直奔主题。

`Diff`算法的核心就是**针对具有相同父节点的同层新旧子节点进行比较，而不是使用逐层搜索递归遍历的方式**。**时间复杂度为`O(n)`**。

如何理解？

说白点，就是**当新旧`VNode`树在同一层具有相同的`VNode`节点时，才会继续对其子节点进行比较**。一旦旧`VNode`树同层中的节点在新`VNode`树中不存在或者是多余的，都会在新的真实`DOM`中进行添加或者删除。下面就拿一副图进行解释。

![diff-process](https://raw.githubusercontent.com/Andraw-lin/about-Vue/master/images/diff-process.jpg)

从上面的示例图可以看到，`Diff`算法中只会对同一层的元素进行比较，并且必须拥有相同节点元素，才会对其子节点进行比较，其他多余的同层节点都会一律做删除或添加操作。

接下来，我们就从源码角度来看看这过程到底是如何发生的。🤔



## 从源码角度进行探究

我们依然是从`_update`方法入手，看看到底是如何操作的。

```javascript
Vue.prototype._update = function (vnode, hydrating) {
  var vm = this; // 缓存vue实例
  var prevEl = vm.$el; // 获取实例中真实DOM元素
  var prevVnode = vm._vnode; // 获取旧VNode树
  vm._vnode = vnode; // 将新VNode树保存到实例的_vnode上，便于下次更新获取旧VNode树
  
  if (!prevVnode) { // 判断是否有旧VNode树，并进行相应的处理
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */); // 最开始的一次，即第一次渲染时是没有旧VNode树，直接执行__patch__
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode); // 新VNode树与旧VNode树进行__patch__
  }
  // ...
}
```

每一次更新模板时，都会先将渲染好的新`VNode`树保存到实例的`_vnode`属性上，这样做的目的是为了下一次更新时，能获取到旧`VNode`树进行比较。

针对是否拥有旧的`VNode`树，使用`__patch__`方法执行相应逻辑，也即执行了`patch`过程。

```javascript
var inBrowser = typeof window !== 'undefined'; // 浏览器环境
Vue.prototype.__patch__ = inBrowser ? patch : noop; // 只有在浏览器环境才能进行patch

var patch = createPatchFunction({ nodeOps: nodeOps, modules: modules })
```

可以看到，**只有在浏览器的环境下才能进行`patch`过程**，而实现`patch`的，就是`createPatchFunction`方法，我们接着看下去。

```javascript
function createPatchFunction (backend) {
  // ...
  // 省略了很多私有工具方法，下面会拿出一些进行说明
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(oldVnode)) { // 当旧VNode树不存在时，则直接创建一个根元素
      // empty mount (likely as component), create new root element
      isInitialPatch = true;
      createElm(vnode, insertedVnodeQueue); // 直接根据新VNode树并生成真实DOM
    } else { // 当存在旧VNode树时，则进行相应的比较
      // ...
      if (sameVnode(oldVnode, vnode)) { // 当新旧节点间是否相同时
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly); // 当新旧节点相同时则进行patch比较
      } else { // 当新旧节点间不相同时
        var oldElm = oldVnode.elm; // 获取旧节点元素
        var parentElm = nodeOps.parentNode(oldElm); // 获取旧节点的父节点
        
        // create new node
        createElm( // 由于新旧节点是不同的，因此会根据新节点创建一个新的节点
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        );
        
        // destroy old node
        if (isDef(parentElm)) { // 当创建好新节点后，也会把旧节点进行删除
          removeVnodes(parentElm, [oldVnode], 0, 0);
        } else if (isDef(oldVnode.tag)) { // 删除响应节点后，也会调用相应的回调
          invokeDestroyHook(oldVnode);
        }
      }
    }
  }
}
```

好啦，对于`patch`比较过程，你也应该有了一个大概了解。现在就来简单总结一下上述代码。

- 当旧`VNode`树不存在时，直接根据新`VNode`树创建相应的真实`DOM`。
- 当旧`VNode`树存在时，则会调用`sameVnode`方法比较当前新旧节点是否相同。
  - 当新旧节点是相同时，会调用`patchVnode`方法比较新旧节点（过程就是继续比较其子节点，递归下去～）。
  - 当新旧节点是不同时，则会先按照新`VNode`节点创建新的真实`DOM`节点，再根据旧`VNode`节点将相应的真实`DOM`节点进行删除。

是不是很简单 🤔...那么问题来了，不是说`patch`过程是使用`Diff`算法进行比较的吗？怎么还看不到，甭急，下面我会讲到哈。

在上面的总结中，我们是可以看到两个方法，分别是`sameVnode`方法和`patchVnode`方法。接下来我们就来探讨一下这两个方法。

```javascript
function sameVnode (a, b) { // 判断两个节点间是否相同
  return (
    a.key === b.key && ( // 两个节点间相同，首先是唯一标识key必须相同
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b) // 接着就是节点标签名、是否为注释、数据是否为空、input类型都必须相同
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}

function sameInputType (a, b) { // 比较两个节点的input类型是否相同
  if (a.tag !== 'input') { return true }
  var i;
  var typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type;
  var typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type;
  return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}
```

比较两个新旧节点间是很简单的，主要是按照下面几个属性进行判断。

- `VNode`节点唯一标识`key`。
- 是否同为注释`isComment`。
- 数据属性是否为空`isDef`。
- 是否为相同的`input`类型`sameInputType`。

此时，不禁又有问题了，判断两个节点相同，只是仅仅判断其数据属性是否为空就可以了吗？？

其实，这样做不是没有目的的，**首先两个相同的节点，若仅仅只是`data`值不一样时，这样就没必要新创建一个新的节点，并把新的`data`赋值到节点。而是在原有节点上，仅仅将其`data`进行更改，这样一来就可以对性能得到有效的提升**。

好啦，接着就到我们的主角`patchVnode`方法了，这个才是`Diff`相关方法，我们先来看看源码是如何实现的。🤔

```javascript
function patchVnode (
 oldVnode,
 vnode,
 insertedVnodeQueue,
 ownerArray,
 index,
 removeOnly
) {
   if (oldVnode === vnode) { // 当发现两个节点是完全一模一样时，则直接返回
     return
   }
   // ...
   var elm = vnode.elm = oldVnode.elm;
	 // ...
   var i;
   var data = vnode.data;
   if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
     i(oldVnode, vnode); // 根据新VNode更新旧VNode的选项配置、数据属性、propsData等
   }
   
   var oldCh = oldVnode.children; // 获取旧节点的子节点集合
   var ch = vnode.children; // 获取新节点的子节点集合
   // ...
   if (isUndef(vnode.text)) { // 当新VNode节点不是文本节点时
     if (isDef(oldCh) && isDef(ch)) { // 当旧VNode子节点和新VNode子节点都不为空时
       if (oldCh !== ch) { updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly); } // 当旧VNode节点子节点和新VNode节点子节点不等时，再递归执行updateChildren比较子节点
     } else if (isDef(ch)) { // 当只有新VNode子节点存在而旧VNode子节点不存在时
       // ...
       if (isDef(oldVnode.text)) { nodeOps.setTextContent(elm, ''); } // 当旧VNode节点是文本节点时，先置空文本
       addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue); // 根据位置对真实DOM添加新的节点
     } else if (isDef(oldCh)) { // 当只有旧VNode子节点存在而新VNode子节点不存在时
       removeVnodes(elm, oldCh, 0, oldCh.length - 1); // 直接移除所有多余节点
     } else if (isDef(oldVnode.text)) { // 当只有旧VNode子节点存在并且是文本节点时
       nodeOps.setTextContent(elm, ''); // 直接置空文本处理
     }
   } else if (oldVnode.text !== vnode.text) { // 当旧VNode文本节点不等于新VNode文本节点时
     nodeOps.setTextContent(elm, vnode.text); // 直接将旧VNode节点设置为新VNode节点文本内容
   }
   // ...
 }
```

`patchVnode`方法做的事情不多，最主要就是按照一下场景做了处理。

- 当新`VNode`节点不是文本节点时。
  - 当新`VNode`节点和旧`VNode`节点都存在时，若两个节点不等，直接执行`updateChildren`递归执行其子节点进行比较。
  - 当只有新`VNode`节点存在时，若旧`VNode`节点是文本节点，先置空文本内容，再直接在真实`DOM`中相应位置新增新`VNode`节点。
  - 当只有旧`VNode`节点存在时，直接移除真实`DOM`中相应位置的多余旧`VNode`节点。
  - 当只有旧`VNode`节点存在并且它是文本节点时，直接置空文本内容即可。
- 当新`VNode`节点是文本节点时。若两个文本节点内容不一致，直接在真实`DOM`中对应旧`VNode`节点位置文本内容设置为新`VNode`节点文本内容即可。

接下来才是最重点呀。。😅 在上面中留下了`updateChildren`方法，那么这个方法又是干啥？

不瞒你说，**`updateChildren`方法在根据场景`Diff`后，将旧`VNode`树作出相应的改动**。在没有看源码之前，我会先阐述一下。

**`Diff`算法过程中，在将旧`VNode`树改动时，优先考虑相同位置的相同节点，再考虑需要移动的相同节点，最后才考虑创建或删除节点**。

有了上面的简单理解，我们就来继续探究啦 😄。

```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  var oldStartIdx = 0; // 旧节点开始位置
  var newStartIdx = 0; // 新节点开始位置
  var oldEndIdx = oldCh.length - 1; // 旧节点结束位置
  var oldStartVnode = oldCh[0]; // 旧节点第一个元素
  var oldEndVnode = oldCh[oldEndIdx]; // 旧节点最后一个元素
  var newEndIdx = newCh.length - 1; // 新节点结束位置
  var newStartVnode = newCh[0]; // 新节点第一个元素
  var newEndVnode = newCh[newEndIdx]; // 新节点最后一个元素
  var oldKeyToIdx, idxInOld, vnodeToMove, refElm;
  // ...
  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) { // 同时从新旧子节点集合开始遍历
    if (isUndef(oldStartVnode)) { // 从第一项开始，一直遍历旧节点初始元素直到不为空为止
      oldStartVnode = oldCh[++oldStartIdx]; // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) { // 从最后一项开始，一直遍历旧节点直到不为空为止
      oldEndVnode = oldCh[--oldEndIdx];
    } else if (sameVnode(oldStartVnode, newStartVnode)) { // （相同位置场景）当第一项旧节点和第一项新节点相同时，则继续执行patchVnode递归执行下去
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];
    } else if (sameVnode(oldEndVnode, newEndVnode)) { // （相同位置场景）当最后一项旧节点和最后一项新节点相同时，则继续执行patchVnode递归执行下去
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // （需要移动场景）当第一项旧节点和最后一项新节点相同时，先执行patchVnode递归执行下去，再执行insertBefore将真实DOM节点插入到相应位置
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm));
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // （需要移动场景）当最后一项旧节点和第一项新节点相同时，先执行patchVnode递归执行下去，再执行insertBefore将真实DOM节点插入到相应位置
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else { // 比较头尾都无相同元素时，直接判断新节点是否在旧节点结合中，若有则直接移动相应的位置，若无则直接新建一个节点
      if (isUndef(oldKeyToIdx)) { oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); } // 将旧节点结合创建一个哈希表
      idxInOld = isDef(newStartVnode.key)
            ? oldKeyToIdx[newStartVnode.key]
            : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx); // 根据哈希表，判断新节点是否在哈希表中，并获得对应旧节点的索引位置
      if (isUndef(idxInOld)) { // 当新节点不在旧节点集合中时，新建一个真实DOM节点
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
      } else { // 当新节点在旧节点集合中时，则会先判断两个节点是否相同
        vnodeToMove = oldCh[idxInOld]; // 根据索引位置获得旧节点
        if (sameVnode(vnodeToMove, newStartVnode)) { // 当两个节点是相同时，继续执行patchVnode递归执行下去，再执行insertBefore将真实DOM节点插入到相应位置
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
          oldCh[idxInOld] = undefined;
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
        } else { // 当两个节点不同时，直接新建一个新的DOM节点
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
        }
      }
      newStartVnode = newCh[++newStartIdx]
    }
  }
  if (oldStartIdx > oldEndIdx) { // 跳出循环后，若新节点依旧存在，那么就要遍历剩余的新节点并逐个新增到真实DOM中
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
  } else if (newStartIdx > newEndIdx) { // 跳出循环后，若旧节点依旧存在，那么就要将真实DOM中对应旧VNode节点进行删除操作
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}
```

看完上面一坨代码，是否会觉得晕晕的？既然晕，那我们就一步一步地来总结下，总会找到突破点的。🤔

1. 执行方法时，首先会先获取新旧`VNode`子节点集合的初始位置、结束位置、第一项元素以及最后一项元素。

2. 接着使用`sameVnode`方法判断新旧子节点集合头头、尾尾节点是否相同。分两种情况：

   - 当旧子节点集合第一项元素和新子节点集合第一项元素相同时，执行`patchVnode`方法递归遍历它们子节点集合下去。
   - 当旧子节点集合最后一项元素和新子节点集合最后一项元素相同时，执行`patchVnode`方法递归遍历它们子节点集合下去。

   这两种情况均属于**原地将元素更改即可，无需移动元素**，消耗性能最小。

3. 再使用`sameVnode`方法判断新旧子节点集合头尾、尾头节点是否相同。分为两种情况：

   - 当旧子节点集合第一项元素和新子节点集合最后一项元素相同时，执行`patchVnode`方法递归遍历它们子节点集合下去，然后执行`insertBefore`方法将真实`DOM`对应旧`VNode`节点的位置移到最后一位。（看图好理解 😂）

     ![](https://raw.githubusercontent.com/Andraw-lin/about-Vue/master/images/old-one-and-new-length.jpg)

     

   - 当旧子节点集合最后一项元素和新子节点集合第一项元素相同时，执行`patchVnode`方法递归遍历它们子节点集合下去，然后执行`insertBefore`方法将真实`DOM`对应旧`VNode`节点的位置移到第一位。（看图好理解 😂）

     ![](https://raw.githubusercontent.com/Andraw-lin/about-Vue/master/images/old-length-and-new-one.jpg)

   这两种可均属于**移动元素更改，需移动元素**，消耗性能一般。

4. 当头尾节点都比较完毕后，使用`createKeyToOldIdx`方法将旧子节点集合转化一个哈希表形式，然后获取新节点在旧子节点哈希表中位置 t。也分为两种情况：

   - 当位置 t 不存在时，那么直接使用`createElm`方法在真实`DOM`中创建节点。
   - 当位置 t 存在时。分为两种情况：
     + 若同一个位置的新旧两个节点是相同节点，执行`patchVnode`方法递归遍历它们子节点集合下去，然后执行`insertBefore`方法将真实`DOM`对应旧`VNode`节点移到位置 t 。
     + 若同一个位置的新旧两个节点是不相同节点，执行`createElm`方法在真实`DOM`中位置 t 创建节点。

5. 跳出循环后，剩余两种情况处理：

   - 当新子节点集合中还有剩余节点时，那么遍历其剩余节点，使用`createElm`方法逐个创建真实`DOM`节点。
   - 当旧子节点集合中还有剩余节点时，那么遍历其剩余节点，使用`removeNode`方法逐个在真实`DOM`中进行删除。

看完上面的总结后，有木有一种豁然开朗的感觉 😄。

或许你会对上面又会产生另外的疑惑，为什么最后使用哈希表来查询？不瞒你说，使用哈希表目的，在于能更快地根据传递值找出相应的位置是否拥有值。相关知识童鞋们可以自行查找。在这里我就不在陈述了～

终于要写完啦，😂 哈哈。敲的我手都快软了。可能你会觉得还有些知识点并没有提到，的确，由于我的时间有一定限制，所以也么有把所有知识点都一一讲述，但我还是希望大家能够一起学习一起进步的。

最后的最后，还是很感谢童鞋们的观看 💪。也祝大家在即将踏入的2020年里身体健康、万事如意，最重要的还是升官发财哈 😆







 







































