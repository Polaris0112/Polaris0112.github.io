---
title: Python实现八大排序算法
date: 2018-03-22
categories: Python八大算法
tags: 
- Python

---
本帖子记录的是整理基于Python的八大排序算法。

本文用Python实现了插入排序、希尔排序、冒泡排序、快速排序、直接选择排序、堆排序、归并排序、基数排序。


## 冒泡排序

### 原理

冒泡排序(Bubble Sort)是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。


### 步骤

冒泡排序算法的运作如下：

- 比较相邻的元素。如果第一个比第二个大，就交换他们两个。
- 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数。
- 针对所有的元素重复以上的步骤，除了最后一个。
- 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。


### 代码

```python
def bubble_sort(list):
    length = len(list)
    # 第一级遍历
    for index in range(length):
        # 第二级遍历
        for j in range(1, length - index):
            if list[j - 1] > list[j]:
                # 交换两者数据，这里没用temp是因为python 特性元组。
                list[j - 1], list[j] = list[j], list[j - 1]
    return list
```


优化
```python
def bubble_sort_flag(list):
    length = len(list)
    for index in range(length):
        # 标志位
        flag = True
        for j in range(1, length - index):
            if list[j - 1] > list[j]:
                list[j - 1], list[j] = list[j], list[j - 1]
                flag = False
        if flag:
            # 没有发生交换，直接返回list
            return list
    return list
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/bubble-sort.py)


## 选择排序

### 原理

选择排序（Selection sort）是一种简单直观的排序算法。它的工作原理大致是将后面的元素最小元素一个个取出然后按顺序放置。


### 步骤

- 在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，
- 再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。
- 重复第二步，直到所有元素均排序完毕。


### 代码

```python
def selection_sort(list):
    n=len(list)
    for i in range (0,n):
        min = i
        for j in range(i+1,n):
            if list[j]<list[min]:
                min=j
                list[min],list[i]=list[i],list[min]
    return list
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/select-sort.py)



## 插入排序

### 原理

插入排序（Insertion Sort）是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。


### 步骤

- 从第一个元素开始，该元素可以认为已经被排序
- 取出下一个元素，在已经排序的元素序列中从后向前扫描
- 如果该元素（已排序）大于新元素，将该元素移到下一位置
- 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
- 将新元素插入到该位置后
- 重复步骤2~5


### 代码

```python
def insert_sort(list):
    n = len(list)
    for i in range(1, n):
        # 后一个元素和前一个元素比较
        # 如果比前一个小
        if list[i] < list[i - 1]:
            # 将这个数取出
            temp = list[i]
            # 保存下标
            index = i
            # 从后往前依次比较每个元素
            for j in range(i - 1, -1, -1):
                # 和比取出元素大的元素交换
                if list[j] > temp:
                    list[j + 1] = list[j]
                    index = j
                else:
                    break
            # 插入元素
            list[index] = temp
    return list
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/insert-sort.py)



## 希尔排序

### 原理

希尔排序，也称递减增量排序算法，是插入排序的一种更高效的改进版本。希尔排序是非稳定排序算法。
希尔排序是基于插入排序的以下两点性质而提出改进方法的：
插入排序在对几乎已经排好序的数据操作时，效率高，即可以达到线性排序的效率
但插入排序一般来说是低效的，因为插入排序每次只能将数据移动一位。


### 步骤

每次以一定步长(就是跳过等距的数)进行排序，直至步长为1.


### 代码

```python
def shell_sort(list):
    n = len(list)
    # 初始步长
    gap = round(n / 2)
    while gap > 0:
        for i in range(gap, n):
            # 每个步长进行插入排序
            temp = list[i]
            j = i
            # 插入排序
            while j >= gap and list[j - gap] > temp:
                list[j] = list[j - gap]
                j -= gap
            list[j] = temp
        # 得到新的步长
        gap = round(gap / 2)
    return list
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/shell-sort.py)



## 归并排序

### 原理

归并操作(归并算法)，指的是将两个已经排序的序列合并成一个序列的操作。归并排序算法依赖归并操作。


### 步骤

迭代法

- 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
- 设定两个指针，最初位置分别为两个已经排序序列的起始位置
- 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
- 重复步骤3直到某一指针到达序列尾
- 将另一序列剩下的所有元素直接复制到合并序列尾

递归法

假设序列共有n个元素：
- 将序列每相邻两个数字进行归并操作，形成 {\displaystyle floor(n/2)} floor(n/2)个序列，排序后每个序列包含两个元素
- 将上述序列再次归并，形成 {\displaystyle floor(n/4)} floor(n/4)个序列，每个序列包含四个元素
- 重复步骤2，直到所有元素排序完毕


