---
title: 图解排序算法（一）冒泡，选择，直接插入  
date: 2017-01-05 23:01:15
tags: 算法
---


## 冒泡排序

### 基本思想

比较相邻的元素。如果第一个比第二个大，就交换他们两个。

对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。在这一点，最后的元素应该会是最大的数。 针对所有的元素重复以上的步骤，除了最后一个。持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

### Java实现

加入标记状态 flag 若在一次冒泡中，没有交换 则说明可以停止 减少运行时

```java
    //冒泡排序
    public static void bubbleSort(int[] nums) {
        int size = nums.length;
        int temp;
        boolean flag = true;
        for (int i = 0; i < size - 1 && flag; i++) {
            flag = false;
            for (int j = 0; j < size -1 -i; j++) {
                if (nums[j] > nums[j+1]) {
                    temp = nums[j];
                    nums[j] = nums[j+1];
                    nums[j+1] = temp;
                    flag = true;
                }
            }
        }
    }
```

时间复杂度O(n*n)

<!--more-->

## 简单选择排序

### 基本思想

在要排序的一组数中，选出最小的一个数与第一个位置的数交换；然后在剩下的数当中再找最小的与第二个位置的数交换，如此循环到倒数第二个数和最后一个数比较为止。

### Java实现

```java
    //简单选择排序
    public static void selectSort(int[] nums) {
        int size = nums.length;
        int temp;//中间变量
        for (int i = 0; i < size - 1; i++) {
            int k = i; //待确定的位置
            for (int j = size - 1; j > i ; j--) {
                if (nums[j] < nums[k]) {
                    k = j;
                }
            }
            //交换两个数
            if (k != i) {
                temp = nums[i];
                nums[i] = nums[k];
                nums[k] = temp;
            }
        }
    }
```
时间复杂度O(n*n) 性能上优于冒泡排序 交换次数少

## 直接插入排序

### 基本思想

每一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。

<img src="/img/201701/insertSort.png" alt="insert sort png" style="width: 600px;">

### Java实现

```java
    //插入排序
    public static void insertSort(int[] nums) {
        int size = nums.length;
        int temp;
        int j;
        for (int i = 1; i < size; i++) {
            temp = nums[i];
            //如果temp比前面的值小，则将前面的值往后移
            for (j = i;  j >= 1 && temp < nums[j-1]; j--) {
                nums[j] = nums[j-1];
            }
            nums[j] = temp;
        }
    }
```

简单插入排序在最好情况下，需要比较n-1次，无需交换元素，时间复杂度为O(n);在最坏情况下，时间复杂度依然为O(n2)。但是在数组元素随机排列的情况下，插入排序还是要优于上面两种排序的。





