# 进程和线程
我们打开电脑可以边听歌边写代码，这就是多任务。但是多年前的电脑可是单核的，它其实是让各个任务交替运行，只是每个任务运行很短的时间就切换到下一个任务，这样子就好像并行。其实真正的多任务并行只能在多核CPU上。不过任务一般都远远多于核心，所以其实还是多任务交替执行
对于操作系统而言，一个任务就是一个进程。一个任务一般而言不会只干一件事，就像看电影，不可能先看完在听音吧，这么些子任务其实就是线程，它和多任务一样也是交替执行
**举个栗子，我们打开浏览器一个页签就是创建了一个进程，然后这个进程里有渲染线程、JS引擎、http线程等等**
可以把这俩当做单位，操作系统能够调度的单位。
>程序是指令、数据以及其组织形式的描述，进程就是程序的实体，也是线程的容器

# 并发和并行
并行是指同时执行，这个无论是宏观还是微观，程序都是一起执行的
并发是指在一个时间段之内多个程序执行，这个在微观上是顺序执行的，宏观上是同时的

# 宏任务和微任务
1. 宏任务（macrotask）也叫`tasks`，一些异步任务的回调会被放进这个队列等待后续被调用，包括：setTimeout、setInterval、setImmediate、I/O、requestAnimationFrame（浏览器）、UI rendering（浏览器）、MessageChannel、ajax、script（JS文件）、postMessage
2. 微任务（microtask）也叫`jobs`，另外一些异步任务会掉会被放进这个队列，包括：Promise、Object.observe（已废弃，类似`Proxy`）、MutationObserver（监听`DOM`变化）、process.nextTick（Node）
# 执行栈
JS代码运行是在`执行上下文`里进行的。有三种情况：
+ 全局执行上下文，代码首先进入的环境，在浏览器里是Window对象
+ 函数执行上下文，函数被调用时的环境
+ eval执行上下文，它会产生自己上下文
一般来说我们代码不止一个上下文，这时候我们就会有个`栈`来存储这么些上下文，先进后出。如下所示
![](https://upload-images.jianshu.io/upload_images/4874009-af02d9e5ec061fc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![栗子](https://upload-images.jianshu.io/upload_images/4874009-09aec7a88a29f6a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其实这个浏览器里的可能还不是那么好理解，如下图所示：
```javascript
console.log('script start');

setTimeout(function cb1() {
      console.log('setTimeout');
}, 0);

Promise.resolve().then(function cb2() {
      console.log('promise1');
}).then(function cb3() {
      console.log('promise2');
});

console.log('script end');
```
[写了个小栗子](https://nymlc.github.io/blogCode/2019/11/10/task/)，参见[Jake Archibald 的[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

+ 入栈`main()`全局上下文，`main()`加入`Tasks`队列
+ 打印`script start`
+ `cb1`加入`Tasks`队列（因为`setTimeout`是异步任务，交由`timer模块`处理，因为是0所以立即（有一点点延迟，看各浏览器情况）加入）
+ `cb2`加入`Jobs`队列
+ 打印`script end`
+ 退栈`main()`，执行`Jobs`
+ `Jobs`入栈
+ 打印`promise1`
+ `cb3`加入`Jobs`队列（Jobs执行期间，加入的也算本轮次）
+ 退栈`cb2`，`Jobs`移除`cb2`
+ `cb3`入栈
+ 打印`promise2`
+ 退栈`cb3`，`Jobs`移除`cb3`
+ 一次事件循环完毕，这时候浏览器可能进行页面渲染
+ `Tasks`移除`main`
+ 取`Tasks`的`task`入栈
+ 打印`setTimeout`
+ 退栈`cb1`，`Tasks`移除`cb1`
+程序执行完毕

>栈的容量是有限制的，一旦存放过多，是会爆栈的

>Tasks和Jobs交替入栈，区别在于Tasks里的先入栈，且单个入栈，Jobs的在栈空了被全部执行

>Jobs执行期间新加入的job也算本轮次
# 任务队列
其实执行栈的时候就说到这了
JS是单线程源于JS出现为了解决用户互动一些简单任务，多线程的话就有复杂的同步问题
不过单线程就意味着需要排队，所以任务队列应运而生，只要先不管IO、http之类的任务，等到它们有了结果再回过头来继续执行。
任务分俩种，`同步任务`、`异步任务`。`同步任务`就是主线程上顺序执行的任务，`异步任务`就是不进入主线程，而是进入`任务队列`的任务。它们都有相应的模块去处理（DOM Binding、network、timer），就像setTimeout由timer模块处理，到点之后就会把回调函数加入到`任务队列`。
`异步执行`机制如下：
+ 所有的同步任务都在主线程上执行，形成`执行栈`
+ 执行栈之外还有一个`任务队列`，只要异步任务完成了就会加入到`任务队列`
+ 执行栈空了就会任务队列里的任务，然后将其压入执行栈开始执行，**只不过是先执行Tasks的队首，然后执行整个Jobs**
+ 就是不断重复上三步
>JS的异步本质上来看还是同步的
# 浏览器线程
浏览器通常有GUI渲染线程、JavaScript引擎线程、定时触发器线程、事件触发器线程、异步HTTP请求线程。以定时触发器线程为例，遇到`setTimeout`的话就会交给它来处理，到点就加到`tasks队列`
# 后语
其实到这已经明白了，它是宿主环境为了解决JS单线程运行不阻塞的一种机制。
**不过上面所诉其实都是浏览器的表现，即热式一种机制，不同的宿主自然也不一定一样的**
+ 浏览器的在[html5规范](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)有定义，具体实现看各大厂商
+ NodeJS的是基于`libuv`实现，它已经实现了。[Node官网](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)、[libuv官网](http://docs.libuv.org/en/v1.x/design.html)
