我们词法、语法编译就是为了把这个模板编译成`AST`对象，所以我们其实在着手编译之前就应该知道这个对象长啥样，有什么属性，本章就是干这么件事

`AST`对象呢其实就是用来描述节点的，也就是这个节点长什么样子，拥有什么属性等等，这样子考虑的话然后配合现有的对象定义反推就简单多了

我们首先从`flow`看下这个`AST`对象的定义

```js
declare type ASTNode = ASTElement | ASTText | ASTExpression;
```

可见它有三种类型，我们一个个来。在此之前我们先说下区分它们三的一个属性`type`：

+ `1`：代表元素节点
+ `2`：代表带有表达式的文本节点
+ `3`：代表文本节点，包括注释节点

# ASTExpression

顾名思义就是包含表达式的文本节点对象，不要多想我们肯定需要标识这个类型(`type`)、存储这个表达式文本(`text`)、表达式的代码(`expression`)

```js
declare type ASTExpression = {
  type: 2;
  expression: string;
  text: string;
  tokens: Array<string | Object>;
  static?: boolean;
  // 2.4 ssr optimization
  ssrOptimizability?: number;
};
// {{width}}
/**
{
    "type": 2,
    "expression": "_s(width)",
    "tokens": [{
        "@binding": "width"
    }],
    "text": "{{width}}",
    "static": false
}
*/
```

## type

就是代表表达式元素节点

## expression

顾名思义就是表达式，这个是编译之后生成的代码。它也可以用于检测错误(`detectErrors`)

## text

就是表达是文本，这个其实就用来检测错误(`detectErrors`)，看看表达式符不符合规范

## tokens

这个其实是`weex`才用到，没学过，之后有机会看看有啥用

## static

标识是不是静态节点。不涉及到数据变化的静态节点每次渲染结果都是一样的，所以把这些标记出来就可以跳过这些节点的比对

## ssrOptimizability

顾名思义就是`ssr`的优化用到的



# ASTText

就是文本节点，也包括注释节点，这里只说独有的属性

```js
declare type ASTText = {
  type: 3;
  text: string;
  static?: boolean;
  isComment?: boolean;
  // 2.4 ssr optimization
  ssrOptimizability?: number;
};
```

## isComment

就是用来判断是不是注释，毕竟注释是`createComment`，一般的文本节点是`createTextNode`

# ASTElement

元素节点，这个就不一样了，它可是能带各种各样的属性，而且在`vue`里还能有各种指令，所以需要一一处理

```js
declare type ASTElement = {
  type: 1;
  tag: string;
  attrsList: Array<{ name: string; value: any }>;
  attrsMap: { [key: string]: any };
  parent: ASTElement | void;
  children: Array<ASTNode>;

  processed?: true;

  static?: boolean;
  staticRoot?: boolean;
  staticInFor?: boolean;
  staticProcessed?: boolean;
  hasBindings?: boolean;

  text?: string;
  attrs?: Array<{ name: string; value: any }>;
  props?: Array<{ name: string; value: string }>;
  plain?: boolean;
  pre?: true;
  ns?: string;

  component?: string;
  inlineTemplate?: true;
  transitionMode?: string | null;
  slotName?: ?string;
  slotTarget?: ?string;
  slotScope?: ?string;
  scopedSlots?: { [name: string]: ASTElement };

  ref?: string;
  refInFor?: boolean;

  if?: string;
  ifProcessed?: boolean;
  elseif?: string;
  else?: true;
  ifConditions?: ASTIfConditions;

  for?: string;
  forProcessed?: boolean;
  key?: string;
  alias?: string;
  iterator1?: string;
  iterator2?: string;

  staticClass?: string;
  classBinding?: string;
  staticStyle?: string;
  styleBinding?: string;
  events?: ASTElementHandlers;
  nativeEvents?: ASTElementHandlers;

  transition?: string | true;
  transitionOnAppear?: boolean;

  model?: {
    value: string;
    callback: string;
    expression: string;
  };

  directives?: Array<ASTDirective>;

  forbidden?: true;
  once?: true;
  onceProcessed?: boolean;
  wrapData?: (code: string) => string;
  wrapListeners?: (code: string) => string;

  // 2.4 ssr optimization
  ssrOptimizability?: number;

  // weex specific
  appendAsTree?: boolean;
};
```

