---
title: sort
date: 2019-06-12 23:33:18
description: 排序作为最基础的算法, 应用更是十分广泛, 从刚开始接触编程开始一直到学习算法始终伴随着我们, 把排序完全理解, 常规的算法与数据结构基本理解了一半了吧(有吗?, 差不多吧). 这里对排序算法进行一次整理, 从最简单的冒泡排序到高端的快排和堆排. 均以从小到大的顺序举例.
tags:
    排序整理
categories:
    算法
mathjax:
    true
---



# <center/><font size = 10> 冒泡排序</font></center>

原理十分简单, 对于一个长度为$ n$数组, 遍历$n-1$次, 每次遍历时判断相邻两个值大小, 如果不满足顺序则进行交换, 这样, 第$i$次遍历数组中第$i$大的元素就会被排序到导数第$i$的位置. 这样遍历$n-1$次后即完成排序. 时间复杂度为$O(n^2)$

code:

```c++
template<class T>
void bubbling_sort(T a[], int n)
{
    for(int i=0; i<n-1; i++)
    {
        for(int j=0; j<n-i-1; j++)
        {
            if(a[j]>a[j+1])
            {
                swap(a[j], a[j+1]);
            }
        }
    }
}
```



# <center/><font size = 10> 插入排序</font></center>

插入排序原理也十分简单, 与冒泡类似, 对与一个长度为$n$的数组, 进行$n-1$次遍历, 每次选取对大的未排序完成的部分中最大的与最后一个未完成排序的元素交换, 这样未完成排序的元素将减少一个. 复杂度为$O(n^2)$.

code:

```c++
template<class T>
void insert_sort(T a[], int n)
{
    for(int i=0; i<n-1; i++)
    {
        int index = 0;
        for(int j=1; j<n-i; j++)
        {
            if(a[index]<a[j])
                index = j;
        }
        swap(a[index], a[n-i-1]);
    }
}
```



# <center/><font size = 10> 归并排序</font></center>

归并排序使用到了分治的思想, 将一个数组分为两部分, 并分别将两部分排序完成, 而后只用对两个排序好的子序列进行合并即可. 对两个子序列的合并操作也使用相同的策略, 递归的进行. 在进行合并的时候, 需要将原来的数组排序结果保存到另一个数组中, 空间复杂度为$O(n)$.  归并排序时间复杂度为$O(nlog(n))$. 典型应用为求逆序对数.

code:

```c++
template<class T>
void merge_sort(T dst[], T src[], int start, int end)
{
    if(end - start < 1)
        return;
    if(end - start == 1)
    {
        if(src[start] > src[end])
        {
            swap(src[end], src[start]);
        }
        dst[end] = src[end];
        dst[start] = src[end];
        return;
    }
    int mod = (start+end)/2;
    merge_sort(dst, src, start, mod);
    merge_sort(dst, src, mod+1, end);
    int p=start, q=mod+1, k=start;
    while(p!=mod+1 && q!=end+1)
    {
        if(src[p]<src[q])
            dst[k++] = src[p++];
        else
        {
            dst[k++] = src[q++];
        }
    }
    if(p==mod+1)
    {
        for(k; k<end+1; k++)
        {
            dst[k] = src[q++];
        }
    }
    else
    {
        for(k; k<end+1; k++)
        {
            dst[k] = src[p++];
        }
    }
    for(int i=start; i<end+1; i++)
    {
        src[i] = dst[i];
    }
}
```



# <center/><font size = 10>快排</font></center>

快排也使用的是分治的策略, 不过与归并不同的是, 归并指定了每次分割的位置(中间). 快排并不指定中间位置而是根据当前子序列的第一(最后一个)个元素的大小进行划分, 分成大于它的和小于它的两部分, 而后对这两部分再进行分类.  对这两部分分类也是采取递归的策略.  快排最差情况下时间复杂度为$O(n^2)$, 期望时间复杂度为$O(nlog(n))$. 典型应用为线性时间求取数组中第K大数.

code:

```
void quite_sort(T a[], int start, int end)
{
    if(end-start < 1)
        return;
    if(end-start == 1)
    {
        if(a[start]>a[end])
            swap(a[start], a[end]);
        return;
    }
    int key = a[start], i=start, j=end;
    while (i<j)
    {
        while(i<j &&  key < a[j])
            j--;
        a[i] = a[j];
        while(i<j && key > a[i])
            i++;
        a[j] = a[i];
    }
    a[i] = key;
    quite_sort(a, start, i-1);
    quite_sort(a, i+1, end);
}
```



# <center/><font size = 10>堆排</font></center>

堆排使用了二叉树, 在二叉树中维护一个性质, 即大顶堆或者小顶堆. 首先我们要将一个数组构建成为一个满足大顶堆或者小顶堆的一个完全二叉树. 而后我们每一次将大顶堆中的堆顶元素与未排序完成的堆的最后一个元素交换位置, 同时从堆中删除堆最后一个元素(即之前的最大值). 同时要保证交换完成之后的堆保持原来大顶堆的性质. 逐步进行, 直到将整个堆完全删除, 即得到排序完成的数组.

## <font size=6>从数组构建满足要求的完全二叉树:</font>

对于长度为$n$的元素, 只有索引小于等于int(n/2)的位置有叶节点. 对于非叶节点来说, 其自然满足大顶堆性质, 对于存在页节点的元素来说, 我们要使其满足大顶堆性质则要进行判断是否满足父节点大于两个子节点. 如果满足, 则该节点满足条件, 不用有任何操作(注意: 我们在访问某个节点使其满足大顶堆性质时, 即子树必然已经满足该性质, 这是因为我们的操作是自底向上进行的, 越底部越先被满足). 如果不满足条件则要进行调整. 但调整后我们可能会破坏子树的大顶堆性质, 此时我们应该顺着可能被破坏大顶堆性质的子数进行搜索, 维持大顶堆性质. 时间复杂度为$O(nloh(n))$.

## <font size=6>堆顶与最后一个值进行交换:</font>

当交换完成后, 大顶堆的性质很有可能被破坏, 此时我们也要进行操作以维护大顶堆性质. 从堆顶自顶向下进行搜索, 将不满足大顶堆性质的节点进行交换. 

code:

```c++
// 维护大顶堆性质
template<class T>
void remain_max_heap(T a[], int index, int length)
{
    if(index >= length)
        return;
    while(index*2<=length)
    {
        if(index*2 +1 <=length)
        {
            if(a[index]<a[index*2] && a[index*2+1]<a[index*2])
            {
                swap(a[index], a[index*2]);
                index = index*2;
            }
            else
            {
                if(a[index]<a[index*2+1] && a[index*2]<a[index*2+1])
                {
                    swap(a[index], a[index*2+1]);
                    index = index*2+1;
                }
                else
                {
                    break;
                } 
            }
            
        }
        else
        {
            if(a[index]<a[index*2])
            {
                swap(a[index], a[index*2]);
            }
            break;
        }
    }
}

template<class T>
void heap_sort(T a[], int length)
{
    // 将数组转化为满足大顶堆性质的完全二叉树
    for(int i=length/2; i>-1; i--)
    {
        remain_max_heap(a, i, length);
    }
    // 排序, 取堆顶元素进行交换.
    for(int i=0; i<length; i++)
    {
        swap(a[0], a[length-i]);
        remain_max_heap(a, 0, length-i-1);
    }
}
```

