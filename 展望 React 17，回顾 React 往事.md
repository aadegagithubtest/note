至 React 16 正式发布已经过去了一年，回望 React 16 当时带来的变化，如 Fragment，Portal，Error Boundary，SSR 优化等等期盼已久的 feature。最重要的其实还是 new architecture (Fiber) 的平稳落地。虽然在这之前的很多公开场合，React core team 的人已经多次谈论到这个新的架构，但是大家的关注点往往会被更上层的形式化的东西所吸引（就如同新的 React Fire 中大家更关心 className -> class 一样），当然了，这个不上一两句话能说清楚的，要展开讲就有点要扯皮了，我们这里就不开会了。



回到 React 16 的话题。要讨论未来（React 17），必须要了解过去。以 React 16 为分水岭，我们可以分为两方面来看这个问题。

一是为什么
二是怎么办
1. 为什么？
「是不是」这个问题我们就不讨论了，如果笔者没有穿越的话，应该是「是」的。

为什么要进行这样一次内核的架构调整或者说完全重现呢？我们能够找到的资料基本是从 Seb（Sebastian Markbåge） 提出 New Core Algorithm facebook/react 开始的。然后被熟知是通过 acdlite/react-fiber-architecture 。


https://github.com/facebook/react/graphs/contributors
Seb 是谁呢，其它的头衔这里就不展开了，熟悉 React repo 的人应该知道，16 年以后，即从 16 架构的提出以及至今整个最核心代码的设计和部分书写，绝大部分是由 Seb 来决定的。当然啦，这里没有否认其它人的劳动成果的意思，但是从客观上我们要知道这么一件事。


https://twitter.com/dan_abramov/status/996755754215530496
咳咳，扯得有点远了，回过头来。最初提出的时候，我们可以看到，这只是一个针对 reconciliation 阶段的重构的算法，注意一开始并没有提到 Fiber 这个概念。为什么要重构呢，很简单嘛，一些新的 feature 现有的算法不好弄，甚至没法弄。两个月后，在 facebook 上 Seb 正式提到了 Fiber 这个概念。到后来演变为对整个框架的重写。其中解释了为什么要进行重写：

Once you have each stack frame as an object on the heap you can do clever things like reusing it during future updates and yielding to the event loop without losing any of your currently in progress data.
reusing it during future updates: ReactChildFiber.js#L303
yielding to the event loop without losing any of your currently in progress data: ReactFiberScheduler.js#L1121
这些都已经体现在了目前的 React 代码中。


找不到 youtube 链接了，还望大家告知
说了这么多，到底有什么用呢？其实说到底，React 所做的一切（包括现在以至将来）的首要目的都是为了一件事：「用户体验」。只要从这个出发点入手，就能明白它的用意。（当然第二肯定是「开发体验」了）

注意这里的用户体验是框架层面的，不是 UI 设计层面的。对于框架而言，关注用户体验体现在两个方面，一是「快」，二是「流畅/自然」。

「快」很好理解，之前我 diff 要 800 ms，现在只要 500 ms，自然就快了。（怎么做呢，我们在第二点「怎么办」中再讲）

而「流畅/自然」这个就比较难理解和麻烦了。首先不知道它指的是什么，页面流畅，更新流畅，还是交互流畅？其实是一个问题。我们首先想想，假如用户用的都是超级无敌计算机，计算斐波拉切的 1E 项只需要 1ms，只要我们不搞个像死循环这样的对吧，很多问题都没有了。

但是问题是用户用的是小米 2 （主要是上周遇到个 bug 是这个的。。）这样的机器，这还不要紧，最重要的，我们是依赖于浏览器的。即，JS 本身的单线程（不要提 worker 和 stage 0 的一些提案了朋友们，注意这里的语境，后面不再阐述）的限制，加上浏览器自身（event loop）以及后台的多种工作的协调的限制。我们必须面临如下的问题：


https://github.com/tdresser/should-yield#the-problem
其中的第二点尤为重要，虽然这个图和 React 没什么关系，这是 chrome 的人提出的（虽然可能也参考 React）。但是这一点是说到根本上的问题之一了，即如何快速响应用户的交互。举例来说，用 React 以前的这种同步地递归式的方式，第一，无法在中途结束或者说停下来。第二，无法分成 chunk 去执行。造成的后果是，卡（主线程）和掉帧的风险增加。



