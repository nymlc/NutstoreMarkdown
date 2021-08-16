`RuleSet`是用于解析`module.rules`这个配置项的，它会在创建`module`的时候告诉你加载当前这个模块需要哪些`loader`，如图所示是我们自定义的`rules`，它也有默认的`rules`，这也是我们什么都不配也能打包`js`文件的原因

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1608196141431.png" alt="image-20201217170851357" style="zoom:50%;" />

```js
// options.defaultRules就是默认的rules
/**
options.defaultRules
[
    {
        type: "javascript/auto",
        resolve: {}
    },
    {
        test: /\.mjs$/i,
        type: "javascript/esm",
        resolve: {
            mainFields: ['browser', 'main']
        }
    },
    {
        test: /\.json$/i,
        type: "json"
    },
    {
        test: /\.wasm$/i,
        type: "webassembly/experimental"
    }
]
*/
this.ruleSet = new RuleSet(options.defaultRules.concat(options.rules));
```
从代码可见，传入的`rules`就是默认的和自定义的结合

# 设计

这个类其实就是个工具类，很独立，输入一个`rules`得到需要的结果即可，所以我们可以从输入、输出来看下它的设计以及如何实现的

```js
// 输入的配置项
input = [
    {
        type: "javascript/auto",
        resolve: {}
    },
    {
        test: /\.mjs$/i,
        type: "javascript/esm",
        resolve: {
            mainFields: ['browser', 'main']
        }
    },
    {
        test: /\.json$/i,
        type: "json"
    },
    {
        test: /\.wasm$/i,
        type: "webassembly/experimental"
    },
    {
        test: /.js$/,
        loader: "babel-loader"
    },
    {
        test: /.vue$/,
        loader: "vue-loader"
    },
    {
        test: /.scss$/,
        use: [
            "vue-style-loader",
            "css-loader",
            {
                loader: "sass-loader",
                options: {
                    data: "$color: red;"
                }
            }
        ]
    }
]
// 处理过的配置项
output = {
    references: {
        "ref--6-2": {
            data: "$color: red;"
        }
    },
    rules: [
        {
            type: "javascript/auto",
            resolve: {}
        },
        {
            resource: /\.mjs$/i.test.bind(/\.mjs$/i),
            type: "javascript/esm",
            resolve: {
                mainFields: ["browser", "main"]
            }
        },
        {
            resource: /\.json$/i.test.bind(/\.json$/i),
            type: "json"
        },
        {
            resource: /\.wasm$/i.test.bind(/\.wasm$/i),
            type: "webassembly/experimental"
        },
        {
            resource: /.js$/.test.bind(/.js$/),
            use: [
                {
                    loader: "babel-loader"
                }
            ]
        },
        {
            resource: /.vue$/.test.bind(/.vue$/),
            use: [
                {
                    loader: "vue-loader"
                }
            ]
        },
        {
            resource: /.scss$/.test.bind(/.scss$/),
            use: [
                {
                    loader: "vue-style-loader"
                },
                {
                    loader: "css-loader"
                },
                {
                    options: {
                        data: "$color: red;"
                    },
                    ident: "ref--6-2",
                    loader: "sass-loader"
                }
            ]
        }
    ]
}
// 根据当前的模块过滤出的所需的配置项，这里test/include/exclude/resource之类的配置已经没用了，因为这些配置就是用于判定过滤的，过滤完了自然就没用了
exec = [
    {
        type: "type",
        value: "javascript/auto"
    },
    {
        type: "reslove",
        value: {}
    },
    {
        type: "use",
        value: { 
            loader: "babel-loader"
        }
    }
]
```

可见大体上俩者还是很相像的，首先输出是个对象，有两个属性：`references、rules`，如下

```js
{
    references: {
        "ref--6-2": {
            data: "$color: red;"
        }
    },
    rules: []
}
```

`references`可见就是用于方便查询获取`loader`附带的`options`，`rules`的子项也多了个`resource`，可以把它看做`test`的一个处理值，更方便之后的过滤

现在任务就是怎么实现两者的转化，要知道这个配置可远远不止栗子所示这么简单，具体配置可见官网

# 实现

