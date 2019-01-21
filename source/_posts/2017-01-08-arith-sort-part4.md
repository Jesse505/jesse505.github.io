---
title: 图解排序算法（四）之归并排序 
date: 2017-01-08 20:11:56
tags: 算法
categories: [算法]
---


### 基本思想

归并排序（MERGE-SORT）是利用**归并**的思想实现的排序方法，该算法采用经典的**分治**（divide-and-conquer）策略（分治法将问题分(divide)成一些小的问题然后**递归**求解，而治(conquer)的阶段则将分的阶段得到的各答案"修补"在一起，即**分而治之**)。  
**分而治之**
<img src="/img/201701/mergeSort1.png" alt="merge sort png" style="width: 600px;">

可以看到这种结构很像一棵完全二叉树，本文的归并排序我们采用递归去实现（也可采用迭代的方式去实现）。分阶段可以理解为就是递归拆分子序列的过程，递归深度为log2n。

<!--more-->

### 合并相邻有序子序列

再来看看治阶段，我们需要将两个已经有序的子序列合并成一个有序序列，比如上图中的最后一次合并，要将[4,5,7,8]和[1,2,3,6]两个已经有序的子序列，合并为最终序列[1,2,3,4,5,6,7,8]，来看下实现步骤。

<img src="/img/201701/mergeSort2.png" alt="merge sort png" style="width: 600px;">

<img src="/img/201701/mergeSort3.png" alt="merge sort png" style="width: 600px;">

### Java实现

```java
    //归并排序
    public static void mergeSort(int[] arr) {
        int[] temp = new int[arr.length];
        sort(arr,0, arr.length-1, temp);
    }

    private static void sort(int[] arr, int left, int right, int[] temp) {
        if (left < right) {
            int mid = (left + right)/2;
            sort(arr, left, mid, temp);//左子序列归并排序，使左子序列有序
            sort(arr, mid+1, right, temp);//右子序列归并排序，使右子序列有序
            merge(arr, left, mid, right, temp);
        }
    }

    //合并两个有序的子序列
    private static void merge(int[] data, int left, int mid, int right, int[] temp) {
        int i = left; //左序列指针
        int j = mid + 1; //右序列指针
        int t = 0;    //临时数组指针
        while (i <= mid && j <= right) {
            if (data[i] < data[j]) {
                temp[t++] = data[i++];
            } else {
                temp[t++] = data[j++];
            }
        }
        //把左边剩余元素填充到temp中
        while (i <= mid) {
            temp[t++] = data[i++];
        }
        while (j <= right) {
            temp[t++] = data[j++];
        }
        t = 0;
        //将temp中的元素全部拷贝到原数组中
        while(left <= right) {
            data[left++] = temp[t++];
        }
    }
```

### 总结

归并排序是稳定排序，它也是一种十分高效的排序，能利用完全二叉树特性的排序一般性能都不会太差。java中Arrays.sort()采用了一种名为TimSort的排序算法，就是归并排序的优化版本。从上文的图中可看出，每次合并操作的平均时间复杂度为O(n)，而完全二叉树的深度为|log2n|。总的平均时间复杂度为O(nlogn)。而且，归并排序的最好，最坏，平均时间复杂度均为O(nlogn)。





