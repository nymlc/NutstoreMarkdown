我们知道`Virtual DOM`就是把页面节点用对象描述，这样子更新页面的时候就可以比对新旧对象然后更新有变动的那块即可，不用频繁操作`DOM`从而提高性能

而这个节点描述对象就是`VNode`，这个就是本章的重点，**如何设计VNode？**

# 描述真实节点

一个真实的节点对象是很庞大的，如下图所示

![image-20200329145005003](/Users/linchen/Library/Application Support/typora-user-images/image-20200329145005003.png)

这么庞大的对象我们当然不可能全部照搬的，其实只需要其中一部分就好了。就像我们`createElement`创建节点，我们只需要提供一些必要信息就行了，同样的道理`VNode`也只需要一些必要的信息就是了，至于哪些其实也很简单，就像下面这个

```html
<div class="app" style="color: #111;" @click="change">
    <section>
      	section
    </section>
</div>
```

首先我们想下从这个模板到`VNode`经过了什么？我们先把这模板词、语法分析生成`AST`，然后通过`AST`生成代码，最后通过这个代码生成`VNode`，也就是`VNode`所包含的其实就是模板所能表现得，在此例的`div`而言就是`class, style, click`以及子节点`section`

我们单以`div`节点来看可以表现成如下：
```js
let vnode = {
    tag: 'div',
    data: {
        staticClass: 'app',
        staticStyle: {
            color: '#111'
        },
        on: {
            click: function change() {
                // change function
            }
        }
    }
}
```

`data`就是用来存储这个节点上的各种属性、事件啥的，`tag`自然就是这节点名了

那我们如何描述这个子节点呢，那么自然就是加个`children`

```js
let vnode = {
    tag: 'div',
    data: {
        staticClass: 'app',
        staticStyle: {
            color: '#111'
        },
        on: {
            click: function change() {
                // change function
            }
        }
    },
    children: [{
        tag: 'section',
        data: undefined,
        children: [{
            tag: undefined,
            data: undefined,
            text: 'section'
        }]
    }]
}
```

值得注意的是这里`section`下的文本节点算是它的子节点，也得分层，而且`text`属性存储文本，其实也可以如下

```js
{
    tag: 'section',
    data: undefined,
    children: [{
        tag: undefined,
        data: undefined,
        children: 'section'
    }]
}
```

其实这完全看你怎么设计了，只要复用得当，语义通顺就是了

# 描述组件节点

组件节点和真实的节点不一样，就如下所示

```js
{
    components: {
        comp: {
            template: `<section>section</section>`
        }
    },
    template: `<comp/>`
}
```

这个模板是渲染一个`<comp/>`标签，但是实际上这个标签是不存在的，它是我们定义的一个组件，我们要渲染的是它对应的输出

```js
let vnode = {
    tag: 'comp',
    data: undefined,
    children: undefined
}
let vnode = {
    tag: 'section',
    data: undefined,
    children: [{
        tag: undefined,
        data: undefined,
        text: 'section'
    }]
}
```

但是这俩者其实都是它的描述，也就是这俩个都是对的