如图所示，从结构来说还是挺简单的

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1608207587872.png" alt="image-20201217201942997" style="zoom:50%;" />

```js

class RuleSet {
    constructor(rules) {
        this.references = Object.create(null);
        this.rules = RuleSet.normalizeRules(rules, this.references, "ref-");
    }

	static normalizeRules(rules, refs, ident) {
      if (Array.isArray(rules)) {
          return rules.map((rule, idx) => {
              return RuleSet.normalizeRule(rule, refs, `${ident}-${idx}`);
          });
      } else if (rules) {
        	return [RuleSet.normalizeRule(rules, refs, ident)];
      } else {
        	return [];
      }
    }
}
```

首先就是调用`normalizeRules`方法来格式化传入的`rules`，此时也传入了定义过的`references`

`normalizeRules`其实也很简单，它就是区分下`rules`的类型，若是数组的话就遍历处理子项，最终结果得返回一个数组类型

## normalizeRule

这里其实是一个错误检测、子项处理分发入口，也就是具体的处理都是分发到其它分发去处理了，以下段段分析

```js
/*	webpack.config.js
			module: {
				rules: ['babel-loader']
			}
		*/
if (typeof rule === "string") {
    return {
        use: [
            {
                loader: rule
            }
        ]
    };
}
if (!rule) {
    throw new Error("Unexcepted null when object was expected as rule");
}
// 也就是要么是字符串，要么就得是对象
if (typeof rule !== "object") {
    throw new Error(
        "Unexcepted " +
            typeof rule +
            " when object was expected as rule (" +
            rule +
            ")"
    );
}
```

首先这是判定下`rule`如注释所示配置为字符串的时候就可以直接返回了，然后要是`rule`不存在那么肯定有问题的，比如`rules:['']`，最后`rule`不是字符串就必须得是对象了（数组也是`object`）

### checkUseSource/checkResourceSource

```js
const newRule = {};
let useSource;
let resourceSource;
let condition;
// 检测是否提供了多个结果源，就是不能loader/loaders和use共存
const checkUseSource = newSource => {
    if (useSource && useSource !== newSource) {
        throw new Error(
            RuleSet.buildErrorMessage(
                rule,
                new Error(
                    "Rule can only have one result source (provided " +
                    newSource +
                    " and " +
                    useSource +
                    ")"
                )
            )
        );
    }
    useSource = newSource;
};

// 用于判断resource重复的（test/include/exclude和resource只能二选一）
const checkResourceSource = newSource => {
    if (resourceSource && resourceSource !== newSource) {
        throw new Error(
            RuleSet.buildErrorMessage(
                rule,
                new Error(
                    "Rule can only have one resource source (provided " +
                    newSource +
                    " and " +
                    resourceSource +
                    ")"
                )
            )
        );
    }
    resourceSource = newSource;
};
```

这两个就是工具函数，用于检测是否有提供重复的配置，即：

+ loader/loaders和use不能共存

+ test/include/exclude和resource不能共存

  <img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1608540273159.png" alt="image-20201221164420971" style="zoom:50%;" />

```js
// 这三个其实就是resource的简写，所以可以拼接成resource去处理
if (rule.test || rule.include || rule.exclude) {
    checkResourceSource("test + include + exclude");
    condition = {
        test: rule.test,
        include: rule.include,
        exclude: rule.exclude
    };
    try {
        newRule.resource = RuleSet.normalizeCondition(condition);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(condition, error));
    }
}
// 处理resource
if (rule.resource) {
    checkResourceSource("resource");
    try {
        newRule.resource = RuleSet.normalizeCondition(rule.resource);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(rule.resource, error));
    }
}

// 处理realResource
if (rule.realResource) {
    try {
        newRule.realResource = RuleSet.normalizeCondition(rule.realResource);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(rule.realResource, error));
    }
}

// 处理resourceQuery
if (rule.resourceQuery) {
    try {
        newRule.resourceQuery = RuleSet.normalizeCondition(rule.resourceQuery);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(rule.resourceQuery, error));
    }
}

// 处理compiler
if (rule.compiler) {
    try {
        newRule.compiler = RuleSet.normalizeCondition(rule.compiler);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(rule.compiler, error));
    }
}

// 处理issuer
if (rule.issuer) {
    try {
        newRule.issuer = RuleSet.normalizeCondition(rule.issuer);
    } catch (error) {
        throw new Error(RuleSet.buildErrorMessage(rule.issuer, error));
    }
}
// 不能同时提供这两，其实loaders已经废弃了，不用考虑
if (rule.loader && rule.loaders) {
    throw new Error(
        RuleSet.buildErrorMessage(
            rule,
            new Error(
                "Provided loader and loaders for rule (use only one of them)"
            )
        )
    );
}
```

