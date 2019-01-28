---
layout: post
title: "编译器是怎么工作的（一）—— 生成AST（抽象语法树）"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

虽然每天的工作中都会用到编译器，但我从来没有研究过编译器到底是怎么工作的，昨天阿润推荐了一个 [the-super-tiny-compiler](https://github.com/jamiebuilds/the-super-tiny-compiler)，看了之后对编译器有了一些初步的认识，写点笔记以作总结。

*本文所说到的编译是指前端框架、前端工具所做的编译工作，不是源代码转二进制机器码的编译过程。*

## 思考

*react.js*
```js
class Index extends React.Component {
  render() {
    return (
      <h1>Hello React!</h1>
    )
  }
}
```
*vue.js*
```html
<div id="app">
  {{ message }}
</div>
```
```js
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```
使用react.js和vue.js语法写出来的代码都是浏览器所不认识的，如果要交给浏览器运行，就要事先经过编译器的处理。

## 分析

为了理解vue、react以及其它编译器所做的复杂的工作，可以找一个小的切入点来了解这项工作的大致流程。比如 [the-super-tiny-compiler](https://github.com/jamiebuilds/the-super-tiny-compiler) 中的例子：

`(add 2 (subtract 4 2))` => `add(2, subtract(4, 2))`

将LISP风格的代码转成浏览器能够正确解析的JS代码。

这项工作大体上又分为以下三个步骤：

1. 源代码分析
2. 分析结果转换
3. 新代码生成

具体来说是这样的：

### 源代码分析阶段

#### 第一步 词法分析

**tokenizer.js**
```js
export default function tokenizer(input) {
  let current = 0;
  let tokens = [];
  while (current < input.length) {
    let char = input[current];
    if (char === '(') {
      tokens.push({
        type: 'paren',
        value: '(',
      });
      current++;
      continue;
    }
    if (char === ')') {
      tokens.push({
        type: 'paren',
        value: ')',
      });
      current++;
      continue;
    }
    let WHITESPACE = /\s/;
    if (WHITESPACE.test(char)) {
      current++;
      continue;
    }
    let NUMBERS = /[0-9]/;
    if (NUMBERS.test(char)) {
      let value = '';
      while (NUMBERS.test(char)) {
        value += char;
        char = input[++current];
      }
      tokens.push({ type: 'number', value });
      continue;
    }
    if (char === '"') {
      let value = '';
      char = input[++current];
      while (char !== '"') {
        value += char;
        char = input[++current];
      }
      char = input[++current];
      tokens.push({ type: 'string', value });
      continue;
    }
    let LETTERS = /[a-z]/i;
    if (LETTERS.test(char)) {
      let value = '';
      while (LETTERS.test(char)) {
        value += char;
        char = input[++current];
      }
      tokens.push({ type: 'name', value });
      continue;
    }
    throw new TypeError('I dont know what this character is: ' + char);
  }
  return tokens;
}
```
**test.js**
```js
import tokenizer from './tokenizer'

const tokenizer_res = tokenizer('(add 2 (subtract 4 2))')
console.log(tokenizer_res)
// [
//   {
//     "type": "paren",
//     "value": "("
//   },
//   {
//     "type": "name",
//     "value": "add"
//   },
//   {
//     "type": "number",
//     "value": "2"
//   },
//   {
//     "type": "paren",
//     "value": "("
//   },
//   {
//     "type": "name",
//     "value": "subtract"
//   },
//   {
//     "type": "number",
//     "value": "4"
//   },
//   {
//     "type": "number",
//     "value": "2"
//   },
//   {
//     "type": "paren",
//     "value": ")"
//   },
//   {
//     "type": "paren",
//     "value": ")"
//   }
// ]
```
tokenizer.js 所做的工作实际上就是进行词法分析，然后分词。按照语义将输入的代码拆分开来，每一个小块最终都将拼装成一个对象，该对象拥有 `type` 和 `value` 属性，最终这些小块都被 push 进一个数组当中，这个数组实际上就是第一步词法分析的结果，tokenizer 方法最终将这个数组返回。

具体来说，tokenizer 方法将输入的代码使用 while 循环逐字遍历，遍历过程中使用条件判断语句分别查找 `'('`， `')'`，空格，数字，`"`，字母。虽然是逐字遍历，但匹配到数字，`"`，字母后的条件语句块内又有一个 while 循环，这个 while 循环的作用是：
* 如果当前字符是数字的话，就继续看下一个字符是否为数字，如果是，就继续这一动作，一直找到不是数字为止。

  将找到的这些连续的数字作为 `value`，组装成 `{ type: 'number', value }` 的对象，放进数组中。
* 如果当前字符是 `"` 的话，就继续看下一个字符是否为 `"`，如果不是，就继续往下看，如果是，就终止。

  最终取出了双引号内的字符作为 `value`，组装成 `{ type: 'string', value }` 的对象，放进了数组中。

  但该例中是没有 `type` 为 `string` 的内容的，所以也没有进这个 if 语句块。如果输入为 `'(add 2 (subtract "4" 2))'` 的话，就会进了。
* 如果当前字符是字母的话，就继续看下一个字符，如果还是字母，就继续该动作，如果不是，if 语句块内的 while 循环就终止。

  最终拿出了连续的字母作为 `value`，组装成 `{ type: 'name', value }` 的对象放进数组。

剩下的几种情况：
* 如果是空格的话就跳过，继续下一个字符的判断。
* 如果是 `'('` 或者 `')'`，那么对应的对象的 `type` 就为 `paren`，表示它是括号，`value` 就是相应的 `'('` 或者 `')'`。

#### 第二步 句法分析（语法分析）

**parser.js**
```js
export default function parser(tokens) {
  let current = 0;
  function walk() {
    let token = tokens[current];
    if (token.type === 'number') {
      current++;
      return {
        type: 'NumberLiteral',
        value: token.value,
      };
    }
    if (token.type === 'string') {
      current++;
      return {
        type: 'StringLiteral',
        value: token.value,
      };
    }
    if (
      token.type === 'paren' &&
      token.value === '('
    ) {
      token = tokens[++current];
      let node = {
        type: 'CallExpression',
        name: token.value,
        params: [],
      };
      token = tokens[++current];
      while (
        (token.type !== 'paren') ||
        (token.type === 'paren' && token.value !== ')')
      ) {
        node.params.push(walk());
        token = tokens[current];
      }
      current++;
      return node;
    }
    throw new TypeError(token.type);
  }
  let ast = {
    type: 'Program',
    body: [],
  };
  while (current < tokens.length) {
    ast.body.push(walk());
  }
  return ast;
}
```

**test.js**
```js
import tokenizer from './tokenizer'
import parser from './parser'

const tokenizer_res = tokenizer('(add 2 (subtract 4 2))')
const parser_res = parser(tokenizer_res)
console.log(parser_res)

// {
//   "type": "Program",
//   "body": [
//     {
//       "type": "CallExpression",
//       "name": "add",
//       "params": [
//         {
//           "type": "NumberLiteral",
//           "value": "2"
//         },
//         {
//           "type": "CallExpression",
//           "name": "subtract",
//           "params": [
//             {
//               "type": "NumberLiteral",
//               "value": "4"
//             },
//             {
//               "type": "NumberLiteral",
//               "value": "2"
//             }
//           ]
//         }
//       ]
//     }
//   ]
// }
```
parser.js 所做的工作实际上就是，对词法分析之后的结果（tokenizer 返回的数组）再次进行分析，分析过程中将该数组按照特定的格式转换成一个对象，该对象描述了原输入语句（`'(add 2 (subtract 4 2))'`）中各部分的具体内容/类型以及它们之间的关系，这就是我们所说的句法分析，句法分析之后输出的对象就是 **AST** —— Abstract Syntax Tree，抽象语法树。

具体来说，该例中输出的 AST `type` 为 `Program`，`body` 中存放的是句法分析的结果。

parser.js 中，使用了 while 循环对词法分析之后返回的数组进行遍历，每一次遍历都调用一次 `walk` 方法，并将 `walk` 方法返回的结果 push 进 `body` 中。`walk` 方法的内部主要进行了三项工作：

* 当前遍历到的对象 `token` 如果为 number，那么 `walk` 方法的返回值就为：
```js
{
  type: 'NumberLiteral',
  value: token.value
}
```

* 当前遍历到的对象 `token` 如果为 string，那么 `walk` 方法的返回值就为：
```js
{
  type: 'StringLiteral',
  value: token.value
}
```
当然前面已经说过了，这个例子中是没有字符串节点对象的。

* 当前遍历到的对象 `token` 如果为 `'('`，处理起来会稍微复杂一些。首先该括号节点对象会有两个属相，`type` 和 `name`。其中，`type` 固定为 `CallExpression`，`name` 的值取决于括号 `'('` 之后紧跟着的操作类型，`add` 或者 `subtract`。再然后，句法分析程序会把括号内的内容（除去操作类型）作为当前括号节点对象的参数，具体来说是 `params` 属性。`walk` 方法的返回值就会是这样：

```js
{
  type: 'CallExpression',
  name: 'add / subtract',
  params: [/*省略*/]
}
```

  再来看 `params` 属性的具体内容，代码中还是使用了一个 while 循环来对 `params` 属性进行处理：

  ```js
  if (
    token.type === 'paren' &&
    token.value === '('
  ) {
    token = tokens[++current];
    let node = {
      type: 'CallExpression',
      name: token.value,
      params: [],
    };
    token = tokens[++current];
    // 这个while循环
    while (
      (token.type !== 'paren') ||
      (token.type === 'paren' && token.value !== ')')
    ) {
      node.params.push(walk());
      token = tokens[current];
    }
    current++;
    return node;
  }
  ```
实际上就是在词法分析返回的数组中，对当前括号之内（括号之内可能还有括号）所有除去操作类型（`add` 或者 `subtract`）和 `'('` 的对象（词法分析返回的数组中的元素）递归调用 `walk` 方法，这时将 `walk` 方法返回的结果 push 进 `params` 属性中。

如果 while 循环终止了，会继续执行外面的 while 循环分析之后的数组元素：
```js
while (current < tokens.length) {
  ast.body.push(walk());
}
```
由于 `current` 是一个全局维护的变量，所以也不会重复分析和重复收集。
