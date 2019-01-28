---
layout: post
title: "编译器是怎么工作的（二）—— AST的转换"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

杭州一夜大雪，所有的树枝都被压弯了腰。正好有一枝不偏不倚弯到窗前，打开窗帘时着实吓了一跳。但定睛看时，又觉得一片粉妆玉砌，好不漂亮。此时此刻望着这一派雪景，连我这个粗人都想吟诗一首，但意料之中，什么也想不起来，只好接着来写我的博客。

书接上回，继续写编译器的工作方式。上一篇已经写完了第一步的内容 —— 源代码分析，还剩下两步 —— 分析结果转换和新代码生成。

### 分析结果转换

**traverser.js**
```js
export default function traverser(ast, visitor) {
  function traverseArray(array, parent) {
    array.forEach(child => {
      traverseNode(child, parent);
    });
  }
  function traverseNode(node, parent) {
    let methods = visitor[node.type];
    if (methods && methods.enter) {
      methods.enter(node, parent);
    }
    switch (node.type) {
      case 'Program':
        traverseArray(node.body, node);
        break;
      case 'CallExpression':
        traverseArray(node.params, node);
        break;
      case 'NumberLiteral':
      case 'StringLiteral':
        break;
      default:
        throw new TypeError(node.type);
    }
    if (methods && methods.exit) {
      methods.exit(node, parent);
    }
  }
  traverseNode(ast, null);
}
```
**transformer.js**
```js
import traverser from './traverser'

export default function transformer(ast) {
  let newAst = {
    type: 'Program',
    body: [],
  };
  ast._context = newAst.body;
  traverser(ast, {
    NumberLiteral: {
      enter(node, parent) {
        parent._context.push({
          type: 'NumberLiteral',
          value: node.value,
        });
      },
    },
    StringLiteral: {
      enter(node, parent) {
        parent._context.push({
          type: 'StringLiteral',
          value: node.value,
        });
      },
    },
    CallExpression: {
      enter(node, parent) {
        let expression = {
          type: 'CallExpression',
          callee: {
            type: 'Identifier',
            name: node.name,
          },
          arguments: [],
        };
        node._context = expression.arguments;
        if (parent.type !== 'CallExpression') {
          expression = {
            type: 'ExpressionStatement',
            expression: expression,
          };
        }
        parent._context.push(expression);
      },
    }
  });
  return newAst;
}
```
**test.js**
```js
import tokenizer from './tokenizer'
import parser from './parser'
import transformer from './transformer'

const tokenizer_res = tokenizer('(add 2 (subtract 4 2))')
const parser_res = parser(tokenizer_res)
const transformer_res = transformer(parser_res)
console.log(transformer_res)

// {
//   "type": "Program",
//   "body": [
//     {
//       "type": "ExpressionStatement",
//       "expression": {
//         "type": "CallExpression",
//         "callee": {
//           "type": "Identifier",
//           "name": "add"
//         },
//         "arguments": [
//           {
//             "type": "NumberLiteral",
//             "value": "2"
//           },
//           {
//             "type": "CallExpression",
//             "callee": {
//               "type": "Identifier",
//               "name": "subtract"
//             },
//             "arguments": [
//               {
//                 "type": "NumberLiteral",
//                 "value": "4"
//               },
//               {
//                 "type": "NumberLiteral",
//                 "value": "2"
//               }
//             ]
//           }
//         ]
//       }
//     }
//   ]
// }
```

这段代码运行之后的结果是将之前 parser 生成的 AST 转成了一个新的 AST，那为什么要转这么一步呢，可以这么理解，旧的 AST 是 LISP 风格代码的抽象语法树，而新的 AST 是 JS 代码的抽象语法树。

总之，经过这段程序转换之后生成的新 AST 虽然更长了一点，但是更能说明 JS 风格代码各部分之间的关系。既然我们要基于原来的代码生成一份新的代码，那么最好先生成一份新的 AST。就像如果非要将 React 代码的抽象语法树直接转成 JS 代码的话肯定也是能做到的，但是会非常吃力，也会非常不合理，可维护性和可扩展性等都会非常差。

