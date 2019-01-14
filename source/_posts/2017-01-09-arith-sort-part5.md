---
title: 图解排序算法（五）之快速排序和排序总结  
date: 2017-01-09 22:43:54
tags: 算法
---

### 基本思想

快速排序是冒泡排序的改进版，也是最好的一种内排序，在很多面试题中都会出现，也是作为程序员必须掌握的一种排序方法。  
思想:	   
1.在待排序的元素任取一个元素作为基准(通常选第一个元素，但最的选择方法是从待排序元素中随机选取一个作为基准)，称为基准元素；  
2.将待排序的元素进行分区，比基准元素大的元素放在它的右边，比其小的放在它的左边；  
3.对左右两个分区重复以上步骤直到所有元素都是有序的。  
所以我是把快速排序联想成东拆西补或西拆东补，一边拆一边补，直到所有元素达到有序状态。

<!--more-->

下面再看看示图理解下吧：

<img src="/img/201701/quickSort.png" alt="quick sort png" style="width: 600px;">


对元素5两边的元素也重复以上操作，直到元素达到有序状态。


### Java实现

```java
    //快速排序
    public static void quickSort(int arr[],int _left,int _right){
        int left = _left;
        int right = _right;
        int temp ;
        if(left < right){   //待排序的元素至少有两个的情况
            temp = arr[left];  //待排序的第一个元素作为基准元素
            while(left != right){   //从左右两边交替扫描，直到left = right

                while(right > left && arr[right] >= temp)
                    right --;        //从右往左扫描，找到第一个比基准元素小的元素
                arr[left] = arr[right];  //找到这种元素arr[right]后与arr[left]交换

                while(left < right && arr[left] <= temp)
                    left ++;         //从左往右扫描，找到第一个比基准元素大的元素
                arr[right] = arr[left];  //找到这种元素arr[left]后，与arr[right]交换

            }
            arr[right] = temp;    //基准元素归位
            quickSort(arr,_left,left-1);  //对基准元素左边的元素进行递归排序
            quickSort(arr, right+1,_right);  //对基准元素右边的进行递归排序
        }
    }
```

时间复杂度O(nlogn）

快速排序在序列中元素很少时，效率将比较低，不如插入排序，因此一般在序列中元素很少时使用插入排序，这样可以提高整体效率。

### 排序算法总结

如图

<img src="/img/201701/allSort.png" alt="all sort png" style="width: 600px;">