总的来说，这便是主要的问题。日益降低的用户的忍耐力同落后的用户体验的矛盾已经代替了之前的日益增加的智能设备数同落后的生产力的矛盾。

所以 React 所在思考和解决的，本质就是如何实现更好的用户体验。这个笔者在如何评价React的新功能Time Slice 和Suspense？已经提到过。

当前了，生产力也是需要兼顾的，JSX 和 class component 在这之前已经首当其冲了。后面16 的 Fragment, .....。16.x 的 new context, new ref, new lifecycle api, profiler。17 的 .....(后面再说)。这些也反映了对开发者的人文主义关怀。

2. 怎么办？
好的，啰嗦了这么多，问题有了，咋做呢？

我们先不慌，我们先思考下，这些问题是 React 最先发现的吗？又仅仅是 React 需要解决的吗？

肯定不是啦，浏览器也意识到了这个问题，比如 chrome 很早在 timeline/performance 面板里就有 jank frame 的红色警告。而开发者也可以通过 w3c/longtasks 来检测。也包括其它的一些提案比如 first-input-delay 等等。但是这都是手段，没有解决根本问题，当然啦，也主要是解决不了，毕竟这是用户端（开发者）自己才能解决的事，所以更好的方式也是放到框架层面来解决。

所以具体到 React 要解决这些问题，就需要做到（虽然这个图也和 React 没有关系，但是我们还是可以用它来说明问题。）：


https://github.com/tdresser/should-yield#the-problem
a unit of work 或者说 chunk：想想要是以前的递归，这就完全没得搞了（当然也可以，只是很麻烦，总之弊大于利）。所以重写的第一步，改变内核的数据结构，即现在的 fiber（我们知道 fiber 是一种数据结构，当前还有另外一层含义）。改了之后第二步就有希望了，即将递归改成循环，然后我们就能一个一个处理了（至少在数据结构上更方便了，形式上还需要其它的变化）。也就是源码里的 performUnitOfWork 。

https://twitter.com/pomber/status/919937964104437761
fiber 的数据结构的应用，是 React16 的本质更新，而 Async Render 以及其它一些 feature 是 React17 的本质更新，所以从本质上来说，React 16 是 React 重构过程中的一个中间形态和毕竟过程。（毕竟一口吃个饼容易噎着。。）

2. continue the work：continue 就意味着两件事，interrupt 和 resume/continue。当然，也就意味着这是异步的了。这个毕竟复杂，我们放到后面 async render 的地方单独再讲。

ok，到现在我们整体粗略的介绍了下 React 16 之前的现状和 React 16 所做的努力。接下来我们就需要介绍 React 16 之后到 React 17 的变化了。



那么现在 React 16 已经发布了，我们前面说过这只是 React 重构过程中的一个分水岭，React 的最终目标还没有达到，那么 React 最终的目标是什么呢？

我们可以简单的概括为：Async render（完整的应该叫 Async Rendering，这里简化下）。



在讲这个之前，我们先回顾下我们之前谈的第二个目标：「流畅/自然」

那么怎么让浏览器的一切活动流畅呢，要回答这个问题，我们不妨从反方向思考，即如何让浏览器卡顿呢，这样就很简单了，只需要我们在对交互行为（不管是来自用户的交互还是 DOM 本身的动画）做出响应的时候消耗过长的时间，这样到浏览器绘制新的界面（即交互动作完成后更新后的界面）给用户看的时候，就有了一个时间差，用通俗的话来讲，就是用户感觉卡了。

我们在玩游戏的时候可能会对此有更直观的感受，比如就玩魂斗罗或者超级玛丽吧，你按了一下发射子弹的按钮，但是画面上并没有看到子弹，1 秒钟过后，敌人死了。。或者你在桥的这一头，按了半天前进按钮没反应，你没有移动，然后突然发现你已经在桥的这一头了。。



举例来说，比如一个纯数字的输入框，输入的数字代表要在界面上显示多少个气球。当我按下 1，接着按下 2 的时候：

