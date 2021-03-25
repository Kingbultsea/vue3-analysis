### vue在触发dom点击事件的时候，需要验证event.timeStamp
##### 历史来源：

vue-2.4.2 #6566

##### 操作：

![image-20210325214307246](https://res.psy-1.com/FsAmfYi3QDyxmrfznLfst4ix1heZ)

点击**element-1**，触发了**countA++**和**countB++**。

##### 过程：

点击element-1，触发element-1本身的click事件，```expand = 1; countA++```。

由```expand```变动，把组件effect事件丢进微任务。

触发微任务，由于diff算法的原因，block-1会被复用，```patchProps```更新block-1的click事件为block-2的click事件。

由element-1所触发的冒泡事件，引起block-1中的click触发（click事件已经被更新成block-2的了）。

##### 解决方法：

在创建事件的时候，新增一个时间戳，该时间戳为**页面启动时间->创建该事件的时间**。

当触发click事件的时候，判断Event的timeStamp是否大于事件的创建时间。

![image-20210325220037306](https://res.psy-1.com/FgdwrD1jKy9ZyaK3KcFVXWZ8UWFD)

这样冒泡的时候，Event.timeStamp 小于 事件的创建时间，那么该事件就不会触发了。



哈哈哈？以为这样就完了？直到我看到 vue-next-#2513

https://github.com/vuejs/vue-next/issues/2513

在iframe中，Event.timeStamp，是由该iframe被创建开始算起的。

所以会导致iframe中的点击事件的timeStamp，小于**源页面启动时间->创建该事件的时间**。

从而事件不被触发。

**除非你愿意等待这些时间。**

https://stackblitz.com/edit/vue3-iframe-teleport-demo?file=src%2FApp.vue

尝试一下，点击渲染iframe再疯狂点击count?你就会发现一开并不能触发，要等**页面启动时间->创建该事件的时间**才能触发事件。

count事件触发后，点击toggle，再重新渲染iframe，你会发现你要等待的时间更久了！