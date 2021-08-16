# 内置类型
JS有七种内置类型：null、undefined、number、boolean、string、object、symbol。**除了object，其它都是基本类型**
typeof的结果都一一对应，除了null，这个是个bug。不过应该不会修改，毕竟改了的话多少系统都有问题了
这里要提的主要是function。函数对象typeof结果是function，不过并不代表它是内置类型，它只是object的一个子类型。
>function对象拥有内部属性[[Call]]，所以可被调用。还有一个length属性，是参数个数
# 值和类型
得分清楚变量和值。**变量是没有类型的，值才有类型**。
```javascript
let a = 3
typeof a // number
a = true
typeof a // boolean
```
1. undefined和undeclared
变量未持有值时为undefined。不过若是typeof一个未声明的变量结果也是undefined，这个若是返回undeclared可能会好点
2. typeof Undeclared
这个其实可能会觉得会报ReferenceError，其实并不会，因为typeof有个安全防范机制。
这个机制还是挺有用的，比如我们有个全局变量DEBUG用于是否打开调试。这个变量在debug.js里声明，不过只在开发模式加载，那么生产环境下就取不到这个变量```if(DEBUG) {}```，这样子是会报错的，这时候我们就可以用typeof，当然也可以```if(window.DEBUG) {}```

>总而言之，通过typeof来检查undeclared变量有时候是个不错的选择