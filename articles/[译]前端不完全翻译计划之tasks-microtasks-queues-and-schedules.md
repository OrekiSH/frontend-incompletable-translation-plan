## 译者序
原文地址: https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/

作者Jake Archibald为Google Chrome团队的英国籍工程师, 原文发表与其[个人博客](https://jakearchibald.com/)

为了保证行文流畅,译文并未完全依据原文进行翻译,并根据译者自身理解对部分关键字进行了高亮.译者水平有限, 文中难免有疏漏和错误, 希望读者不吝批评指正.

本译文共**7718**字, 阅读大致需要花费**20**分钟.

## 正文

当我与我的同事[Matt Gaunt](https://twitter.com/gauntface)提及,自己打算写一些关于**microtask在浏览器的event loop中的入队和执行**的东西时，他这么对我说:"老实跟你讲Jake, 我是不会看的"．好吧, 尽管如此我还是写下了这篇文章，还望诸君就坐,与我一同读完这篇文章．

事实上你若是更偏爱视频，[Philip Roberts](https://twitter.com/philip_roberts)在[JSConf上精彩的演讲](https://www.youtube.com/watch?v=8aGhZQkoFbQ)会更适合你．他尚未提及microtask，但对event loop中剩余之处进行了详细的阐述．总之先让我们言归正传...

看一眼下面的JavaScript代码, 你觉得打印的顺序会是如何?
```js
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```
正确答案是: `script start`, `script end`, `promise1`, `promise2`, `setTimeout`, 但是就浏览器的各自的表现而言，着实非同寻常．

Microsoft Edge, Firefox 40, iOS Safari以及桌面端的Safari 8.0.8先打印了`setTimeout`而后打印`promise1`, `promise2`. --这似乎是由竞态条件(race condition)导致的，可奇怪的是，Firefox 39与Safari 8.0.7又始终打印出了正确的顺序．

## 为什么会这样呢
为理解上文代码的打印顺序，须知event loop如何处理task与microtask．你之前若是从未接触过相关的内容或许会稍有理解不畅，不妨先做个深呼吸...

**每个"线程"都应有自己的event loop**，对于一个web worker而言，有自己的event loop是其独立执行脚本的条件，而对于同源的window而言共享的event loop是它们能够同步通信的前提．

event loop始终在不间断的运行,孜孜不倦地执行每个出队的task．一个event loop存在多个任务来源(译注: 像是鼠标的点击或是发送网络请求),每个来源的task执行顺序是固定的([像是IndexDB](http://w3c.github.io/IndexedDB/#database-access-task-source)等规范就定义了自己的执行顺序)．但是，浏览器只能为event loop的一次执行从某个任务来源选取一个task,否则浏览器就不能给那些稍不留神就会有性能问题的task**更高的优先级**,像是用户的输入．嘿,你还在听吗...

只有当task计划执行时,浏览器才能将执行逻辑从内部区域转向操作JavaScript/DOM的区域,并且能够让这些操作有序的执行．**在上一个任务执行完成下一个任务尚未执行的时候，浏览器可能会更新渲染**.从鼠标点击事件到事件回调需要安排一个任务，同样的解析HTML以及上文的setTimeout也是如此

setTimeout在等待了指定的延迟时间后**为其回调安排了一个新的task**, 这就解释了为何`setTimeout`在`script end`之后被打印:　打印`script end`是第一个task,而打印`setTimeout`是另一个单独的task．

好了，我们就快结束这部分内容了，我需要你为接下来的内容打起精神来..

microtasks常用于当前运行脚本执行结束后需即刻执行的代码逻辑, 像是响应一系列的操作,或是想让一些事情异步执行而不需要付出一个新的task的代价．microtask队列**只要当前没有JavaScript脚本正在执行，且当前任务所有的回调已经执行完成之后**就会被立即处理．当microtask被推入event loop的队尾，额外产生的microtask都会被入队并处理．microtask包括mutation observer回调，以及上文例子中的promise回调

当一个**promise结束(settle, 即promise的状态变为fulfilled或是rejected, 而不再是pending)或是已经结束**时，它的回调会作为一个microtask入队．这能够保证即使promise早已结束(译注: 例如`const p = Promise.resolve();`)，promise的回调仍是异步的.

那么调用紧接着一个已经结束的promise的`.then(yey, nay)`方法会立刻入队一个microtask也就不难理解了．这就是`promise1`和`promise2`在`script end`之后打印的原因: **microtask必须在当前运行的脚本执行结束后才能被处理**．`promise1`和`promise2`在`setTimeout`之前被打印的原因是由于:**microtask永远在下一个task执行之前被处理**

好了，让我们一步步来:

event loop的队列中存在`Run script`这一task

=> setTimeout的回调作为一个task被推入某个task队列A

=> Promise的回调作为一个microtask被推入microtask队列B

=> `Run script`执行完成后，执行Promise的回调

=> Promise的回调返回undefined, Promise的下一回调作为一个microtask被推入B, 执行Promise的下一回调

=> 这两个microtask执行完成后,`Run script`这一task就算结束了，浏览器可能会更新渲染

=> 执行setTimeout的回调


是的，我写了一个可以一步步执行的动画示意图．周末过得如何?与友人畅游山水?猜猜我周末干了什么(微笑)．

嗯,为了避免有人不清楚我炫酷的UI设计，试试点击上面的箭头．(译注: Jake在他的文章里放了可交互的动画示意图，强烈推荐[点击原文](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)查看)

## 为什么一些浏览器的表现不一样?

一些浏览器打印的顺序是`script start`, `script end`, `setTimeout`, `promise1`, `promise2`．它们会在`setTimeout`之后调用promise回调．这可能是由于**它们调用promise回调的时候将它视为一个新的task而不是一个microtask**

出现这样的情况多少情有可原．因为promise是作为ECMAScript规范而不是HTML规范提出的.ECMAScript规范中有和microtask相似的概念叫做"jobs"，但是它们之间的关系分外模糊，除了在那个[实际上也不甚明晰邮件列表讨论](https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16)上有所界定．话虽如此，大家的最终共识还是觉得应该将promise视为microtask队列的一部分，这么做是为了大家好．

由于回调可能会被任务相关的某些东西(像是渲染)不必要的延迟，将promise视为任务会导致性能问题．同时也导致了由多个任务来源交互带来的不确定性(译注: 同样的输入不一定有同样的输出),并且打乱了与其他API的交互，不过这些我们以后再说

这是一个Edge会将promises视为microtasks的[公告](https://connect.microsoft.com/IE/feedback/details/1658365)，Webkit测试版的表现也是正常的，那我就姑且认为Safari最终还是会修复这个问题．从Firefox 43的表现来看这个问题已经被修复了．

十分有趣的是Safari和Firefox在问题修复前都经历了历史的倒退,我在想这是否只是个巧合．

## 如何分辨用的是task还是microtask
测试是一个法子.看打印promise和setTimeout的相对位置，虽然这依赖于(浏览器)的实现是否正确．

更现实的方法是，去看规范．举例来说，[第14步setTimeout](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timer-initialisation-steps)入队一个task,[第5步入队一个mutation record](https://dom.spec.whatwg.org/#queue-a-mutation-record)则入队一个microtask．

上文也提到了，在ECMAScript领域，他们将microtasks称为"jobs",在[第8.a步的PerformPromiseThen](http://www.ecma-international.org/ecma-262/6.0/#sec-performpromisethen),EnqueueJob被用于入队一个microtask.

现在让我们来看一个更复杂的例子．镜头一转，一个满脸焦急的(JS)学徒大喊: "不，他们还没有准备好!"．别理他，你已经准备好了,让我们开始..

## 一级Boss战
在写这篇文章之前我自己也弄错了一些概念，来看下面的一段html:
```html
<div class="outer">
  <div class="inner"></div>
</div>
```
结合下面这段JS，你觉得如果我点击了`div.inner`会打印出什么?
```js
// Let's get hold of those elements
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// …which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

## 谁是对的?

![bossfight](https://raw.githubusercontent.com/OrekiSH/frontend-incompletable-translation-plan/master/pictures/tasks-microtasks-queues-and-schedules/bossfight.png)

"click"事件的触发是一个task，而Mutation observer和promise的回调作为microtask入队，`setTimeout`的回调作为一个task入队,所以真相是:

event loop的队列中存在`Dispatching click event`这一task

=> setTimeout的回调作为一个task(译注: 姑且叫T1)被推入某个task队列A

=> Promise的回调作为一个microtask(译注: 姑且叫PMT1)被推入microtask队列B

=> mutation为处理observers将一个microtask(译注: 姑且叫MMT1)被推入B

=> **虽然`Dispatching click event`这一task还在执行中,但是由于JS栈是空的(不存在同步JS脚本的执行),microtasks在回调后被处理**

=> 执行PMT1

=> 执行MMT1

=> 事件冒泡, 所以回调(译注: onClick函数)再次被`outer`元素调用

=> setTimeout的回调作为一个task(译注: 姑且叫T2)被推入A

=> Promise的回调作为一个microtask(译注: 姑且叫PMT2)被推入B

=> mutation为处理observers将一个microtask(译注: 姑且叫MMT2)被推入B

=> 执行PMT2

=> 执行MMT2

=> 执行T1

=> 执行T2

所以Chrome的实现才是正确的．我的"新发现"是microtask是在回调之后被处理的(当没有其他的JS脚本在执行)．我之前以为是被限定于task结束之后．这条规则来自HTML规范中的调用一个回调小节:

> If the stack of script settings objects is now empty, perform a microtask checkpoint — HTML: Cleaning up after a callback step 3

...一个microtask检查点需要检查整个microtask队列,除非我们已在处理microtask队列(译注: 应该指的是event loop的标志位(performing a microtask checkpoint)从开始处理microtask队列到结束的这段时间都为true).同样地，ECMAScript对于jobs是这样描述的:

> Execution of a Job can be initiated only when there is no running execution context and the execution context stack is empty… — ECMAScript: Jobs and Job Queues

虽然和HTML规范相比，"可以"变成了"必须"．

## 为什么其他浏览器都做错了?
就mutation的回调来看，Firefox和Safari在两个click监听之间正确地清空了microtask队列，但promise似乎是以不同的顺序入队的．由于jobs和microtasks的模糊界定，这多少情有可原，尽管如此我仍然希望可以在两个监听回调之间执行．

对于Edge而言我们已经看出它错误地将promise出队，而且它也没有在click监听之间清空microtask队列，而是在所有监听调用结束之后，这使得只有一个`mutate`被打印于两个`click`之后


## 一级boss发怒的表哥
哇.继续用上面的例子,当我们执行`inner.click()`的时候会发生什么?

这同之前一样仍会触发事件，只不过用的是脚本而不是真正的用户交互

![boss_angry_older_brother](https://raw.githubusercontent.com/OrekiSH/frontend-incompletable-translation-plan/master/pictures/tasks-microtasks-queues-and-schedules/boss_angry_older_brother.png)

我发誓我在Chrome中始终得到的是不同的结果，我更新了上面的那幅图无数次最终才意识到我测试的是Chrome Canary．如果你在Chrome中得到了不同的结果，请在评论中告诉我你用的版本．

## 为什么和之前的不一样?
事情的经过应该是这样的:

event loop的队列中存在`Run script`这一task

=> setTimeout的回调作为一个task(译注: 姑且叫T1)被推入某个task队列A

=> Promise的回调作为一个microtask(译注: 姑且叫PMT1)被推入microtask队列B

=> mutation为处理observers将一个microtask(译注: 姑且叫MMT)被推入B

=> **JS栈非空(存在`inner.click()`这一同步JS脚本),不处理microtasks**

=> 事件冒泡, 所以回调(译注: onClick函数)再次被`outer`元素调用

=> setTimeout的回调作为一个task(译注: 姑且叫T2)被推入A

=> Promise的回调作为一个microtask(译注: 姑且叫PMT2)被推入B

=> MMT仍处于pending, 不添加另外的mutation microtask

=> `Run script`执行完成后，执行PMT1

=> 执行MMT

=> 执行PMT2

=> 执行T1

=> 执行T2

所以正确的顺序是: `click`, `click`, `promise`, `mutate`, `promise`, `timeout`, `timeout`, Chrome似乎是对的

在每个监听回调被调用后...

> If the stack of script settings objects is now empty, perform a microtask checkpoint — HTML: Cleaning up after a callback step 3

同上个例子一样，microtasks应当在监听回调之间运行，但是`.click()`导致事件被同步地触发了，所以调用`.click`的脚本仍在回调之间的(JS)栈里，上文的规则确保**microtasks不会打断正在执行的JavaScript脚本**．这意味着我们不会在监听回调之间处理microtask队列，它们会在两个监听回调执行结束后被处理．

## 知道这些有什么用吗?
呃, 它也许会躲在阴暗的小角落里偷偷咬你一口(哎呦)．我尝试[用promise为IndexDB做一层简单的封装](https://github.com/jakearchibald/indexeddb-promised/blob/master/lib/idb.js)而不是用怪异的IDBRequest对象时就遇到了一些问题．(浏览器的错误实现)几乎让使用IDB成了一个笑话．

当IDB触发了一个Success事件,[关联的事务对象变为非激活状态](http://w3c.github.io/IndexedDB/#fire-a-success-event)(第4步)．如果我创建了一个于该事件触发后resolve的promise,它的回调应该在第4步之前,当事务仍出于激活状态的时候运行，但是除了Chrome其他浏览器都不是这么做的，这使得这个库有点鸡肋．

实际上你可以在Firefox中解决这个问题，因为像es6-promise之类的promise polyfills使用了(正确使用microtask的)mutation observer执行回调，Safari似乎在修复microtask的bug后仍存在竞态条件，不过那应该归咎于他们[对IDB糟糕的实现](http://www.raymondcamden.com/2014/09/25/IndexedDB-on-iOS-8-Broken-Bad)，不幸的是，IE/Edge始终是无法使用的，因为mutation事件压根就不在回调之后被处理．

但愿我们可以尽快看到一些(IE/Edge)的可操作性．

## 你做到了!
总结一下:

- task顺序执行, 并且浏览器可能会在两个任务之间更新渲染
- microtask顺序执行, 并且在满足以下条件的任意一个的时候被执行:
  - 没有其他Javas脚本正在执行的每个回调之后
  - 每个task执行结束后

希望你对event loop有了自己的认识，或者说，至少有了小憩一会的理由．

事实上我想知道，还有人在读吗?有人吗?有吗?