以前的方式：必须等到 1 被处理完了之后，才能响应 2。这种响应体现在两个层面，一个是输入框里的数字的更新。一个是界面上气球的个数的更新。所以，如果处理 1 消耗的时间过长，那么就会出现非常糟糕的用户体验。即，当我按下 2 之后，一段时间内输入框里并不会出现 2，界面上的气球也不会变多。而当它更新和变多之后，我已经按下 3 甚至 45678 了。。实际上每次消耗的时间都比较长的话，中间的状态就全部丢失了，俗称丢帧，也就是上面提到的你看到的时候敌人已经死了，或者你已经在桥的这头了。比如 4 后面按了 5，6，7，8。你看到的是一瞬间 5678 已经都在屏幕上了，而不是按下 5 就显示 5，按下 6 就显示 6。
理想的（现在的）方式：当我按了 1 之后按下 2，用户期待的其实是先显示 1 个气球，然后显示 12 个气球（注意这就是为什么不能用 debounce 来代替）。


浏览器环境也是类似的道理。所以要想避免这种情况，我们需要做的就是：

不要妨碍浏览器自己的渲染更新逻辑（即 60 FPS 的帧渲染）。（一个消耗时间太长的任务就会妨碍浏览器的正常渲染工作，使它掉帧）
但是伟大的毛主席教育过我们，又要马儿跑，又不给马儿吃草是不行的。这也就是 Dan 说的：It's the same amount of work overall, so if it's not async, then it's sync and blocks the thread.（笔者译：总的工作量是一样的，所以如果它不是异步的，那么就必然是同步且阻塞的）。所以消耗时间太长的任务不应该怪罪在它本身消耗时间长（加上不是不合理的消耗），所以需要把它分割成很多小的片段，每次执行一点（避免阻塞造成掉帧），同时在下一次执行的时候，能够充分利用之前的执行的结果，从而尽最大努力减少工作量。
光靠利用之前，效果还不够，工作量是没法再减少了，但是马儿会不会没吃够呢？于是 React 的人自然也想到了利用空闲时间预渲染。熟悉的同学应该会想到浏览器早就有这招了，即 resource hint api 里的 preconnect, prfetch, prerender（虽然已经被废弃）等等。


所以为了实现这样的目标，React 将工作流划分为了两个阶段，Render 阶段和 Commit 阶段。

Render phase & commit phase
Render 阶段是可以被中断和分割的，那么怎么把设置 `innerHTML`，`textContent` 这样的东西分割出来呢。React 采用的方式是 effectTag 。

bvaughn：What is meant within the README of `create-subscription` by async limitations? Can it be clarified? · Issue #13186 · facebook/react

- The render phase determines what changes need to be made to e.g. the DOM. During this phase, React calls render and then compares the result to the previous render.
- The commit phase is when React applies any changes. (In the case of React DOM, this is when React inserts, updates, and removes DOM nodes.) React also calls lifecycles like componentDidMount and componentDidUpdate during this phase.
笔者译：划分 render 阶段和 commit 阶段的意义是：

组件在 render 阶段发生错误，React 能安全的丢弃正在处理中的任务，然后让 Error Boundary 去决定到底渲染什么。
如果有很多的组件需要 React 去渲染，React 能够将这些工作分割成更小的 chunk 执行，从而避免阻塞（block）浏览器。一旦所有的组件被渲染完毕（笔注：指 React 自身的 render，而不是渲染到界面上），react 能够同步地提交（commit）这些工作。比如，DOM 的更新。（这是新的实验性的 async rendering 模型的核心）（笔注：即异步 render，同步更新）
React 能够对任务划分优先级。如果低优先级的任务被处理的同时，更高的优先级的任务被调度了，React 能够安全地暂时搁置或者说暂停低优先级的任务，转而去处理更高优先级的任务。因为 React 只在 commit 阶段应用更新到 DOM 上，所以它不需要担心留下一个只有部分状态被更新的应用（笔注：即只更新需要更新的状态里的一部分并不会反映在 UI 界面上，所以不会出现界面上一部分更新了，一部分没更新，只有当所以组件更新完毕后才会一起 commit）
所以在源码中我们可以看到，在 Render 阶段 React 只给 fiber 打上各种 `effectTag` 标记，并不会进行任何实质的 DOM 操作，因为实际对应的 DOM 操作放到 commit 阶段去进行。



具体到代码层面怎么实现呢？答案就是 custom requestIdleCallback based on requestAnimationFrame。

Async Render

https://medium.com/@paul_irish/requestanimationframe-scheduling-for-nerds-9c57f7438ef4

