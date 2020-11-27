# 新版Diff 算法

![熊猫紧张](https://res.psy-1.com/Fr9pcXuMBigc_ofuRmebvi-XsUx_)好了好了，我们来聊聊diff吧，我就不上代码了...  其实之前哪些写的代码，是可看可不看的。

```typescript
const childOne = [1,2,3,4,5]
const childTwo = [1,2,3,4,6,5]
```

假设你的vnode拥有[1,2,3,4,5]子元素，你的vnode类型是ELEMENT，

拥有子5个vnode：[h('span', 1), h('span', 2), h('span', 3), h('span', 4), h('span', 5)]

现在把包含这5个vnode的数组简写成[1,2,3,4,5]。

进行初始渲染后，因为要更新child，所以再次进行patch。

> 如果子元素数组没有key，但是字符串相等，那么他们会被判定为同一个vnode。

> 啥是patch啊？emm... 竟然你都问了，你就当它是一个更新器吧



## 步骤一

从头对比是不是同一个vnode，是则跳过不做任何处理，不是则标记。

重复上步骤，从尾到头（标记点也算是头）。

按照上述，我们标记到4和5，目前没有进行任何的patch处理。

**如果没有尾或者头有相同的vnode，索引依旧在两端，并不是不存在。**

![img](https://res.psy-1.com/Fmod92Snutw8T_JM5zcVtReXWGmv)



## 步骤二

对两个标记之间的元素进行patch，也就是进行更新。

![img](https://res.psy-1.com/Fi6klYnkdrNsovrkxfSkFEqxgJb7)



## 步骤三

对从头标记之后尾标记之前的元素进行patch。

![img](https://res.psy-1.com/Fley670v7q7armu7AanOkAXKhuGo)



## 步骤四

对旧的子vnode要进行unmount。

![img](https://res.psy-1.com/Fl9Wlh8LcAIA42OPTCw7dTYge3yA)



## 步骤五（重点哦）

... 明天再更新~