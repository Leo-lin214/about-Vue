## 介绍

Vuejs 源码分析在网上的分享有很多，本人也有参考和学习过，依然有一种体会就是，轮子虽多，也不及自己去探究一遍。

本项目的创建旨在个人对 Vuejs 源码的探究和学习（由于本人能力有限，各位大佬看了若有问题的话，还是多提点一下小弟 :see_no_evil: ），当然也可以是作为个人的一个小笔记，然后拿来跟大家一起分享探讨，一起成长。

（由于本人时间有限，更新的速度会稍微慢一丢丢，所以分享都会慢慢滴待续～）

看本项目之前，我也希望你对 Vuejs 有一定的了解，若有疑问可以看一下官方文档 https://cn.vuejs.org/v2/guide/ 。


---


## 目录

- [【 Vue 源码分析 】数据初始化之响应式探究（上）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%95%B0%E6%8D%AE%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B9%8B%E5%93%8D%E5%BA%94%E5%BC%8F%E6%8E%A2%E7%A9%B6%EF%BC%88%E4%B8%8A%EF%BC%89.md)
- [【 Vue 源码分析 】数据初始化之响应式探究（下）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%95%B0%E6%8D%AE%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B9%8B%E5%93%8D%E5%BA%94%E5%BC%8F%E6%8E%A2%E7%A9%B6%EF%BC%88%E4%B8%8B%EF%BC%89.md)
- [【 Vue 源码分析 】数据初始化之依赖收集（上）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%95%B0%E6%8D%AE%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B9%8B%E4%BE%9D%E8%B5%96%E6%94%B6%E9%9B%86%EF%BC%88%E4%B8%8A%EF%BC%89.md)
- [【 Vue 源码分析 】数据初始化之依赖收集（下）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%95%B0%E6%8D%AE%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B9%8B%E4%BE%9D%E8%B5%96%E6%94%B6%E9%9B%86%EF%BC%88%E4%B8%8B%EF%BC%89.md)
- [【 Vue 源码分析 】数据初始化之依赖更新](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E6%95%B0%E6%8D%AE%E5%88%9D%E5%A7%8B%E5%8C%96%E4%B9%8B%E4%BE%9D%E8%B5%96%E6%9B%B4%E6%96%B0.md)
- [【 Vue 源码分析 】为什么不推荐使用 $forceUpdate](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E6%8E%A8%E8%8D%90%E4%BD%BF%E7%94%A8%20%24forceUpdate.md)

- [【 Vue 源码分析 】计算属性 Computed](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E8%AE%A1%E7%AE%97%E5%B1%9E%E6%80%A7%20Computed.md)

- [【 Vue 源码分析 】侦听器 Watch](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E4%BE%A6%E5%90%AC%E5%99%A8%20Watch.md)

- [【 Vue 源码分析 】方法 Methods](https://github.com/Andraw-lin/about-Vue/blob/master/docs/【%20Vue%20源码分析%20】方法%20Methods.md)

- [【 Vue 源码分析 】运行机制之 Props](https://github.com/Andraw-lin/about-Vue/blob/master/docs/%E3%80%90%20Vue%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%20%E3%80%91%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%E4%B9%8B%20Props.md)

- [【 Vue 源码分析 】混合 Mixin（上）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/【%20Vue%20源码分析%20】混合%20Mixin(上).md)

- [【 Vue 源码分析 】混合 Mixin（下）](https://github.com/Andraw-lin/about-Vue/blob/master/docs/【%20Vue%20源码分析%20】混合%20Mixin(下).md)

- 待续～～

## 其他话题

- [Object.create(null) 与 {} 究竟有什么不一样](https://github.com/Andraw-lin/about-Vue/blob/master/docs/Object.create(null)%20%E4%B8%8E%20%7B%7D%20%E7%A9%B6%E7%AB%9F%E6%9C%89%E4%BB%80%E4%B9%88%E4%B8%8D%E4%B8%80%E6%A0%B7.md)

---


## 对本项目的疑惑

若各位大佬在看的同时，对本项目有任何想法或者意见，欢迎各位多提点 issue，因为我也想和各位大佬们一起学习和探讨。

另外，若对项目中文章提到的知识点有疑问，我是希望你可以看看 `javascript高级程序设计` 和 `设计模式` 的。因为源码里对 javascript 的基础比较看重，还有就是涉及到的设计模式会很多，所以在探讨过程中还是要慢慢领会和斟酌～

