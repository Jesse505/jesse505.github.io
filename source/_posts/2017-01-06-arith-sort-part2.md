---
title: 图解排序算法（二）之希尔排序 
date: 2017-01-06 22:30:15
tags: 算法
---

### 基本思想

希尔排序是直接插入排序的强化版，也称为缩小增量排序。

先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，待整个序列中的记录“基本有序”时，再对全体记录进行依次直接插入排序。  
我们来看下希尔排序的基本步骤，在此我们选择增量gap=length/2，缩小增量继续以gap = gap/2的方式，这种增量选择我们可以用一个序列来表示，{n/2,(n/2)/2...1}，称为增量序列。希尔排序的增量序列的选择与证明是个数学难题，我们选择的这个增量序列是比较常用的，也是希尔建议的增量，称为希尔增量，但其实这个增量序列不是最优的。此处我们做示例使用希尔增量。

<!--more-->

<img src="/img/201701/shellSort.png" alt="shell sort png" style="width: 600px;">

### Java实现

```java
    //希尔排序(增强版的插入排序)
    public static void shellSort(int[] nums) {
        int size = nums.length;
        int temp;
        int j;
        for(int increment = size/2; increment > 0; increment /=2 ) {
            for (int i = increment; i < size; i++) {
                temp = nums[i];
                for (j = i; j >= increment; j-=increment) {
                    if (temp < nums[j-increment]) {
                        nums[j] = nums[j-increment];
                    } else {
                        break;
                    }
                }
                nums[j] = temp;
            }
        }
    }
```

时间复杂度O(n^1.5)






