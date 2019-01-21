---
title: 图解排序算法（三）之堆排序 
date: 2017-01-07 22:30:15
tags: 算法
categories: [算法]
---

### 预备知识

##### 堆排序

堆排序是利用堆这种数据结构而设计的一种排序算法，堆排序是对直接选择排序的改进，**它的最坏，最好，平均时间复杂度均为O(nlogn)，它也是不稳定排序，不适合待排序序列较少的情况**。首先简单了解下堆结构。

##### 堆

**堆是具有以下性质的完全二叉树：每个结点的值都大于或等于其左右孩子结点的值，称为大顶堆；或者每个结点的值都小于或等于其左右孩子结点的值，称为小顶堆。如下图：**

<img src="/img/201701/heapSort1.png" alt="heap sort png">

<!--more-->

同时，我们对堆中的结点按层进行编号，将这种逻辑结构映射到数组中就是下面这个样子

<img src="/img/201701/heapSort2.png" alt="heap sort png" style="width: 600px;">

该数组从逻辑上讲就是一个堆结构，我们用简单的公式来描述一下堆的定义就是：  
**大顶堆：arr[i] >= arr[2i+1] && arr[i] >= arr[2i+2]**   
**小顶堆：arr[i] <= arr[2i+1] && arr[i] <= arr[2i+2]**  
ok，了解了这些定义。接下来，我们来看看堆排序的基本思想及基本步骤：

### 基本思想及基本步骤

>堆排序的基本思想是：将待排序序列构造成一个大顶堆，此时，整个序列的最大值就是堆顶的根节点。将其与末尾元素进行交换，此时末尾就为最大值。然后将剩余n-1个元素重新构造成一个堆，这样会得到n个元素的次小值。如此反复执行，便能得到一个有序序列了

**步骤一 构造初始堆。将给定无序序列构造成一个大顶堆（一般升序采用大顶堆，降序采用小顶堆)。**  

1.假设给定无序序列结构如下  
<img src="/img/201701/heapSort3.png" alt="heap sort png" style="width: 300px;">

2.此时我们从最后一个非叶子结点开始（叶结点自然不用调整，第一个非叶子结点 arr.length/2-1=5/2-1=1，也就是下面的6结点），从左至右，从下至上进行调整。   
<img src="/img/201701/heapSort4.png" alt="heap sort png" style="width: 600px;">

3.找到第二个非叶节点4，由于[4,9,8]中9元素最大，4和9交换。 
<img src="/img/201701/heapSort5.png" alt="heap sort png" style="width: 600px;">  

这时，交换导致了子根[4,5,6]结构混乱，继续调整，[4,5,6]中6最大，交换4和6。  
<img src="/img/201701/heapSort6.png" alt="heap sort png" style="width: 600px;">  

此时，我们就将一个无需序列构造成了一个大顶堆。

**步骤二 将堆顶元素与末尾元素进行交换，使末尾元素最大。然后继续调整堆，再将堆顶元素与末尾元素交换，得到第二大元素。如此反复进行交换、重建、交换。**

1.将堆顶元素9和末尾元素4进行交换
<img src="/img/201701/heapSort7.png" alt="heap sort png" style="width: 600px;"> 

2.重新调整结构，使其继续满足堆定义
<img src="/img/201701/heapSort8.png" alt="heap sort png" style="width: 600px;"> 

3.再将堆顶元素8与末尾元素5进行交换，得到第二大元素8.
<img src="/img/201701/heapSort9.png" alt="heap sort png" style="width: 600px;"> 

后续过程，继续进行调整，交换，如此反复进行，最终使得整个序列有序

<img src="/img/201701/heapSort10.png" alt="heap sort png" style="width: 300px;"> 


再简单总结下堆排序的基本思路：

&emsp; &emsp; **1. 将无需序列构建成一个堆，根据升序降序需求选择大顶堆或小顶堆;**

&emsp; &emsp; **2. 将堆顶元素与末尾元素交换，将最大元素"沉"到数组末端;**

&emsp; &emsp; **3. 重新调整结构，使其满足堆定义，然后继续交换堆顶元素与当前末尾元素，反复执行调整+交换步骤，直到整个序列有序。**

### Java实现

```java
    //堆排序（强化版的选择排序）
    public static void heapSort(int[] data) {
        for(int i = 0; i < data.length - 1; i++) {
            buildMaxHeap(data, data.length - 1 - i);
            swap(data, 0, data.length - 1 - i);
        }
    }


    //构建大顶堆
    private static void buildMaxHeap(int[] data, int lastIndex) {
        //从尾节点的父节点开始
        for (int i = (lastIndex - 1)/2; i >= 0; i--) {
            int k = i;
            //说明有叶子节点
            while (k*2 + 1 <= lastIndex) {
                int biggestIndex = k*2 +1;
                //说明有右节点
                if (biggestIndex < lastIndex) {
                    //如果左节点小于右节点，则biggestIndex+1
                    if (data[biggestIndex] < data[biggestIndex+1]) {
                        biggestIndex++;
                    }
                }
                if (data[k] < data[biggestIndex]) {
                    //交换他们
                    swap(data, k, biggestIndex);
                    // 将biggerIndex赋予k，开始while循环的下一次循环，重新保证k节点的值大于其左右子节点的值
                    k = biggestIndex;
                } else {
                    break;
                }
            }
        }
    }

     private static void swap(int[] data, int i, int j) {
        int temp = data[i];
        data[i] = data[j];
        data[j] = temp;
    }
```







