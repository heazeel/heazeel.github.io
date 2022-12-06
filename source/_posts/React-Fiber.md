---
title: React Fiber
toc: true
date: 2022-12-05 17:58:00
tags:
categories: React
---

> 在React15及以前，Reconciler采用递归的方式创建虚拟DOM，递归过程是不能中断的。如果组件树的层级很深，递归会占用线程很多时间，递归更新时间超过了16ms，用户交互就会卡顿。

> 为了解决这个问题，React16将递归的无法中断的更新重构为异步的可中断更新，由于曾经用于递归的虚拟DOM数据结构已经无法满足需要。于是，全新的Fiber架构应运而生。

<!-- more -->

那么如何理解react中的fiber呢，两个层面来解释：

* 从运行机制上来解释，fiber是一种流程让出机制，它能让react中的同步渲染进行中断，并将渲染的控制权让回浏览器，从而达到不阻塞浏览器渲染的目的。
* 从数据角度来解释，fiber能细化成一种数据结构，或者一个执行单元。

react会在跑完一个执行单元后检测自己还剩多少时间（这个所剩时间下文会解释），如果还有时间就继续运行，反之就终止任务并记录任务，同时将控制权还给浏览器，直到下次浏览器自身工作做完，又有了空闲时间，便再将控制权交给react，以此反复。

每一个被创建的虚拟dom都会被包装成一个fiber节点，他具有如下结构：
```js
const fiber = {
  stateNode,  // dom节点实例
  child,      // 当前节点所关联的子节点
  sibling,    // 当前节点所关联的兄弟节点
  return      // 当前节点所关联的父节点
}
```

react中的fiber其实具备如下几点核心特点：
1. 支持增量渲染，fiber将react中的渲染任务拆分到每一帧。（不是一口气全部渲染完，走走停停，有时间就继续渲染，没时间就先暂停）
2. 支持暂停，终止以及恢复之前的渲染任务。（没渲染时间了就将控制权让回浏览器）
3. 通过fiber赋予了不同任务的优先级。（让优先级高的运行，比如事件交互响应，页面渲染等，像网络请求之类的往后排）
4. 支持并发处理（结合第3点理解，面对可变的一堆任务，react始终处理最高优先级，灵活调整处理顺序，保证重要的任务都会在允许的最快时间内响应，而不是死脑筋按顺序来）

> 在一些极端情况下，浏览器会最多给出50ms的空闲时间给我们处理想做的事情，比如我们一些任务非常耗时，浏览器知道我们会耗时，但为了让页面呈现尽可能不要太卡顿，同时又要照顾JS线程，所以它会主动将一帧的用时从16.66ms提升到50ms，也就是说此时1S浏览器至多能渲染20帧

> react在最终实现上并未直接采用requestIdleCallback，一方面是requestIdleCallback目前还是实验中的api，兼容性不是非常好，其次考虑到剩余时间提升到50ms也就20帧左右，体验依旧不是很好。于是react通过`MessageChannel + requestAnimationFrame` 自己模拟实现了requestIdleCallback

1. 当前帧结束时间： 我们知道requestAnimationFrame的回调被执行的时机是当前帧开始绘制之前。也就是说rafTime是当前帧开始时候的时间，如果按照每一帧执行的时间是16.66ms。那么我们就能算出当前帧结束的时间， frameDeadline = rafTime + 16.66。
2. 当前帧剩余时间：当前帧剩余时间 = 当前帧结束时间(frameDeadline) - 当前帧花费的时间。关键是我们怎么知道'当前帧花费的时间'，这个是怎么算的，这里就涉及到js事件循环的知识。react中是用MessageChannel实现的。


```js
let frameDeadline // 当前帧的结束时间
let penddingCallback // requestIdleCallback的回调方法
let channel = new MessageChannel()

// 当执行此方法时，说明requestAnimationFrame的回调已经执行完毕，此时就能算出当前帧的剩余时间了，直接调用timeRemaining()即可。
// 因为MessageChannel是宏任务，需要等主线程任务执行完后才会执行。我们可以理解requestAnimationFrame的回调执行是在当前的主线程中，只有回调执行完毕onmessage这个方法才会执行。
// 这里可以根据setTimeout思考一下，setTimeout也是需要等主线程任务执行完毕后才会执行。
channel.port2.onmessage = function() {
  // 判断当前帧是否结束
  // timeRemaining()计算的是当前帧的剩余时间 如果大于0 说明当前帧还有剩余时间
  let timeRema = timeRemaining()
	if(timeRema > 0){
    	// 执行回调并把参数传给回调
		penddingCallback && penddingCallback({
      		// 当前帧是否完成
      		didTimeout: timeRema < 0,
      		// 计算剩余时间的方法
			timeRemaining
		})
	}
}
// 计算当前帧的剩余时间
function timeRemaining() {
    // 当前帧结束时间 - 当前时间
	// 如果结果 > 0 说明当前帧还有剩余时间
	return frameDeadline - performance.now()
}
window.requestIdleCallback = function(callback) {
	requestAnimationFrame(rafTime => {
      // 算出当前帧的结束时间 这里就先按照16.66ms一帧来计算
      frameDeadline = rafTime + 16.66
      // 存储回调
      penddingCallback = callback
      // 这里发送消息，MessageChannel是一个宏任务，也就是说上面onmessage方法会在当前帧执行完成后才执行
      // 这样就可以计算出当前帧的剩余时间了
      channel.port1.postMessage('haha') // 发送内容随便写了
	})
}
```

那么到这里，我们阐述了react 15以及之前的大量dom渲染时卡顿的原因，从而介绍了帧的概念。

紧接着我们引出了fiber，那么什么是fiber呢？往小了说它就是一种数据结构，包含了任务开始时间，节点关系信息（return,child这些），我们把视角往上抬一点，我们也可以说fiber是一种模拟调用栈的特殊链表，目的是为了解决传统调用栈无法暂停的问题。

而站在宏观角度fiber又是一种调度让出机制，它让react达到了增量渲染的目的，在保证帧数流畅的同时，fiber总是在浏览器有剩余时间的情况下去完成目前目前最高优先级的任务。

所以如果让我来提炼fiber的关键词，我大概给出如下几点：

* fiber是一种数据结构。
* fiber使用父子关系以及next的妙用，以链表形式模拟了传统调用栈。
* fiber是一种调度让出机制，只在有剩余时间的情况下运行。
* fiber实现了增量渲染，在浏览器允许的情况下一点点拼凑出最终渲染效果。
* fiber实现了并发，为任务赋予不同优先级，保证了一有时间总是做最高优先级的事，而不是先来先占位死板的去执行。
* fiber有协调与提交两个阶段，协调包含了fiber创建与diff更新，此过程可暂停。而提交必须同步执行，保证渲染不卡顿。

而通过fiber的协调阶段，我们了解了diff的对比过程，如果将fiber的结构理解成一棵树，那么这个过程本质上还是深度遍历，其顺序为父—父的第一个孩子—孩子的每一个兄弟。

通过源码，我们了解到react的diff是同层比较，最先比较key，如果key不相同，那么不用比较剩余节点直接删除，这也强调了key的重要性，其次会比较元素的type以及props。而且这个比较过程其实是拿旧的fiber与新的虚拟dom在比，而不是fiber与fiber或者虚拟dom与虚拟dom比较，其实也不难理解，如果key与type都相同，那说明这个fiber只用做简单的替换，而不是完整重新创建，站在性能角度这确实更有优势。

最后，附上fiber更新调度的执行过程：
![](https://lost-and-find.oss-cn-hangzhou.aliyuncs.com/blog/React-Fiber/fiber.png)