这里其实就是分发了，就是把这么些条件相关的配置都分发到`normalizeCondition`去处理，顺带做了些简单的错误判定

```js
const loader = rule.loaders || rule.loader;
if (typeof loader === "string" && !rule.options && !rule.query) {
    // inline loader，所以没有options、query
    checkUseSource("loader");
    newRule.use = RuleSet.normalizeUse(loader.split("!"), ident);
} else if (typeof loader === "string" && (rule.options || rule.query)) {
    // 处理下use，由此可见use由loader/options/query，这两个相等
    checkUseSource("loader + options/query");
    newRule.use = RuleSet.normalizeUse(
        {
            loader: loader,
            options: rule.options,
            query: rule.query
        },
        ident
    );
} else if (loader && (rule.options || rule.query)) {
    // loader存在却不是字符串，而且options或者query也在，那么options/query就不生效
    throw new Error(
        RuleSet.buildErrorMessage(
            rule,
            new Error(
                "options/query cannot be used with loaders (use options for each array item)"
            )
        )
    );
} else if (loader) {
    checkUseSource("loaders");
    newRule.use = RuleSet.normalizeUse(loader, ident);
} else if (rule.options || rule.query) {
    // 提供了options/query，却没有提供loader明显是不行的
    throw new Error(
        RuleSet.buildErrorMessage(
            rule,
            new Error(
                "options/query provided without loader (use loader + options)"
            )
        )
    );
}
```

这里就是处理`loader`，这个就区分`inline loader`和普通`loader`。最终结果就是把它们都转成`use`形式，因为这两者是一致的，而`use`是对象，可配性更高了，也是为了统一

代码可见其实也是分发到`normalizeUse`去的，也做了些重复配置判定等一些错误提示

```js
// 有了use就不用上面处理了
if (rule.use) {
    checkUseSource("use");
    newRule.use = RuleSet.normalizeUse(rule.use, ident);
}

// 有了rules就遍历处理子项
if (rule.rules) {
    newRule.rules = RuleSet.normalizeRules(
        rule.rules,
        refs,
        `${ident}-rules`
    );
}
// 处理oneOf
if (rule.oneOf) {
    newRule.oneOf = RuleSet.normalizeRules(
        rule.oneOf,
        refs,
        `${ident}-oneOf`
    );
}
```

这里就是简单了，首先`use`若是提供了就得判定下是否提供了`loader/loaders`（调用`checkUseSource`）

`rules`的话就是遍历处理子项即可（就是调用`normalizeRules`），`oneOf`的话就是第一个匹配上的生效，和`rules`一样处理了（具体差异具体方法会处理）

```js
// 到这rule必然是对象，把有效的属性摘出来，然后赋到newRule返回值即可
const keys = Object.keys(rule).filter(key => {
    return ![
        "resource",
        "resourceQuery",
        "compiler",
        "test",
        "include",
        "exclude",
        "issuer",
        "loader",
        "options",
        "query",
        "loaders",
        "use",
        "rules",
        "oneOf"
    ].includes(key);
});
for (const key of keys) {
    newRule[key] = rule[key];
}

// 这里处理下options数据，放进references，好查询
if (Array.isArray(newRule.use)) {
    for (const item of newRule.use) {
        if (item.ident) {
            refs[item.ident] = item.options;
        }
    }
}

return newRule;
```

首先是构建`keys`了，也就是把判定这个模块是否匹配当前的规则所需的属性全部剔除（因为这些属性是用来构建条件去判定是否匹配的，构建完了自然就没用了），剩下的属性（`use`处理过了，所以不用处理），然后把这些属性赋值给`newRule`等待返回

