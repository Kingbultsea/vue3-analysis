## vue3渲染流程



## 一些问题
#### 1.当连续触发响应式的trigger会导致组件的effectComponent连续执行更新吗？
并不会。第一次触发trigger的时候会把事件队列状态改成Pending，把flushJobs丢进microtask来执行，第二次触发trigger会检测到当前状态如果是Pending或者是Flushing状态的时候将不进行下去。
可在packages/runtime-core/\_\_tests\_\_/rendererAttrsFallthrough.spec.ts 'should allow attrs to fallthrough'测试用例中调试。

![流程图](https://res.psy-1.com/FuStWNAasmJ0eOOjTm_nC_7ON0uz)

#### 2.patch到底有多少种类型？分别是干什么的？
在patch中查看有processText、processCommentNode、mountStaticNode、patchStaticNode、processFragment、processElement（流程图已有）、processComponent（流程图已有）、Teleport、SUSPENSE。
