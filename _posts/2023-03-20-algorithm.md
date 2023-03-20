---
title: 几道基础算法题
author: 天枫皓月
date: 2023-03-20
category: javascript
layout: post
---

好基友最近在准备面试，没事刷刷算法题，发给了我几道，本着编程的乐趣，就一起尝试写了写。

一、 **给定整数数组 nums 和整数 k ，请返回数组中第 k 个最大的元素。请注意，你需要找的是数组排序后的第 k 个最大的元素，而不是第 k 个不同的元素。**

这道题其实考的是冒泡排序，冒泡排序的特点是逐个将最大/小的数找出来，而本题只需要找到第k个最大的元素，所以冒泡到第k个数之后，就可以中止排序，无需将整个数组排序。

~~~javascript
/**
 * @param {number[]} nums 待查询数组
 * @param {number} nth 第几大的数
 * @returns {number}
 */
function findNthMax(nums, nth) {
  const len = nums.length;
  const realNth = nth > len ? len : nth;
  
  for (let i=0; i < realNth; i++) {
    for (let j=i; j < len - 1; j++) {
      const n1 = nums[i];
      const n2 = nums[j + 1];
      if (n2 > n1) {
        nums[i] = n2;
        nums[j + 1] = n1;
      } 
    }
  }

  return nums[nth - 1];
}

console.log(findNthMax([2, 1, 4, 5, 6, 6], 2));
~~~
顺手用vue组件写了个冒泡排序过程的动画，加深理解

![冒泡排序](/assets/imgs/bsort.gif)

二、**归并排序**

```typescript
/**
 * 归并排序
 * @param {number[]} nums 待排序数组
 * @returns {number[]} 由小到大排序后的数组 
 */
function mergeSort(nums) {
  const len = nums.length;

  if (len > 2) {
    const splitPoint = Math.ceil((len - 1) / 2);
    return merge(mergeSort(nums.slice(0, splitPoint)), mergeSort(nums.slice(splitPoint)));
  }
  if (len === 2) {
    const [n1, n2] = nums;
    return n1 <= n2 ? nums : [n2, n1];
  }

  return nums;
}
/**
 * 将两个排好序的数组进行合并
 * @param {number[]} a 待合并数组
 * @param {number[]} b 待合并数组
 * @returns {number[]}
 */
function merge(a, b) {
  const result = [];

  while (a.length || b.length) {
    // 如果a数组的元素先取空，则将b数组剩余的元素全部合并至result
    if (!a.length){
      return result.concat(b);
    }
    // 同理，如果b数组的元素先取空，则将a数组剩余的元素全部合并至result
    if (!b.length) {
      return result.concat(a);
    }
  
    result.push(a[0] < b[0] ? a.shift() : b.shift());
  }

  return result;
}
mergeSort([3, 1, 2, 6, 1, 5, 2, 9]);
```
试着用haskell实现了一下，函数式编程，真的很优雅

```haskell
merge :: Ord a => [a] -> [a] -> [a]
merge xs [] = xs
merge [] ys = ys
merge (x:xs) (y:ys) | x <= y    = x:merge xs (y:ys)
                    | otherwise = y:merge (x:xs) ys
msort :: Ord a => [a] -> [a]
msort [] = []
msort [a] = [a]
msort xs = merge (msort (firstHalf xs)) (msort (secondHalf xs))
firstHalf xs = let { n = length xs } in take (div n 2) xs
secondHalf xs = let { n = length xs } in drop (div n 2) xs
main = print (msort [2, 1, 3, 5, 6, 0, 2])
```

三、**快速排序**

haskell对快排的实现从我接触这门语言起，就给我留下了深刻的印象，代码就是对算法的描述，没有任何的多余。

```haskell
qsort :: Ord a => [a] -> [a]
qsort [] = []
qsort (x:xs) = qsort [a | a <-xs, a < x ] ++ [x] ++ qsort [b | b <-xs, b >= x]
```
为了给基友解释，还特意画了一张图

![haskell的快速排序](/assets/imgs/qsort.jpeg)