然后处理下`options`数据，这个是把它放进`references`，这样子之后取值方便（`.ident`在`normalizeUseItem`里设置了）

最后返回`newRule`即可

## normalizeCondition

这个其实就是`resource/realResource/resourceQuery/compiler/issuer`等字段的处理方法，可见它就是返回一个接受模块路径的函数，这样子就可以很方便的根据路径来判断当前模块是否需要使用该条件对应的`loader`

```js
if (!condition) throw new Error("Expected condition but got falsy value");
// 若是字符串的话，只需要路径包含该字符串即可
if (typeof condition === "string") {
    return str => str.indexOf(condition) === 0;
}
// 若是函数的话就直接返回
if (typeof condition === "function") {
    return condition;
}
// 正则的话就返回.test
if (condition instanceof RegExp) {
    return condition.test.bind(condition);
}
// 数组的话就就只要一个子项匹配就行
if (Array.isArray(condition)) {
    const items = condition.map(c => RuleSet.normalizeCondition(c));
    return orMatcher(items);
}
// 若之上都没进入，还不是对象，说明这个配置有问题
if (typeof condition !== "object") {
    throw Error(
        "Unexcepted " +
        typeof condition +
        " when condition was expected (" +
        condition +
        ")"
    );
}
```

先是简单的判空，然后就是一系列的简单类型的处理，比如字符串的话就返回`indexOf`的包裹函数等

值得注意的是数组的话仅需要一个子项匹配即可，所以得用上`orMatcher`方法处理

最后做个简单的错误判定

```js
// matchers其实就是收集条件对象子项，需要每一项都符合
const matchers = [];
Object.keys(condition).forEach(key => {
    const value = condition[key];
    switch (key) {
        // 这三个就是匹配一个项即可
        case "or":
        case "include":
        case "test":
            if (value) matchers.push(RuleSet.normalizeCondition(value));
            break;
        // and就需要所有子项都匹配
        case "and":
            if (value) {
                const items = value.map(c => RuleSet.normalizeCondition(c));
                matchers.push(andMatcher(items));
            }
            break;
        // 这俩个得不匹配，所以使用notMatcher
        case "not":
        case "exclude":
            if (value) {
                const matcher = RuleSet.normalizeCondition(value);
                matchers.push(notMatcher(matcher));
            }
            break;
        default:
            throw new Error("Unexcepted property " + key + " in condition");
    }
});
// 一般而言这里至少有一项
if (matchers.length === 0) {
    throw new Error("Excepted condition but got " + condition);
}
// 每一项都被处理成了函数，只有一项的话只需要直接返回该子项即可
if (matchers.length === 1) {
    return matchers[0];
}
// 每一项都得匹配上
return andMatcher(matchers);
```

传入的`condition`若是对象的话是形如`{ test: /.js/, include: [/src/], exclude: [/node_modules/] }`，这是得每个属性都匹配上，所以得遍历属性，根据情况来做处理，这里采用的是比较巧妙的法子，可以借鉴

首先他也是把各种情况划分，如代码所示分为三部分，然后按各自情况构建条件判定函数且`push`到数组，这样子就可以根据该数组`matchers`来处理了，也就是使用`andMatcher`来返回一个判定函数，即该数组每一项都得匹配上

### notMatcher/orMatcher/andMatcher

```js
// 传入的路径不匹配即可
const notMatcher = matcher => {
    return str => {
        return !matcher(str);
    };
};

// 传入的数组所有子项有一项匹配即可
const orMatcher = items => {
    return str => {
        for (let i = 0; i < items.length; i++) {
            if (items[i](str)) return true;
        }
        return false;
    };
};

// 传入的数组所有子项都得匹配
const andMatcher = items => {
    return str => {
        for (let i = 0; i < items.length; i++) {
            if (!items[i](str)) return false;
        }
        return true;
    };
};
```

这三个工具方法逻辑很简单，扫一眼即可

## normalizeUse**

这三个顾名思义就是处理`Rule.use`的

### normalizeUse

首先自然是`normalizeUse`