https://medium.com/@paul_irish/requestanimationframe-scheduling-for-nerds-9c57f7438ef4
了解了上图 rAF 的过程，我们就可以知道，在一帧的空闲时间里分配给我们执行 js 的时间是有限的，如果我们执行 js 超过了这个时间，就会造成 lag，rAF 保证不了不超过，因为 js 是我们在执行。因此React 的进行了一些优化，自己封装了一下 rAF 从而实现了 rIC，如果超过了 rIC 被分配到的时间（通过 rIC 的第二个形参 `timeout` 判断即可，具体可以看规范），就暂停任务，则会到下一次执行任务的时候再去执行（和嵌套 rIC 或者 rAF 的表现是一致的，即嵌套的 rIC 会被放到下一个 rIC 中去执行，而不是当前的），但是浏览器自己的 rIC 有很多不适合的地方，没法用，所以才需要根据 rAF 自己封装了一下，实现了 rIC 的功能，同时更加细粒度。（当然还有一些 trick，可以参考 spanicker/main-thread-scheduling 里的分析）



浏览器的 rIC 不适合的地方在哪儿呢，主要有两点：

本身不是用来处理重要的任务的，所以被调用次数不够频繁，即帧率达不到
可能出现饥饿现象


基于此，我们就实现了我们的目标。而 React 把这叫做 Time Slicing。当然啦，顺带介绍的还有 Suspense。Suspense 又是什么呢？

Suspense
https://twitter.com/acdlite/status/954798592320942081 这里讲了为什么需要 Suspense 这样的 API，还是那句话，用户体验。。。

- One reason iOS feels so much nicer than the web: fewer unnecessary loading states. Look what happens when you tap an option in the Settings app: rather than transition immediately and show a spinner, it pauses for a moment until the view is ready.

- The responses to my original tweet are correct. When waiting for something to load, it’s best to stay on the current view and wait until it’s ready. Only show the loading state as a fallback, after some threshold is crossed.
如何评价React的新功能Time Slice 和Suspense？
​
www.zhihu.com
图标

嗯，这样的 API 应该是为了和 react-loadable 兼容。（最终定型就是上面这样，后面会讲到）

同时， 这里介绍了 suspense 的最初的想到的实现方式，后面被推翻了

以下两段之前写的，不写也不好，源码读者欢迎交流：

如果使用了 Suspense，那么 Suspense 的组件的 fiber 树里的更新很可能在不同的时间点执行，但是为了界面的不抖动，所以应该是把整棵树一起 flush 到界面上。所以 React 弄了个 ReactFiberPendingPriority 模块，目的就是让 Suspense fiber 树里的所有 fiber 的 expirationTime 保持一致，从而能够让它们被同时 flush 到界面上。pending 是还未执行的意思。

为什么 Suspense 的测试里面要用 `await advanceTimers(xxms)`，因为内部的 rIc 实现本质还是依赖于 rAF，所以执行 Suspense 相关任务的时候是在 current tick 的末尾也就是 microtask 里面执行的，所以要看效果也要到 current tick 的最末尾才看得到。而 `advanceTimers` 先执行 `jest.advanceTimersByTime()`, 然后返回一个 promise，自然就可以了。




Dan:

目前的 React 生态里的 code splitting 的方式会在「哪个组件去加载代码或者数据」里面耦合「在什么地方显示 spinner」的逻辑，Suspense 解决了这个问题（笔注：实现了解耦）。Placehoder 就像一个 `catch` ，会自动处理在它下方的依赖于它的异步的东西。（笔注：因为本质是 判断 catch 到的是不是 Promise 从而进行处理）
笔注：这里说的解耦个人理解是这样的。以前我们做的 Spinner，如常见的上拉加载，我们需要把 `Spinner` 和 `List` 放在**同一个**组件中，如在加载组件`AutoLoadList` 中 `return this.data? <List data={data}> : <Spinner>`。当然你把 Spinner 当成参数传入也行，不过本质还是必须都在 `AutoLoadList` 这一个组件中。Suspense 解耦了这一操作，加载和 Spinner 是**分开的**，你可以在外层，甚至外层的外层套一个 `Placehoder`，而在最里面的组件才进行加载数据（`resource.read(cache)`），两者互无感知。

