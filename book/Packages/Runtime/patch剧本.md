**patch** 在vue中，充当一个渲染角色存在。

**patch**就像一家工厂，把**vnode**（一个描述**vue组件**，或者描述**NODE**的对象）当作生产原料，进行一系列的加工，最终生产出成品（也就是真实的**NODE**）。

一家工厂，可以是由**流程控制器**和**加工机器**组成的。**流程控制器**根据原料**vnode**的类型，来选取机器处理。机器负责把传入来的东西，加工或输出。



对于**patch**这个工厂来说，有哪些流程控制器？

组件渲染流程控制器

普通元素渲染流程控制器



对于**patch**这个工厂来说，有哪些机器？

（property 和 attribute非常容易混淆，两个单词的中文翻译也都非常相近（property：属性，attribute：特性），但实际上，二者是不同的东西，属于不同的范畴。）

- property是DOM中的属性，是JavaScript里的对象；

- attribute是HTML标签上的特性，它的值只能够是字符串；

- property能够从attribute中得到同步；

- attribute不会同步property上的值；

- attribute和property之间的数据绑定是单向的，attribute->property；

- 更改property和attribute上的任意值，都会将更新反映到HTML页面中；

  

**patchAttr**: 使用setAttrbute方法，添加、去除和设置HTMLELEMENT标签上特性。

**patchClass**: 设置HTMLELEMENT的className属性，为什么有patchAttr存在了，还需要patchClass呢，因为速度比patchAttr要快。

**patchStyle**: 利用setProperty修改HTMLELEMENT的style属性。

**patchDomProp**: 处理HTMLELEMENT中的属性。

**patchEvent**: 更新、删除和添加事件（这个事件就是ducument.addEventListener(Event)中的Event）。

**processText**：创建、插入和更换**TextNode**

**processCommentNode**:  创建、插入和更换**CommentNode**  e.g. \<\!\-\- 这就是comment\-\-\>

**mountStaticNode**: 创建Node，把Node丢进目标Node中



q1: 抱歉，我又懵逼了，为什么拥有了patchAttr来设置属性了，还需要patchClass、patchStyle和patchDomProp?

R1: 确实不需要，这么做主要是为了速度，这里就需要了解property和attribute了



patchAttr



小知识：

可以利用双叹号 !! 来把一个值转换成布尔属性（比如0是false，null是false之类的，负数是true，undefinded是false，undefinded是false）。



https://developer.mozilla.org/zh-CN/docs/Web/API/Event/stopImmediatePropagation

[`Event`](https://developer.mozilla.org/zh-CN/docs/Web/API/Event) 接口的 `stopImmediatePropagation()` 方法阻止监听同一事件的其他事件监听器被调用。