```js
// 格式化use
static normalizeUse(use, ident) {
    // 若是函数的话就用该函数处理传入的data
    // 其实最终结果都是处理成如下样子
    /**
     [{
        options: {
            data: "$color: red;"
        },
        ident: "ref--6-2",
        loader: "sass-loader"
    }]
     */
    if (typeof use === "function") {
        return data => RuleSet.normalizeUse(use(data), ident);
    }
    if (Array.isArray(use)) {
        return use
            .map((item, idx) => RuleSet.normalizeUse(item, `${ident}-${idx}`))
            .reduce((arr, items) => arr.concat(items), []);
    }
    return [RuleSet.normalizeUseItem(use, ident)];
}
```

我们得知道我们配置的`use`一般都是下图这样子

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1608552252416.png" alt="image-20201221200406491" style="zoom:50%;" />

> 首先若是`use`是函数，那么就直接返回一个函数，该函数返回一个`RuleSet.normalizeUse`的结果值，传入的参数就是`use`函数的结果值。这样子处理就是动态处理，也就是在运行时`use`函数可以拿到当前的模块信息，从而做出一些处理，动态配置`Rule.use`

若是数组的话就是遍历调用自身处理，`.map`这段就是遍历调用自身（得到的结果是二维数组），`.reduce`就是把`.map`的结果拍平成一维数组

### normalizeUseItem

若是既不是数组，也不是函数，那么调用`normalizeUseItem`

```js
// 处理use 子项
static normalizeUseItem(item, ident) {
    // 字符串就可以直接格式化得到最终值
    if (typeof item === "string") {
        return RuleSet.normalizeUseItemString(item);
    }

    const newItem = {};
    // 这两不能同时提供
    if (item.options && item.query) {
        throw new Error("Provided options and query in use");
    }
    // 到这其实就代表该use子项是对象了，那就得有loader属性
    if (!item.loader) {
        throw new Error("No loader specified");
    }

    newItem.options = item.options || item.query;
    // 简单赋值ident，方便之后提取options
    if (typeof newItem.options === "object" && newItem.options) {
        if (newItem.options.ident) {
            newItem.ident = newItem.options.ident;
        } else {
            newItem.ident = ident;
        }
    }
    // 这里就是把options、query这两属性之外的属性赋给newItem返回值，因为这两个上面已经处理过了
    const keys = Object.keys(item).filter(function (key) {
        return !["options", "query"].includes(key);
    });

    for (const key of keys) {
        newItem[key] = item[key];
    }

    return newItem;
}
```

首先判断该子项是字符串的话就调用`normalizeUseItemString`处理

然后就是简单的一些错误配置的报错提示

再然后是处理`options`，这里主要是赋值`ident`，相当于唯一标识

最后就是把`options、query`之外的属性赋值给`newItem`给返回，因为这俩已经处理过了

### normalizeUseItemString

```js
// 处理带有querystring的loader
static normalizeUseItemString(useItemString) {
    const idx = useItemString.indexOf("?");
    if (idx >= 0) {
        return {
            loader: useItemString.substr(0, idx),
            options: useItemString.substr(idx + 1)
        };
    }
    return {
        loader: useItemString,
        options: undefined
    };
}
```

这个是处理带有`queryString`的`loader`，这里就是把`queryString`当做`options`，处理如下情况

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1608554255751.png" alt="image-20201221203730083" style="zoom:50%;" />

## exec

这个就是根据传入的模块信息过滤所需的`loader`配置

```js
// 这个是用于获取指定模块所匹配的loader。就像man.js就不需要sass-loader，即使配置了它
/**
 * data如下
 {
    "compiler": undefined,
    "resource": "/Users/nymlc/Documents/Nutstore Files/我的坚果云/基础知识/demo/简易实现webpack/webpack/test/message.js",
    "realResource": "/Users/nymlc/Documents/Nutstore Files/我的坚果云/基础知识/demo/简易实现webpack/webpack/test/message.js",
    "resourceQuery": "?type=test",
    "issuer": "/Users/nymlc/Documents/Nutstore Files/我的坚果云/基础知识/demo/简易实现webpack/webpack/test/entry.js"
}
 */
exec(data) {
    const result = [];
    this._run(
        data,
        {
            rules: this.rules
        },
        result
    );
    return result;
}
```