这段程序具体来说，是在 `transformer` 方法中调用了 `traverser`，同时传递了两个参数，旧的 AST 和一个 visitor。这个 visitor 简化一下是这样的：
```js
{
  NumberLiteral: {
    enter(node, parent) {
      /*省略*/
    },
  },
  StringLiteral: {
    enter(node, parent) {
      /*省略*/
    },
  },
  CallExpression: {
    enter(node, parent) {
      /*省略*/
    },
  }
}
```
这时再看 `traverser` 方法。该方法内部又定义了两个方法，`traverseArray` 和 `traverseNode`。当碰到 `node.body` 和 `node.params` 这样的数组时调用 `traverseArray` 进行遍历，对数组中的每个元素再调用 `traverseNode`。实际上就是把旧 AST 中的每个对象节点过了一遍，对每一个都执行一下这几句代码：
```js
let methods = visitor[node.type];
if (methods && methods.enter) {
  methods.enter(node, parent);
}
```
这里的 `parent` 实际上是从 `switch` 语句中的 `traverseArray` 的调用透传过来的：
```js
case 'Program':
  traverseArray(node.body, node);
  break;
case 'CallExpression':
  traverseArray(node.params, node);
  break;
```
当调用 `enter` 方法时，保持了 `node` 和它的父节点对象 `parent` 的关系，保证了 `enter` 方法中能够正确获得每一个节点对象的父元素。

这时再看 `transformer` 方法中传递给 `traverser` 的 `visitor` 中的 `enter` 方法。

`node.type` 为 `NumberLiteral` 时 `enter` 方法为：
```js
enter(node, parent) {
  parent._context.push({
    type: 'NumberLiteral',
    value: node.value,
  });
}
```
`node.type` 为 `StringLiteral` 时 `enter` 方法为：
```js
enter(node, parent) {
  parent._context.push({
    type: 'StringLiteral',
    value: node.value,
  });
}
```
那这两个 `parent._context` 又是什么呢，这时就得看 `node.type` 为 `CallExpression` 时 `enter` 方法了：
```js
enter(node, parent) {
  let expression = {
    type: 'CallExpression',
    callee: {
      type: 'Identifier',
      name: node.name,
    },
    arguments: [],
  };
  // 这里
  node._context = expression.arguments;
  if (parent.type !== 'CallExpression') {
    expression = {
      type: 'ExpressionStatement',
      expression: expression,
    };
  }
  parent._context.push(expression);
}
```
`_context` 指向的实际上是 `expression` 的 `arguments` 属性，这保证了原来的 `CallExpression` 中的参数现在都被放进 `arguments` 中，这么做是因为原来的代码风格是这样：
```js
(add 1 1)
```
现在要转成这样：
```js
add(1, 1)
```
`expression` 这个对象除了 `arguments` 之外还有两个属性，`type` 和 `callee`。`type` 为 `CallExpression`，标识了这是一个要调用的表达式。`callee` 又是一个对象，这个对象的 `type` 属性为 `Identifier`，`name` 属性的值 `node.name` 实际上标记的就是透传过来的 `add` 或 `subtract`，操作类型，那么 `callee` 对象最后就标识了这个表达式到底在调用什么。

这个 `enter` 方法中还有一段剩下的代码：
```js
if (parent.type !== 'CallExpression') {
  expression = {
    type: 'ExpressionStatement',
    expression: expression,
  };
}
parent._context.push(expression);
```
检测如果 `parent` 的类型不为 `CallExpression` 这一条件成立，那么说明这已经是最外层的调用了。这时将 `expression` 对象改为一个新的对象，新对象的 `type` 属性为 `ExpressionStatement`，标识了这是一个表达式语句，新对象的 `expression` 属性为上面生成的 `expression` 对象。如果条件不成立就什么也不做。

最后将 `expression` 对象 push 进 `parent._context` 中。如果此时是最外层的调用，`parent._context` 就是 `newAst.body`，这时条件语句中的代码也执行了，push 进去的对象就是这样：
```js
{
  type: 'ExpressionStatement',
  expression: {
    {
      type: 'CallExpression',
      callee: {
        type: 'Identifier',
        name: /*...*/,
      },
      arguments: [/*...*/]
    }
  }
}
```
如果不是最外层的调用，那 `parent._context` 就是 `expression.arguments`，push 进去的对象就是这样：
```js
{
  type: 'CallExpression',
  callee: {
    type: 'Identifier',
    name: /*...*/,
  },
  arguments: [/*...*/]
}
```
