---
layout: post
title: Median of Two Sorted Arrays
date: 2016-10-18 23:52:14
categories: [cn, dev]
tags: [leetcode, interview]
---

编号004，你懂的。

假设两数列总共的元素数量为偶数2k，中位数则为k和k+1大的两数值的平均值；而当总元素数量为奇数2k+1，中位数则为第k+1大的元素。
这题简单的解答方式是实现一个merge函数，将两个已排序的数列合并成一个排序的数列，再根据中位数的定义求出值。
我们也还可以做一些基本的优化，例如说我们并不需要最后的排序结果，所以我们只需要维护一个counter，按序读出数字，然后再counter为0时计算中位数。
这个方法的时间复杂度是`O(n)`。

而如果想进一步达到`O(log(m + n))`的话，我们可以用分治的思路，定义一个查找两个排序数列中第k大的元素`find_kth`。
与简单的二分法应用有所不同的话，我们这里有两个数列，所以我们需要仔细考虑一下递归时的边界条件。

假设数组`a`的长度为`m`，数组`b`的长度为`n`，且`m > n >= k/2`，那么我们可以用两个变量`i`和`j`分别指向两数列中的第`k/2`个数。
如果`a[i] == b[j]`，那我们就已经找到中位数了；如果`a[i] > b[j]`，则`j`所指的元素比`a`中的`k/2`个元素都要小，所以`b[0:k/2]`都不可能存在中位数，因此我们可以只考虑`b[k/2+1:]`部分；如果`a[i] < b[j]`则同理，我们只考虑`a[k/2+1:]`部分。
每一次迭代，`a`和`b`的总长度都会减少`k/2`
之后我们再在剩下的两个数列里寻找第`(k - k/2)`大的元素。

接下来我们考虑几个base cases。
随着递归过程一直减少的变量分别有：`a`和`b`的长度`m`和`n`，还有`k`。
假设`m > n`，那么当`n`的长度为0时，我们就直接在`a`里面找第`k`大的元素。
当`k = 1`时，我们直接看两数列的开头，然后选择较小的一个。

我们还要考虑边界条件。
假设`m > n`，当`n < k/2`时，我们最多只能在`b`中取`n`个数，而在`a`中我们要取`k - n`个数。
结合`n >= k/2`的情况，可知我们在`b`中取`kb = min(k/2, n)`，而相应在`a`中取`ka = k - kb`。

最后的Python实现如下:

```python
def findMedianSortedArrays(self, nums1, nums2):
    def find_kth(k, a, i, b, j):
        m = len(a) - i
        n = len(b) - j
        if m < n:
            return find_kth(k, b, j, a, i)
        if n == 0:
            return a[i + k - 1]
        if k == 1:
            return min(a[i], b[j])

        kb = min(k // 2, n) # len(a) >= len(b), therefore ka >= k
        ka = k - kb
        ma = i + ka - 1
        mb = j + kb - 1
        if a[ma] == b[mb]:
            return a[ma]
        if a[ma] > b[mb]:
            return find_kth(k - kb, a, i, b, j + kb)
        if a[ma] < b[mb]:
            return find_kth(k - ka, a, i + ka, b, j)

    total = len(nums1) + len(nums2)
    if total % 2 == 0:
        return (find_kth(total // 2 + 1, nums1, 0, nums2, 0) +
                find_kth(total // 2, nums1, 0, nums2, 0)) / 2.0
    else:
        return find_kth(total // 2 + 1, nums1, 0, nums2, 0)
```