可见这里就是简单的接收下传入的`data`，实际工作都是交由`_run`执行的

### _run

这里就是实际上过滤配置的地方

```js

// result数组子项如下
/**
 {
    "type": "use",
    "value": {
        "options": {
            "data": "$color: red;"
        },
        "ident": "ref--6-2",
        "loader": "sass-loader"
    }
}
 */
_run(data, rule, result) {
    // test conditions
    // 这些if就是判断是否需要将对应的loader加入到result数组中
    if (rule.resource && !data.resource) return false;
    if (rule.realResource && !data.realResource) return false;
    if (rule.resourceQuery && !data.resourceQuery) return false;
    if (rule.compiler && !data.compiler) return false;
    if (rule.issuer && !data.issuer) return false;
    // 这个就是用于判断模块路径是否匹配当前rule了，rule.resource就是处理过的函数
    if (rule.resource && !rule.resource(data.resource)) return false;
    if (rule.realResource && !rule.realResource(data.realResource))
        return false;
    if (data.issuer && rule.issuer && !rule.issuer(data.issuer)) return false;
    if (
        data.resourceQuery &&
        rule.resourceQuery &&
        !rule.resourceQuery(data.resourceQuery)
    ) {
        return false;
    }
    if (data.compiler && rule.compiler && !rule.compiler(data.compiler)) {
        return false;
    }
    // ......
}
```

这些判断都是用于确定该模块是否匹配当前的规则，不匹配的话就`return false`

在此可见`rule.resource`给处理成函数的优势了，这也是`normalizeCondition`的作用了

```js
// 到这说明通过了校验，该rule对应的loader可以用于处理该模块
// 这里就是把rule里和判定条件无关的属性给筛选出来，然后加入到result里返回
// apply
const keys = Object.keys(rule).filter(key => {
    return ![
        "resource",
        "realResource",
        "resourceQuery",
        "compiler",
        "issuer",
        "rules",
        "oneOf",
        "use",
        "enforce"
    ].includes(key);
});
for (const key of keys) {
    result.push({
        type: key,
        value: rule[key]
    });
}
```

这就是常规操作了，把没处理的属性给塞进结果集里，待返回

```js
// 若是有use的属性话，就把里面的项给拆出来（可能是数组，也可能是字符串）
if (rule.use) {
    const process = use => {
        if (typeof use === "function") {
            process(use(data));
        } else if (Array.isArray(use)) {
            use.forEach(process);
        } else {
            result.push({
                type: "use",
                value: use,
                enforce: rule.enforce
            });
        }
    };
    process(rule.use);
}
```
这里处理`use`，还是有点东西的。这个就是得处理`use`是函数、数组、对象等情况
+ 函数的话就得先调用该函数得到对象，然后继续递归处理
+ 数组的话就得遍历处理
+ 其它的话就是直接可以`push`到结果集
```js
// 就是遍历rules，递归处理
if (rule.rules) {
    for (let i = 0; i < rule.rules.length; i++) {
        this._run(data, rule.rules[i], result);
    }
}
// 和rules基本一样，不过因为它只生效有效的第一项，所以有个break
if (rule.oneOf) {
    for (let i = 0; i < rule.oneOf.length; i++) {
        if (this._run(data, rule.oneOf[i], result)) break;
    }
}

return true;
```
这里就是处理`rules`和`oneOf`，这两个还是基本一致的，`rules`就是遍历处理，`oneOf`关键在于就第一项有效的生效，所以得对结果值判定，若是有效就`break`。最终`return true`，这个就是有效的判定
## findOptionsByIdent

这个就是简单的根据`ident`查询获取`options`
```js
// 因为references如下，所以根据ident可以很方便的获取对应的options
/**
{
    references: {
        "ref--6-2": {
            data: "$color: red;"
        }
    }
}
 */
findOptionsByIdent(ident) {
    const options = this.references[ident];
    if (!options) {
        throw new Error("Can't find options with ident '" + ident + "'");
    }
    return options;
}
```
这里就是做了个判空错误提示