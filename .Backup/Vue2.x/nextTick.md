# 前言
可先看[EventLoop](https://www.jianshu.com/p/20997517b0ea)
具体参见[src/core/util/next-tick.js](https://nymlc.github.io/vue2.x-analysis/src/src/core/util/next-tick.js)

什么是`tick`？`EvenLoop`每一次循环（`一个mactask、一串mictask`）就是一个`tick`
# 正文
### flushCallbacks
这里唯一一点是`copies`，如下所示，要是没有复制，那么当循环执行到外层回调时，就会执行内部的`$nextTick`，这时候内层回调会被添加到本次`tick`，也就提前执行了
```
this.$nextTick(() => {
    this.$nextTick(() => {
        console.log('nest')
    })
})
```
### macroTimerFunc、microTimerFunc取舍
在2.4版本之前都是使用`microtask`，但是在某些情况下它具有太高的优先级并且可能在连续顺序事件之间（[＃4521](https://github.com/vuejs/vue/issues/4521)，[＃6690](https://github.com/vuejs/vue/issues/6690)）或者甚至在同一事件的事件冒泡过程中之间触发（[＃6566](https://github.com/vuejs/vue/issues/6566)）。要是全部改成`macrotask`，对一些重绘或者动画的场景有性能影响（[#6813](https://github.com/vuejs/vue/issues/6813)）。所以在之后的版本默认使用`microtask`，但是有需要（`v-on`事件处理）强制使用`macrotask`（`src/src/platforms/web/runtime/modules/events.js`里的`handler = withMacroTask(handler)`）

### macroTimerFunc
macroTimerFunc就三个选择，依次降级setImmediate、MessageChannel、setTimeout
因为[setTimeout](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setTimeout)、[timers](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers)表现，`setTimeout`调用间隔至少`4ms`，实际上受到任务队列影响，这个时间明显不止，那么没有最小延迟的`setImmediate、MessageChannel`明显更适合了
>`setImmediate`在高版本IE以及Edge才支持
>![setImmediate](https://upload-images.jianshu.io/upload_images/4874009-fc46d400588943b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 例子
>[demo](https://nymlc.github.io/vue2.x-analysis/demo/nextTick/1.html)
>change1与change2区别在于看`flushCallbacks`。cbs是顺序加入，在改变`txt`变量（对应的`render watcher`）之前加入的`nextTick`执行的时候页面还未变化，因为这个`render watcher`在其之后，页面还未渲染