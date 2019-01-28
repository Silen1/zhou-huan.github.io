---
layout: post
title: "编译器是怎么工作的（三）—— 代码生成"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

前两篇博客（[一](https://southrill.cn/2018/12/09/how-dose-compiler-work1/)，[二](https://southrill.cn/2018/12/09/how-dose-compiler-work2/)）已经将源代码分析和分析结果转换的这两个过程写完了，还剩下最后一个过程 —— 新代码的生成。

### 新代码的生成

**codeGenerator.js**

```js
export default function codeGenerator(node) {
  switch (node.type) {
    case 'Program':
      return node.body.map(codeGenerator)
        .join('\n');
    case 'ExpressionStatement':
      return (
        codeGenerator(node.expression) +
        ';'
      );
    case 'CallExpression':
      return (
        codeGenerator(node.callee) +
        '(' +
        node.arguments.map(codeGenerator)
          .join(', ') +
        ')'
      );
    case 'Identifier':
      return node.name;
    case 'NumberLiteral':
      return node.value;
    case 'StringLiteral':
      return '"' + node.value + '"';
    default:
      throw new TypeError(node.type);
  }
}
```

**test.js**

```js
import tokenizer from './tokenizer'
import parser from './parser'
import transformer from './transformer'
import codeGenerator from './codeGenerator'

const tokenizer_res = tokenizer('(add 2 (subtract 4 2))')
const parser_res = parser(tokenizer_res)
const transformer_res = transformer(parser_res)
const codeGenerator_res = codeGenerator(transformer_res)
console.log(codeGenerator_res)

// add(2, subtract(4, 2));
```

这一步就是将 `transformer` 输出的新 AST 处理并生成 JS 代码字符串的过程。

具体来说，`switch` 语句根据传给 `codeGenerator` 的节点对象（`node`）的 `type` 属性来决定采用哪种处理方式：

* 如果 `type` 为 `Program`，将节点对象的 `body` 属性进行 `map`，处理函数为 `codeGenerator`，实际上是将 `body` 中的元素一一递归，返回了数组后，元素间进行换行，但该例中 `body` 只有一个元素不会换行。最终将输出的字符串返回。

* 如果 `type` 为 `ExpressionStatement`，那么将节点对象的 `expression` 属性交给 `codeGenerator` 进行递归，并将返回的结果加上分号之后一并返回。

* 如果 `type` 为 `CallExpression`，那么返回的字符串是这样的，将节点对象的 `callee` 属性交给 `codeGenerator` 递归，递归的结果拼上左括号，再拼上 `node.arguments.map(codeGenerator).join(', ')`（这类似于 `type` 为 `Program` 时的处理方式，只是这时要 `map` 的数组是 `arguments`，产生的数组元素间加上逗号和空格），最后再拼上右括号，完成之后将结果返回。

* 如果 `type` 为 `Identifier`，返回节点对象的 `name` 属性。

* 如果 `type` 为 `NumberLiteral`，返回节点对象的 `value` 属性。

* 如果 `type` 为 `StringLiteral`，节点对象的 `value` 属性加上双引号之后返回，但该例中不存在 `StringLiteral`。

* 如果都不是，抛出一个错误。