还有比较重要的一点是，`Placehoder` 下面比如有 A, B（加载数据）, C，D（加载数据） 四个组件，A 和 C 可以完全不受影响照常显示以及和用户进行交互（即便 B 和 D 已经是 spinner 了），这在以前是无法做到的。比如上面说的 `AutoLoadList` 组件，你用 `<AutoLoadList fetch={fetchUserList} spinner={<Spinner />}><A /><B /><C /><D /></AutoLoadList>`，A 和 C 肯定是会受影响的。（当然你可以用比较 hack 的方式比如获取实例然后看它是不是 intanceof 或者有其它的类似`<A notAFetchComponent />`这种，都不说很..了，也会出现其它的限制，不可取）。
这将我们从 code splitting 的摩擦中解放了出来。你不再需要手动地控制和担心「那将会怎样影响 app 里能感知到的加载顺序」。你可以 code split 一个叶子节点，它上方的现有的 Placehoder 会 catch 住它。


总结，Suspense 的重点在于：

在请求数据的时候，能够不卸载之前的组件，即保持之前屏幕上的内容不变，同时可以交互（通过双缓存来实现）
叶子节点能够延迟某一整棵子树的渲染，即可以实现让视频里的电影海报 img 来延迟整个电影详情页的渲染的时机


说到这讲个题外话：

在 React16 之前，React 是无法感知到跳出被处理的函数「范围外」的 setState 的，比如你在某个 event handler 或者生命周期钩子里面执行 setTimeout，在 setTimeout 里面进行 setState，这个时候的 setState 就是同步的了（即执行完 setState 之后 this.state 就已经是最新的了），这是因为这个 setTimeout 实际上不属于 event handler 或者生命周期钩子的一部分，无法通过像 `感知...; func(); 感知...` 这样的流程被感知到，因为 `func()` 里面包含了异步的 setTimeout。同步的流程自然无法感知到。

而到了 16 之后可以实现感知了。那是因为用了全局变量的关系。



好了，到这基本上 React 17 也就讲得差不多了。前面说了 16 和 16.x 的很多 feature 都是为 17 服务的（特别是新声明周期钩子的改动），官方有博客这里就不多说了。



下面基本是漫谈了，我们先来看看 React 17 还有哪些 feature。

支持 Promise 组件
其实这只不过是稍微在 Suspense 的基础上改了改，所以这个的 PR 改动的地方都不是很多。

<PlaceHolder>
  <div></div>

  <A />
  <B />

  <div></div>
</PlaceHolder>

A:
<div>
  <div></div>

  <C />
</div>
React 期望的是，当 PlaceHolder 的任意子节点有 Lazy 组件（组件是一个 Promise）的时候，初次渲染时不要把除了这些 lazy 的组件之外的其它部分的渲染到界面上，它们觉得这样没有意义，而且体验不好（比如一个表格，内容是 lazy 的，你先把头部展示出来也没意义）。

React 希望的结果是，当这些 lazy 的组件 resolve 的时候，再一起展示整体，而关键的优化点在于，在初次渲染的时候，尽管需要等到 lazy 的组件 resolve 才能展示整体，它**能够**开始渲染其它的组件，只是不把它们 commit（即渲染到界面上）。这样实际上等到 lazy 的组件 resolve的时候，需要进行的工作就非常少了。

所以如果 resolve 的时间非常短，用户根本就不会看到任何的 spinner，只有当 resolve 的时间超过某个值（有默认，你也可以自己设置），才会展示 spinner。

这样就获得了两方面的优点，即让 UI 交互更加流畅，也让渲染性能能够跟上。

而且这样的设计，PlaceHolder 的位置可以非常灵活，你可以给整个 table 套上 PlaceHolder，也可以给某个单元格套上 PlaceHolder（这样 spinner 只会出现在单元格中）。



总的来说，目前你无法做到两件事，一是让 PlaceHolder 能够感知非常深的子组件里面的 Promise（除非你用 redux 这种）。二是套在最外层的 PlaceHolder 无法继续渲染那些不是 lazy 的子组件，只有当 lazy 的组件 resolve 的时候你才能去渲染，浪费了很多性能。

而且这种还可以搭配 Protal 用，组件的层级关系不再受影响，即之前没有 Protal 的时候，DOM 里面的 React root container 是无法感知到 Modal这种组件里的状态变化的

