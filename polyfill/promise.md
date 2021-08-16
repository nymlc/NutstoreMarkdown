栗子都在此[https://github.com/nymlc/polyfill/tree/master/promise](https://github.com/nymlc/polyfill/tree/master/promise)

这个其实是对`promise-polyfill`[https://github.com/taylorhakes/promise-polyfill](https://github.com/taylorhakes/promise-polyfill)的解析，`Promise`的实现还是很自由的，只要符合规范就是了，这个库很精简，适合用来学习`Promise`原理

这个以几个实现由简入难看下它的实现思路，首先简单看下`Promise A+`规范

# Promise规范小结

其实`Promise A+ `规范（[http://malcolmyu.github.io/malnote/2015/06/12/Promises-A-Plus/#x_%E4%B8%BA%E5%AF%B9%E8%B1%A1%E6%88%96%E5%87%BD%E6%95%B0](http://malcolmyu.github.io/malnote/2015/06/12/Promises-A-Plus/#x_%E4%B8%BA%E5%AF%B9%E8%B1%A1%E6%88%96%E5%87%BD%E6%95%B0)）还是很简单的，就这个`thenable`需要关注下

```js
const thenable = {
    then: function(resolvePromise, rejectPromise) {
        console.error(this === thenable) // true
        resolvePromise('success')
        rejectPromise('error')
    }
}
// 和下文类似
// const promise = Promise.resolve(thenable)

const promise = new Promise(function (resolve, reject) {
    resolve(thenable)
})
promise.then((arg) => {
    console.error(arg)
}).catch(arg => {
    console.error(arg)
})
```

可见`thenable.then`是函数的话这个函数将会接收俩个参数，且`this`指向`thenable`

# Step1
```js
const promise = new Promise((resolve) => {
    resolve(1)
})

promise.then((val) => {
    console.log(val)
    return val
})
```
这是个很简单的例子，我们假设这个`resolve`就是同步执行，这个`then`方法也不能链式调用，更没有`catch、all`等一系列方法也没有任何的终值是`thenable`、`Promise`实例的处理
这样子就很简单了，我们直接看实现[https://nymlc.github.io/polyfill/promise/index.html#1](https://nymlc.github.io/polyfill/promise/index.html#1)

可见实现很是简单，流程就是`Promise.prototype.constructor  -->  doResolve  -->  resolve  -->  then`。前俩步其实必然是同步的，第二步到第三步是可以异步的，不过这里暂不管。`then`的话必然是异步的
我们就以这个步骤简单讲下，在看代码之前我们先从`Promise`的使用角度说下这个代码的执行流程
这个很简单，我们也知道它只接收一个函数参数`fn`，这个函数会有俩个参数`resolve, reject`分别用于将状态从`pending`向`fulfilled, rejected`转变，并将结果作为参数传递出去，这样子后续的`then, catch`可以取到

## Promise.prototype.constructor
```js
constructor(fn) {
    this._value = undefined
    this._state = 0
    doResolve(this, fn)
}
```
首先我们将数据都存在实例对象之上，毕竟用到这些数据的都是`then, catch`之类的原型实例方法。`_state`就是这个状态，`0, 1, 2`分别代表`pending, fulfilled, rejected`
## doResolve
这个就是用于执行传入的`fn`，这个是立即执行的
```js
function doResolve(self, fn) {
    let done = false
    try {
        fn((nVal) => {
            if (done) {
                return
            }
            done = true
            resolve.call(self, nVal)
        }, (reason) => {
            if (done) {
                return
            }
            done = true
            reject.call(self, reason)
        })
    } catch (error) {
        if (done) {
            return
        }
        done = true
        reject.call(self, error)
    }
}
```
我们知道对于某个`Promise`实例而言这个状态一旦改变就不能再变化，也就是`resolve, reject`只能执行一次，所以我们需要一个标识符来标识一下，这就是`done`
当然这个`fn`可能会抛出异常，这时候是不能影响外部代码执行的，也就是需要达到异常被吃掉的效果，所以需要`try/catch`
值得注意的是`Promise`的异常其实有别于真正的异常，它其实是自己划分的一种状态，只要这个状态就是`rejected`，然后就可以被`catch`或者`then`的第二参数`onRejected`捕获，当然这个真正的异常必然就是`rejected`
## resolve
这个其实是`pending`转`fulfilled`的方法
```js
function resolve(nVal) {
    this._state = 1
    this._value = nVal
}
```
可见很简单，把`_state`修正下，然后保存下这个`_value`值
## reject
```js
function reject(reason) {
    this._state = 2
    this._value = reason
}
```
这个其实类似于`resolve`
## Promise.prototype.then、handle
这个其实就是用于捕获转变了`fulfilled, rejected`状态之后传递的数据，算是回调，有俩个参数都是函数，函数的参数就是对应的终值、拒因
```js
then(onFulfilled, onRejected) {
    handle(this, {
        onFulfilled, onRejected
    })
}
function handle(self, handler) {
    Promise._immediateFn(() => {
        const cb = self._state === 1 ? handler.onFulfilled : handler.onRejected
        let ret
        try {
            ret = cb(self._value)
        } catch (error) {
            
        }
    })
}
```
这个其实也可以看得出来很简单的调用`Promise._immediateFn`（异步的方法兼容）让这个`onFulfilled, onRejected`在下一个`tick`执行，至于执行哪个就看状态，这就是`then`的原理
也可以看出来其实通过实例所需数据一应俱全

## 缺陷
根据这个使用实例其实就可以很简单的写出这些实现逻辑，但很显然问题也是很多的：
1. `resolve`延迟调用
2. `then`得返回新`Promise`实例且能链式调用
3. 缺少`catch`方法
4. 终值不能是当前`Promise`实例本身
5. 得兼容终值是`Promise`实例、`thenable`情况

# Step2
上个版本有诸多缺陷，比如下面这个就不能满足，所以我们需要改进
```js
const promise = new Promise((resolve) => {
    setTimeout(() => {
        resolve(1)
    }, 1000)
})

promise.then((val) => {
    console.log(1, val)
    return val + 1
}).then((val) => {
    console.log(2, val)
    return val
})
```
若是`resolve`可以异步执行。这样子流程就变成`Promise.prototype.constructor  -->  doResolve  -->  then  -->  resolve`，而我们接收终值、拒因的`onFulfilled, onRejected`是在`then`阶段，也就是这时终值和拒因还没有准备好，这样子就需要对现有的实现做下调整
首先就是`onFulfilled, onRejected`的存储，我们需要把它存在`Promise`实例上，这样子我们可以在一段时间之后还能取到这个`then`上传入的函数

```js
constructor(fn) {
    this._state = 0
    this._value = undefined
    this._deferreds = []
    doResolve(this, fn)
}
```
这个`_deferreds`就是干的这个事。为什么是数组呢？其实是为了适应同一个实例多次调用`then`（非链式调用）
接着就是`then, handle`方法
```js
then(onFulfilled, onRejected) {
    const promise = new Promise(noop)
    handle(this, {
        onFulfilled,
        onRejected,
        promise
    })
    return promise
}
```
首先就是这个`then`方法得返回一个全新的`Promise`实例，这里不能返回自身，因为一旦状态变了就不能再变了，所以需要全新的(不然上一个`then`抛出异常之后也不能捕获了)，它不需要干什么事，只是一个承载`_value, _state`等数据的载体，所以传入`noop`
这里把`promise, onFulfilled, onRejected`三个一起打包给了`handle`
```js
function handle(self, handler) {
    if (self._state === 0) {
        self._deferreds.push(handler)
        return
    }
    // ...
}
```
这里就是核心处理，状态为`0`的时候说明终值、拒因还没准备好，这时候我把这个`promise, onFulfilled, onRejected`存入`_deferreds`
那么这个`_deferreds`什么时候触发呢？
很明显得`resolve`被调用的时候触发
```js
function resolve(nVal) {
    this._state = 1
    this._value = nVal
    final.call(this)
}
function final() {
    let handler
    while (handler = this._deferreds.shift()) {
        handle(this, handler)
    }
}
```
可见一旦这个`resolve`被调用我就遍历`_deferreds`，使用`handle`来处理，所以还需要对`handle`进一步改进
```js
function handle(self, handler) {
    if (self._state === 0) {
        self._deferreds.push(handler)
        return
    }
    Promise._immediateFn(() => {
        const {
            _state,
            _value
        } = self
        const { onFulfilled, onRejected, promise } = handler
        const cb = _state === 1 ? onFulfilled : onRejected
        let ret
        try {
            ret = cb(_value)
        } catch (error) {
            
        }
        resolve.call(promise, ret)
    })
}
```
从这个`handler`我们可以取到`onFulfilled, onRejected, promise`。我们先把`onFulfilled, onRejected`的返回值存储下来，这个得用`try/catch`来包裹`cb`的执行，因为这个是用户传入的不可控，若是没报错的话就会调用`resolve.call(promise, ret)`。这个其实是给`promise`这个实例填充状态、数值而已，当然若是之后没有调用`then, catch`(也就是没有注册`onFulfilled, onRejected`，这个也就没啥用，但是我数据还是得先准备好)

## 解决的缺陷
该版本解决了以下缺陷：
1. `resolve`延迟调用
2. `then`得返回新`Promise`实例且能链式调用

# Step3
我们继续改进
```js
const promise = new Promise((resolve) => {
    setTimeout(() => {
        resolve(1)
    }, 1000)
})

promise.then((val) => {
    console.log(1, val)
    throw new Error('catch')
}).catch((err) => {
    console.error(2, err)
    return err
}).then((val) => {
    console.log(3, val)
})
```
其实`catch`是`then`的一个简要写法`catch(onRejected)  <==>  then(null, onRejected)`
```js
catch(onRejected) {
    return this.then(null, onRejected)
}
```
所以首先就是调用`then`方法，注意得返回，因为`catch`也是支持链式
然后就是`reject`方法，我们之前只是改变了它的状态，并没有处理回调
```js
function reject(reason) {
    this._state = 2
    this._value = reason
    final.call(this)
}
```
简单的调用`final`方法来处理即可，这就是`final`抽出来的原因，可复用
这时候还有一点需要处理，就像这个例子，`then().catch().then()`，第一个`then`抛出了异常，那么这个`onFulfilled`执行就会报错
```js
function handle(self, handler) {
    // ... 
    Promise._immediateFn(() => {
        const {
            _state,
            _value
        } = self
        const { onFulfilled, onRejected, promise } = handler
        const cb = _state === 1 ? onFulfilled : onRejected
        let ret
        try {
            ret = cb(_value)
        } catch (error) {
            reject.call(promise, error)
            return
        }
        resolve.call(promise, ret)
    })
}
```
其实就是这里的`cb`会报错，这时候会进入`catch`，这时候我就需要调用`reject`方法来处理这个`error`拒因，并且需要`return`

> 其实到这里可以很明显的感知到`reject, resolve`其实就是来转换状态然后处理已经存入到`_deferreds`数组里的回调

## 解决的缺陷
该版本解决了以下缺陷：

1. 缺少`catch`方法

# Step4
我们继续改进
```js
const promise = new Promise((resolve) => {
    setTimeout(() => {
        resolve(promise)
    }, 1000)
})

promise.then((val) => {
    console.log(1, val)
    throw new Error('catch')
}).catch((err) => {
    console.error(2, err)
    return err
}).then((val) => {
    console.log(3, val)
})
```
这个实例把`Promise`实例自身当做终值传递给`resolve`，这个是不可以的，会导致无限循环的错误。为什么呢？我们知道若是终值是`Promise`实例的话这个自身的状态就会失效改用这个实例的状态，这样子就自身失效又用自身状态就自相矛盾了
首先肯定是`resolve`方法的改进，在这里终值已经准备好了，所以我们就可以判断了
```js
function resolve(nVal) {
    try {
        if(nVal === this) {
            throw new TypeError('Cannot be resolved with itself!')
        }
        this._state = 1
        this._value = nVal
        final.call(this)
    } catch (error) {
        reject.call(this, error)
    }
}
```
这里用了`try/catch`，这是因为根据规范终值是自身实例的话需要抛出`TypeError`，这样子就需要捕获了，因为需要吃掉这个异常，改用`catch`来暴露到外部
还有个隐蔽点需要注意，就像这个例子`then().catch().then()`，因为`resolve(promise)`导致`reject.call(this, error)`，而这个`this`只调用了第一个`then`，它没有传递第二参数`onRejected`。这样子的话这个`then`就需要跳过往下继续直到`catch`或者`then(null, onRejected)`，所以需要做下处理
```js
function handle(self, handler) {
    // ...
    Promise._immediateFn(() => {
        const { _state, _value } = self
        const { onFulfilled, onRejected, promise } = handler
        const cb = _state === 1 ? onFulfilled : onRejected
        if(cb == null) {
            (_state === 1 ? resolve : reject).call(promise, _value)
            return
        }
        let ret
        try {
            ret = cb(_value)
        } catch (error) {
            reject.call(promise, error)
            return
        }
        resolve.call(promise, ret)
    })
}
```
这里判断了下`cb`是否存在，不存在的话就判断下`_state`来调用`reject\resolve`，注意传入的得是`promise`，也就是到下一个链式方法了，本方法被跳过了
## 解决的缺陷
该版本解决了以下缺陷：

1. 终值不能是当前`Promise`实例本身

# Step5
我们继续改进
```js
const prom = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve(1)
    }, 3000)
})
const promise = new Promise((resolve) => {
    setTimeout(() => {
        resolve(prom)
    }, 0)
})

promise.then(1).catch(err => {
    console.error(2, err)
}).then(val => {
    console.log(3, val)
})

const thenable = {
    a: 'aaa',
    b: 'bbb',
    then(resolve, reject) {
        resolve(this.a)
    }
}
const promise2 = new Promise((resolve) => {
    resolve(thenable)
})

promise2.then(val => {
    console.log(1, val)
})
```
## `then、catch`参数非对象
这个其实和`Step4`的那个跳过`then`的情况很像，也就是我们只要把这个传入的参数处理下，只要不是函数我就置为`null`，这样子就能走入`cb == null`的这个判断
```js
then(onFulfilled, onRejected) {
    const promise = new Promise(noop)
    handle(this, {
        onFulfilled: typeof onFulfilled === 'function' ? onFulfilled : null,
        onRejected: typeof onRejected === 'function' ? onFulfilled : null,
        promise
    })
    return promise
}
```
简单处理即可
## 终值是`Promise`实例、`thenable`
这个很明显得在`resolve`处理
```js
function resolve(nVal) {
    try {
        if(nVal === this) {
            throw new TypeError('Cannot be resolved with itself!')
        }
        if (nVal && nVal.then) {
            const then = nVal.then
            if (nVal instanceof Promise) {
                this._state = 3
                this._value = nVal
                final.call(this)
            } else if(typeof then === 'function') {
                doResolve(this, then.bind(nVal))
                return
            }
        }
        this._state = 1
        this._value = nVal
        final.call(this)
    } catch (error) {
        reject.call(this, error)
    }
}
```
这里就简单的判断下这个终值是不是带有`then`属性，因为无论`thenable`还是`Promise`实例都有`then`属性，若是符合的话就具体判断下是那种情况：
### `Promise`实例
这个就是得状态转移了，就像这个例子，`promise`会丢弃自己的状态等待`prom`状态改变了，也就是`prom`状态决定了`promise`状态
```js
if (nVal instanceof Promise) {
    this._state = 3
    this._value = nVal
    final.call(this)
}
```
这里就是引入了新的`_state = 3`，这个就代表终值是`Promise`实例的情况，这个就需要等待状态改变了
```js
function handle(self, handler) {
    while (self._state === 3) {
        self = self._value
    }
    if (self._state === 0) {
        self._deferreds.push(handler)
        return
    }
    // ....
}
```
这句就是，开启`while`，要是状态是`3`就将当前`Promise`实例对象赋值成这个终值（也是`Promise`实例对象）
就像例子里的`promise`的`resolve`执行之后就会到这里，而这个`handler`就是`promise`对应的在此例就是`then(1)`（`{ onFulfilled: null, onRejected: null, promise: xxx }`）
这时候`_state`必然是`3`，那么`self = prom`，而`prom._state === 0`（延迟三秒），所以这个本该是`promise`的`handler`给转到了`prom`，这样子就完成了`promise`的`handler`转给`prom`
等到`prom`的`resolve`执行之后，自然就能使用原本`promise`的`handler`去处理`prom`的终值
### `thenable`
这个按照规范就是如果这个对象的`then`是函数的话就会将这个对象作为`this`去调用这个函数，而且会传俩个参数`resolvePromise, rejectPromise`。这个其实就是`new Promise(fn)`里的`fn`
所以解决也就简单了
```js
if(typeof then === 'function') {
    doResolve(this, then.bind(nVal))
    return
}
```
其实这里关键就是把`doResolve`的`fn`给替换成了`thenable.then`而已
## 解决的缺陷

该版本解决了以下缺陷：
1. 得兼容终值是`Promise`实例、`thenable`情况
2. `then、catch`参数不是对象得跳过

# Step6

这里其实就新增几个常用的方法，一一讲解，它们非`Promise A+`规范内

## resolve

这个就是把给的对象转成`Promise`对象且状态是`fulfilled`

其实就是`new Promise()`使用

```js
static resolve(val) {
    if (val && typeof val === 'object' && val instanceof Promise) {
        return val
    }
    return new Promise((resolve) => {
        resolve(val)
    })
}
```

首先判断下是不是`Promise`实例，是的话就直接返回就是了

## reject

```js
static reject(reason) {
    return new Promise((resolve, reject) => {
        reject(reason)
    })
}
```

和`resolve`基本一致

## race

```js
static race(promises) {
    return new Promise((resolve, reject) => {
        promises.forEach(promise => {
            Promise.resolve(promise).then(resolve, reject)
        })
    })
}
```

这个思路其实很简单，我返回一个新的`Promise`对象，在其内部遍历传入的`promise`数组，一旦有一个状态变了就调用这个`Promise`对象的`resolve\reject`去决策

## all

```js
static all(promises) {
    let len = promises.length
    return new Promise((resolve, reject) => {
        promises.forEach((promise, i) => {
            Promise.resolve(promise).then(res => {
                promises[i] = res
                if (--len === 0) {
                    resolve(promises)
                }
            }).catch(reason => {
                reject(reason)
            })
        })
    })
}
```

这个其实和`race`类似，不一样的是我需要判断下这个`resolve`调用时机以及处理下这个终值

## finally

```js
finally(cb) {
    const constructor = this.constructor;
    return this.then(
        value => {
            return constructor.resolve(cb()).then(() => value)
        },
        reason => {
            return constructor.resolve(cb()).then(() => {
                throw reason
            })
        }
    )
}
```

这里其实疑点很多，一一详解

### 缓存constructor

嗯，其实实现里我都是直接用`Promise`，没有用`this.constructor`来取构造，其实取构造更好些。因为`finally`它并不是`Promise A+`规范，这样子一些第三方库没有实现，这个补丁打上去的话取构造还是更好点

### Why not `.then(f, f)`?

这个其实很多人会想为什么不是下面这种写法。可以先看下[Why not `.then(f, f)`?](https://github.com/tc39/proposal-promise-finally#why-not-thenf-f)

```js
finally(callback) {
    return this.then(
        value => {
            callback()
            return value
        },
        err => {
            callback()
            return err
        }
    )
}
```

其实这种写法会有问题

```js
const p3 = new Promise((resolve, reject) => {
    setTimeout(() => {
        reject(1)
    }, 3000)
})
Promise.resolve(2).finally(() => {
    console.log(1, 'finally')
    return p3
}).then(val => {
    console.log(2, val)
}).catch(reason => {
    console.error(3, reason)
})
```

如此例，首先输出`1 "finally"`，然后等待`3s`输出`3 1`。这个是正确的表现，但是要是如上面这种实现就会有问题，也就是会立即输出`1 "finally"`以及`2 2`

所以我需要用`Promise.resolve(cb())`处理下这个返回值，这样子就相当于如下所示

```js
Promise.resolve(2).then(
    value => {
        return p3.then(() => value)
    },
    reason => {
        return p3.then(() => {
            throw reason
        })
    }
).then(val => {
    console.log(2, val)
}).catch(reason => {
    console.error(3, reason)
})
```

这个看起来就很简单了，第一个`then`的终值是`Promise`实例`p3`，这样子后续的`then, catch`就会看这个`p3`了

## 解决的缺陷

1. 新增`resolve, reject, race, all`静态方法
2. 新增`finally`实例方法

# Promise A+规范测试

这个就是用`promises-aplus-tests`测试库来测下这个实现

```js
const polyfill = {
    deferred: function () {
        var obj = {}
        var prom = new Promise(function (resolve, reject) {
            obj.resolve = resolve
            obj.reject = reject
        })
        obj.promise = prom
        return obj
    }
}
Promise.deferred = polyfill.deferred
```

把这段加上就是了