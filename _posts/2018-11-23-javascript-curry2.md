---
layout: post
title: "JavaScript柯里化 —— 写个更好的curry方法"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

[上一篇的翻译](https://zhou-huan.github.io/2018/11/22/javascript-curry1/)中，原作者对于柯里化方法的最终实现是这个样子：
```js
// 定义占位符
var _ = '_';

function magician3 (targetfn, ...preset) {
  var numOfArgs = targetfn.length;
  var nextPos = 0; // 下一个有效输入位置的索引，可以是'_'，也可以是preset的结尾

  // 查看是否有足够的有效参数
  if (preset.filter(arg=> arg !== _).length === numOfArgs) {
    return targetfn.apply(null, preset);
  } else {
    // 返回'helper'函数
    return function (...added) {
      // 循环并将added参数添加到preset参数
      while(added.length > 0) {
        var a = added.shift();
        // 获取下一个占位符的位置，可以是'_'也可以是preset的末尾
        while (preset[nextPos] !== _ && nextPos < preset.length) {
          nextPos++
        }
        // 更新preset
        preset[nextPos] = a;
        nextPos++;
      }
      // 绑定更新后的preset
      return magician3.call(null, targetfn, ...preset);
    }
  }
}
```
这个实现存在一点问题：
```js
var abc = function(a, b, c) { return [a, b, c];};
var curried = magician3(abc);

curried(1)(2)(3) // [1, 2, 3]
curried(1)(_, 2)(3) // Uncaught TypeError: curried(...)(...) is not a function
```
`curried` 第一次运行功能正常，但是第二次运行就报错了。而且这个错误和使用占位符没有关系，第二次执行换作 `curried(2)(3)(4)` 也还是会报同样的错误。

一时看不出错误出在哪里，我先理一下上面的代码在运行时到底发生了什么：

1. 初始化变量 `_` ，值为 `'_'` 。

2. 初始化变量 `abc`，指向一个函数 —— `function(a, b, c) { return [a, b, c];}`，该函数用来将传入的参数转化为数组并返回。

3. 初始化变量 `curried`，实际上获得的是 `magician3(abc)` 执行之后的结果。那接着看 `magician3(abc)` 执行时发生了什么：

    * `targetfn` 初始化为传进来的 `abc`，参数 `preset` 的位置什么也没传，那么， `preset` 就被初始化为了一个空数组 `[]`。

    * `numOfArgs` 初始化为 `targetfn.length`，即3。

    * `nextPos` 初始化为0。

    * 进入条件判断语句，判断条件 `preset.filter(arg=> arg !== _).length === numOfArgs` 实际上是在说：检查一下数组 `preset` 中不为 `_` 的元素数量是不是等于 `numOfArgs`，也就是 `targetfn.length`，目标函数所期望的参数数量。这里没有传 `preset`，显然不符合判断条件，进入 `else`。

    * `else` 里面 `return` 出去了一个 `function`。这时，`magician3(abc)` 就执行完了，那 `curried` 拿到的就是 `return` 出来的这个 `function` 的引用。这个 `function` 通过闭包引用了一个空数组 `preset`，以及一个 `numOfArgs`，即所期望的参数数量 —— `3`，一个 `nextPos` —— `0`，还有一个 `targetfn`，即 `abc`。

4. 代码继续运行，到了 `curried(1)(2)(3)` 这句。这句代码会先执行 `curried(1)`，再调用 `curried(1)` 返回的结果并传入 `2`，继续调用返回的结果并传入 `3`。

5. `curried(1)`执行，代码进入 `curried` 所引用的函数，也就是之前 `magician3(abc)` 执行完之后返回的匿名函数，该函数内，`added` 被初始化为 `[1]`。

6. 继续执行，到了 `while` 循环。这个循环内部还有一个 `while` 循环，先看里面这个循环。

这个循环的终止条件是 `preset[nextPos] !== _ && nextPos < preset.length`，其实就是在说：帮我检查一下 `preset` 数组 `nextPos` 位置的元素是不是不等于 `'_'` ，并且 `nextPos` 要小于 `preset` 的长度，不满足这个条件的话 `nextPos` 就一直累加。

那能看出来这个 `while` 循环其实就是在数组 `preset` 中找一个位置，这个位置要么是 `'_'` ，要么是数组的末尾，一旦找到，循环就终止。

结合这些再来看外层的 `while` 循环，这个循环的判断条件是 `added.length > 0`，循环体内首先执行 `added.shift()` 的操作，并将结果给到变量 `a`，其实就是 `added` 中的第一个元素，然后是用来找位置（`nextPos`）的循环，找到之后把 `a` 放到 `preset` 的 `nextPos` 位置上，放完之后再把 `nextPos` 向后移一位，也就是 `nextPos++`，以便需要时继续往下找。

那可以看出，这个大的 `while` 循环作用其实就是将 `added` 中的元素从头到尾按顺序逐个给到 `preset`，并放到正确的位置上。什么是正确的位置呢，就是此时 `added` 里的元素逐个地，要么放到 `preset` 中 `'_'` 的位置（替换），如果没有 `'_'` 的话，就放到 `preset` 末尾（追加）。

那么，经过了外层这段 `while` 循环之后，`preset` 就变成了 `[1]`，而 `added` 变成了 `[]`。

7. 代码继续执行，到了 `return magician3.call(null, targetfn, ...preset);` 。这句又执行了 `magician3` 函数，并将 `targetfn` 和 `...preset`（也就是 `1`）作为 `magician3` 执行时传递的参数。那接着看具体都发生了什么：

    * 进入 `magician3` 方法，`targetfn` 还是那个 `targetfn`，指向 `abc`，`preset` 已经变成了 `[1]`。

    * `numOfArgs` 初始化为 `targetfn.length`，即3。

    * `nextPos` 初始化为0。

    * 进入条件判断语句，这时 `preset` 里只有 `1` 一个元素，显然还是会进入 `else`。

    * `else` 里还是 `return` 出去了一个 `function`，但这时这个 `function` 通过闭包引用到的 `preset` 已经是 `[1]` 了。

8. 继续执行，`curried(1)(2)(3)`，这次调用 `curried(1)` 执行完返回的函数时传入的参数就是 `2` 了。

9. 那么同样地，进入上一步返回的匿名函数，`while` 循环执行完之后，`preset` 变成了 `[1, 2]`。

10. 继续 `return magician3.call(null, targetfn, ...preset);`，同样执行 `magician3` 函数，这次传入的 `preset` 为 `[1, 2]`。继续看 `magician3` 的执行过程：

    * 进入 `magician3` 方法，`targetfn` 还是 `abc`，`preset` 变成了 `[1, 2]`。

    * `numOfArgs` 初始化为 `targetfn.length`，即3。

    * `nextPos` 初始化为0。

    * 进入条件判断语句，这时 `preset` 里只有 `1` 和 `2` 两个元素，还是会进入 `else`。

    * `else` 里还是 `return` 出去一个 `function`，这时通过闭包引用到的 `preset` 是 `[1, 2]` 。

11. 继续执行，`curried(1)(2)(3)`，这次调用返回的函数时传入的参数是 `3`。

12. 同样地，进入返回的匿名函数，`while` 循环执行完之后，`preset` 变为 `[1, 2, 3]`。

13. 继续 `return magician3.call(null, targetfn, ...preset);`，同样执行 `magician3` 函数，这次传入的 `preset` 变成了 `[1, 2, 3]`。继续看 `magician3` 的执行过程：

    * 进入 `magician3` 方法，`targetfn` 还是指向 `abc`，`preset` 变成了 `[1, 2, 3]`。

    * `numOfArgs` 初始化为 `targetfn.length`，即3。

    * `nextPos` 初始化为0。

    * 进入条件判断语句，这次 `preset` 是 `[1, 2, 3]`，满足判断条件 `preset.filter(arg=> arg !== _).length === numOfArgs`，那么进入 `if` 语句块，`return targetfn.apply(null, preset);`，也就是执行 `targetfn`，并传入 `preset`，使用 `apply` 时可以传入参数数组，其实就相当于 `targetfn(1, 2, 3)`，也就是 `abc(1, 2, 3)`，这时进入到 `function(a, b, c) { return [a, b, c];}`，返回了结果 `[1, 2, 3]`。

一直到这里看起来都是正常的，但是，再执行 `curried(1)(_, 2)(3)` 的话就出现了问题。我在想，`curried` 已经是指向一个匿名函数的引用了，那第一次执行这个函数没有问题，第二次执行时怎么就有问题了呢？

想了很久没想清楚这个 `bug` 的来龙去脉，后来在春岩和唐老师的指导下，逐渐明白了其中道理。

`curried(1)(2)(3)` 执行时，先执行 `curried(1)`。

执行 `curried(1)` 之前， `curried` 指向的是 `magician3(abc)` 执行完之后 `return` 出来的那个匿名函数，该匿名函数通过闭包引用着 `preset` —— 一个空数组 `[]`。

那等 `curried(1)` 执行时，该匿名函数通过闭包引用的 `preset` 就变成了 `[1]`（我把这个 `preset` 称为初始 `preset`），最后 `return magician3.call(null, targetfn, ...preset);` ，执行 `magician3` 并传入解构之后的 `preset`。

代码接着走，又返回了另外一个匿名函数，这个匿名函数和之前 `curried` 持有的那个匿名函数之间互不影响，唯一的联系是，`curried` 持有的匿名函数所引用的 `preset` 浅复制之后的值被后来返回的这个匿名函数引用着。

后来的这个匿名函数执行时传入了 `2`，那么这个匿名函数执行完之后 `preset` 就变成了 `[1, 2]`，但是这个 `preset` 和“初始 `preset`”是“存在不同地方的”，所以两者之间互不影响。

继续调用后来返回的这个匿名函数并传入 `3` 时，同样的，`preset`变成了 `[1, 2, 3]`，但是这个 `preset` 又是存在另外一个地方的 `preset`。

所以直到 `curried(1)(2)(3)` 执行完，`curried` 指向的匿名函数通过闭包引用的 `preset` 还是那个被 `curried(1)` 改变后的 `preset`，也就是 `[1]`。

所以，问题就在这里。

`preset` 中已经有一个元素了，这会导致 `curried(1)(_, 2)(3)` 在还没有执行完时（执行了 `curried(1)(_, 2)` ）就返回了一个数组，返回数组之后调用该数组并传入参数 `3`，显然数组是不能当作函数执行的，所以就报出了 `Uncaught TypeError: curried(...)(...) is not a function` 的错误。

`curried(1)(_, 2)(3)` 执行时，`preset` 应该是空数组才对。

那么可以这么改：

```js
var _ = '_';

function magician3 (targetfn, preset = []) {
  var numOfArgs = targetfn.length;

  if (preset.filter(arg=> arg !== _).length === numOfArgs) {
    return targetfn.apply(null, preset);
  } else {
    return function (...added) {
      var newPreset = [...preset]
      var nextPos = 0;
      while(added.length > 0) {
        var a = added.shift();
        while (newPreset[nextPos] !== _ && nextPos < newPreset.length) {
          nextPos++
        }
        newPreset[nextPos] = a;
        nextPos++;
      }
      return magician3.call(null, targetfn, newPreset);
    }
  }
}

var abc = function(a, b, c) { return [a, b, c];};
var curried = magician3(abc);

curried(1)(2)(3) // [1, 2, 3]
curried(1)('_', 2)(3) // [1, 3, 2]
```

把 `preset` 浅复制一份后给 `newPreset`，然后把 `newPreset` 和 `nextPos` 都放在返回的匿名函数内部，每个返回的匿名函数都维护着自己的数据，各自间彻底互不影响，不再像修改之前一样通过闭包引用保存 `preset` 和 `nextPos`。

最后在匿名函数内部把 `newPreset` 传给 `magician3`，`magician3` 用参数 `preset` 来接收 `newPreset`，如果没传的话（ `magician3(abc)` ），默认值为空数组 `[]`。

这样，问题就解决了。

回头再看修改之前的代码，这时会觉得 `magician3` 返回的匿名函数不是一个好函数。因为好函数不应该有副作用，而且应该是幂等的。好函数任意多次执行所产生的影响均与一次执行的影响相同。好函数可以使用相同参数重复执行，并能获得相同的执行结果。也就是说，固定输入，固定输出。很显然，它是一个坏函数。

唐老师看到这个坏函数时说，写出这样的代码不如回家养猪。可就是这样的代码，春岩讲得姿仪万方，我却听得跌跌撞撞。唉，我真的好菜...