Render Props
最初的提出：`render` as a function of props and state · Issue #1387 · facebook/react

getDerivedStateFromCatch
Add stack unwinding phase for handling errors (#12201) · LeonYuAng3NT/react@a3f6936

与 cDC（componentDidCatch）的区别：https://www.reddit.com/r/reactjs/comments/9lp0k3/new_lifecycle_method_getderivedstatefromerror/e79elpl/?utm_source=reddit-android

Support lazy component

即前面 Suspense 里提到的 LazyloadComponent 。






注意两点，一是 Suspense 里就提到的，不要立即显示 spinner。二是复用 fetch data 里的 spinner（放在共同的的父级组件里就行了嘛）。



各种对 explicit 的诠释
Fiber 本身：一个 fiber 只需要在自己的数据结构上声明一下 expriationTime 就行，不需要自己去管我该什么时候去插入，更新。而是完全由 schedule 来控制
为什么不引入 keep-alive：can React support feature like keep-alive in Vue?


嗯，说了这么多，总的来总结一下 React 17：


https://www.reddit.com/r/reactjs/comments/9ksj2l/acdlites_react_roadmap_presentedframework_summit
「非前端技术」
Double buffering
polling object:

unwinding stack
continuations and CPS （嗯...这个很早就提到过了，很多非常多次，最早在看 SICP 的时候就听到过了，一直没有搞懂，这是我看过的最好的文章）
Bitwise operator（用了 fiber 的 effectTag）：Bitwise operators
- `if(tag & ...)`：判断是否有某个 tag

- `tag |= ...`：加上某个 tag

- `tag |= ~(...)`：去掉某个 tag

- `tag ^= ...`：toogle 某个 tag（有就去掉，没有就加上）

copy-on-write
当然代码或者 v8 相关的优化就比较多了，hidden class，hot path，还有好多想不起来也不知道叫啥名字的。。
可重入性：



FAQ
为什么要提出 Expiration Time，并废弃 ReactPriorityLevel 模块（注意没有废弃 priority 这个概念）？

答：在处理饥饿问题时，如果采用优先级算法，那么需要人为的去调整优先级（增加或减少），而用当前的时间与 expirationTime 的差值来代表优先级的话，就不用去认为解决饥饿问题了，因为时间自己就是流逝的，差值肯定会越来越小，优先级自然会自动增加，也就不会存在饥饿现象了



为什么去除对原生 requestIdleCallback 的依赖？（好像前面已经提到过了）

因为浏览器对调度回调不敏感
个人感觉另外的原因是支持太差


为什么不用 generator 去实现 yield，而用了其它的实现？

最初的思想基于 generator，但是源码中却完全没用用到过 generator，而是用了自己的一套方案去实现 generator。（其实一开始是用的，Seb 在演讲的时候提到过，后面去掉了），参考 Seb 的 comment）


React 的理念在于：

Sebastian god

Sebastian god

https://twitter.com/acdlite/status/1047187257852145664
其它未来
seb 最近把重心都放在了OCaml 和 prepack 上，后面应该是有大招的（猜测。。）

结语
写这个文章主要是因为从种种迹象来看，再过 20 天的 React conf 他们应该就要放大招了，到时候再写感觉积极性就失去了很多。



React 即是冰山，也是冰山的入口，打开之后有太多的东西能够学习和探索，很多时候看一个 issue，PR，tweet，都能感受到自己知识的局限和渺小。React 里面的很多东西和思想，包括变量名，函数名，甚至注释，都来自于其它语言，方向。而 React 本身也回馈给了开源社区，如 display-lock，immutable.js proposal，rIC 加强 Priority Queue Option · Issue #68 · w3c/requestidlecallback ，不依赖框架的 Scheduler 等等。



最后，希望能够抛砖引玉，在不久的将来能看到《React 编年史》之类的东西那就太高兴了（国外有一个类似的，但是太少了，三四百字符，自己也尝试过，确实水平和精力不够）。必须要承认，笔者自己是 15.0 之后才刚刚开始接触 React 的，所以对历史甚至是 15.0 的了解也不够深入，表现在没有系统的深入的了解过（除开官方文档的所有内容以及部分 gh，twitter，blog 的资料），即没有阅读过《深入 React 技术栈》类似这样的书籍，也没有阅读过 15 的源码等等。如有错误，还望大家批评纠正