这个其实`Vue`帮我们区分了，我们一步一步来看

## 基本属性

这个就是一个元素节点需要的基本信息：标签名、属性、父节点、子节点

### type

这个就节点类型

### tag

就标签名，我需要知道创建什么标签

### attrsList

属性列表，注意这里存储的是没处理的属性列表，比如`v-for`。这个在转化对应的属性时一般会被删掉，比如`v-for`会被转成`for`系列属性，之后详解

### attrsMap

这个其实是`attrsList`的键值对形式，方便取值，还有个很重要的就是我生成`ast`之后需要对这个`ast`进行检查有没有错误(`detectErrors`)，这时候我需要有这些属性，就落在这了。`attrsList`在处理一些属性和指令时会处理一个就删除一个，所以相对来说`attrsMap`基本上是模板上可见的属性集（在`src/platforms/web/compiler/modules/model.js`会有例外）

## v-if

这个如何设计`v-if`就得看下它的用法，我们知道它有三种`v-if, v-else-if, v-else`，前俩个带有表达式，后一个没有。所以我们可以用`if, elseif`来存储表达式这样子既存了表达式也表明了这个节点有`v-if, v-else-if`，`else`来存储`true`或者别的什么反正有值就成来表示这个节点是有`v-else`。

还有个很重要的就是我们知道这三个其实只能渲染一个，也就是三个其实可以看作是一个。这样子我们就不可以把这三个都存在这个`ast`上，所以`vue`做法就是把第一个也就是`v-if`这个元素节点对象发在这个`ast`上，然后这三个对应的元素节点对象都存在这个对象的`ifConditions`上，这样子我在生成代码时可以用三目运算符依据表达式来对应渲染

```html
<div v-if="1"></div>
<p v-else-if="2"></p>
<span v-else></span>
```
```json
{
    "type": 1,
    "tag": "div",
    "attrsList": [],
    "attrsMap": {
        "v-if": "1"
    },
    "children": [],
    "if": "1",
    "ifConditions": [{
            "exp": "1",
            "block": {
                "type": 1,
                "tag": "div",
                "attrsList": [],
                "attrsMap": {
                    "v-if": "1"
                },
                "children": [],
                "if": "1",
                "ifConditions": [],
                "plain": true,
                "static": false,
                "staticRoot": false,
                "ifProcessed": true
            }
        }, {
            "exp": "2",
            "block": {
                "type": 1,
                "tag": "p",
                "attrsList": [],
                "attrsMap": {
                    "v-else-if": "2"
                },
                "children": [],
                "elseif": "2",
                "plain": true,
                "static": false,
                "staticRoot": false
            }
        }, {
            "block": {
                "type": 1,
                "tag": "span",
                "attrsList": [],
                "attrsMap": {
                    "v-else": ""
                },
                "children": [],
                "else": true,
                "plain": true,
                "static": false,
                "staticRoot": false
            }
        }
    ],
    "plain": true,
    "static": false,
    "staticRoot": false,
    "ifProcessed": true
}
```
```js
with (this) {
    return (1) ? _c('div') :
        (2) ? _c('p') : _c('span')
}
```

### if

很明显这个就是用来存储`v-if`表达式的，这个也是代码生成的时候判断是否是`v-if`节点的标识

### ifProcessed

这个其实是用来标识`if`是不是已经处理过了，这在其他属性也会有这个类似属性，就是为了防止重复处理

### elseif

这个其实在代码生成的时候就没用了，因为`v-else-if, v-else`都放进`v-if`节点下的`ifConditions`，这就足以让我生成代码了。这个其实是用在语法解析阶段，比如可以用来判断`v-if-else-if-else`这种多个根节点的情况

### else

这个其实和`elseif`一样，而且它还没有表达式，就是一个标识，标识这个元素节点对象是带有`v-else`

### ifConditions

这个就是用于存储`v-if`系列的元素节点对象，通过它就可以构建出对应的代码