### 代码

```python
# 递归法
def merge_sort(list):
    # 认为长度不大于1的数列是有序的
    if len(list) <= 1:
        return list
    # 二分列表
    middle = len(list) // 2
    left = merge_sort(list[:middle])
    right = merge_sort(list[middle:])
    # 最后一次合并
    return merge(left, right)
# 合并
def merge(left, right):
    l,r=0,0
    result=[]
    while l<len(left) and r<len(right):
        if left[l] <right[r]:
            result.append(left[l])
            l+=1
        else:
            result.append(right[r])
            r +=1
        reslut +=left[l:]
        result+=right[r:]                
    return result
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/merge-sort.py)




## 快速排序

### 原理

快速排序使用分治法（Divide and conquer）策略来把一个序列（list）分为两个子序列（sub-lists）。


### 步骤

- 从数列中挑出一个元素，称为”基准”（pivot），
- 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
- 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序。


### 代码
```python
# -*- coding:utf-8 -*-
def kp(arr,i,j):#快排总函数
                #制定从哪开始快排
    if i<j:
        base=kpgc(arr,i,j)
        kp(arr,i,base)
        kp(arr,base+1,j)
def kpgc(arr,i,j):#快排排序过程
    base=arr[i]
    while i<j:
        while i<j and arr[j]>=base:
            j-=1
        while i<j and arr[j]<base:
            arr[i]=arr[j]
            i+=1
            arr[j]=arr[i]
        arr[i]=base
    return i
ww=[3,2,4,1,59,23,13,1,3]
print ww
kp(ww,0,len(ww)-1)
print ww
```

下面这段代码出自《Python cookbook 第二版》传说中的三行实现python快速排序。

```python
def qsort(arr):
    if len(arr) <= 1:
        return arr
    else:
        pivot = arr[0]
        return qsort([x for x in arr[1:] if x < pivot]) + \
               [pivot] + \
               qsort([x for x in arr[1:] if x >= pivot])
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/quick-sort.py)



## 堆排序

### 原理

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆积是一个近似完全二叉树的结构，并同时满足堆积的性质：即子结点的键值或索引总是小于（或者大于）它的父节点。


### 步骤

- 创建最大堆:将堆所有数据重新排序，使其成为最大堆
- 最大堆调整:作用是保持最大堆的性质，是创建最大堆的核心子程序
- 堆排序:移除位在第一个数据的根节点，并做最大堆调整的递归运算


### 代码
```python
# -*- coding:utf-8 -*-
def head_sort(list):
	length_list = len(list)
	first=int(length_list/2-1)
	for start in range(first,-1,-1):
		max_heapify(list,start,length_list-1)
	for end in range(length_list-1,0,-1):
		list[end],list[0]=list[0],list[end]
		max_heapify(list,0,end-1)
	return list

def max_heapify(ary,start,end):
	root = start
	while True:
		child = root*2 + 1
		if child > end:
			break
		if child + 1 <= end and ary[child]<ary[child+1]:
			child = child + 1
		if ary[root]<ary[child]:
			ary[root],ary[child]=ary[child],ary[root]
			root=child
		else:
			break
#测试：
list=[10,23,1,53,654,54,16,646,65,3155,546,31]
print head_sort(list)
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/heap-sort.py)




## 计数排序

### 原理

当输入的元素是n个0到k之间的整数时，它的运行时间是Θ(n + k)。计数排序不是比较排序，排序的速度快于任何比较排序算法。

由于用来计数的数组C的长度取决于待排序数组中数据的范围（等于待排序数组的最大值与最小值的差加上1），这使得计数排序对于数据范围很大的数组，需要大量时间和内存。例如：计数排序是用来排序0到100之间的数字的最好的算法，但是它不适合按字母顺序排序人名。但是，计数排序可以用在基数排序算法中，能够更有效的排序数据范围很大的数组。


### 步骤

- 找出待排序的数组中最大和最小的元素
- 统计数组中每个值为i的元素出现的次数，存入数组 C 的第 i 项
- 对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）
- 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1


### 代码

```python
def count_sort(list):
    min = 2147483647
    max = 0
    # 取得最大值和最小值
    for x in list:
        if x < min:
            min = x
        if x > max:
            max = x
    # 创建数组C
    count = [0] * (max - min +1)
    for index in list:
        count[index - min] += 1
    index = 0
    # 填值
    for a in range(max - min+1):
        for c in range(count[a]):
            list[index] = a + min
            index += 1
    return list
```

[测试代码](https://raw.githubusercontent.com/Polaris0112/Ops-Tools/master/selection-algorithm/count-sort.py)






