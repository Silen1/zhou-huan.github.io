---
layout: post
title: "几种排序算法"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - JavaScript
---

#### 快速排序

```js
function quickSort (arr) {
  if (arr.length <= 1) return arr
  const pivot = arr.splice(0, 1)[0]
  let left = []
  let right = []
  arr.forEach(item => {
    if (item < pivot) {
      left.push(item)
    } else {
      right.push(item)
    }
  })
  return quickSort(left).concat([pivot], quickSort(right))
}
```

快速排序使用分治法策略来把一个序列分为两个子序列。

步骤为：

1. 从数列中挑出一个元素，称为“基准”（pivot）。
2. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（相同的数可以到任何一边）。在这个分割结束之后，该基准就处于数列的中间位置。这个称为分割操作。
3. 递归地把小于基准值元素的子数列和大于基准值元素的子数列排序。

递归到最底部时，数列的大小是零或一，也就是已经排序好了。这个算法一定会结束，因为在每次的迭代中，它至少会把一个元素摆到它最后的位置去。

在平均状况下，排序n个项目要O(n log n)次比较。在最坏状况下则需要O(n^2)次比较，但这种状况并不常见。事实上，快速排序O(n log n)通常明显比其他算法更快。

#### 选择排序

```js
function selectionSort (arr) {
  let len = arr.length
  for (let i=0; i<len-1; i++) {
    let min = arr[i]
    let minIndex = i
    for (let j=i+1; j<len; j++) {
      if (arr[j] < min) {
        min = arr[j]
        minIndex = j
      }
    }
    let temp = arr[i]
    arr[i] = min
    arr[minIndex] = temp
  }
  return arr
}
```

选择排序是一种简单直观的排序算法。它的工作原理如下。首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。

选择排序的主要优点与数据移动有关。如果某个元素位于正确的最终位置上，则它不会被移动。选择排序每次交换一对元素，它们当中至少有一个将被移到其最终位置上，因此对n个元素的表进行排序总共进行至多n-1次交换。在所有的完全依靠交换去移动元素的排序方法中，选择排序属于非常好的一种。

#### 插入排序

```js
function insertionSort (arr) {
  for (let i=1; i<arr.length; i++) {
    for (let j=0; j<i; j++) {
      if (arr[i] < arr[j]) {
        arr.splice(j, 0, arr[i])
        arr.splice(i+1, 1)
        break
      }
    }
  }
  return arr
}
```

插入排序是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描（其实扫描顺序没关系，只要根据自己的排序方式（从大到小还是从小到打）选择较快的扫描顺序找到正确的位置插入即可），找到相应位置并插入。

插入排序在实现上，通常采用in-place（原地算法）排序（即只需用到O(1)的额外空间的排序），因而在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。

#### 冒泡排序

```js
function bubbleSort (arr) {
  for (let i=0; i<arr.length; i++) {
    for (let j=0; j<arr.length-1-i; j++) {
      if (arr[j] > arr[j+1]) {
        let box = arr[j]
        arr[j] = arr[j+1]
        arr[j+1] = box
      }
    }
  }
  return arr
}
```

冒泡排序是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。

冒泡排序对n个项目需要O(n^2)的比较次数，且可以原地排序。尽管这个算法是最简单了解和实现的排序算法之一，但它对于包含大量的元素的数列排序是很没有效率的。

