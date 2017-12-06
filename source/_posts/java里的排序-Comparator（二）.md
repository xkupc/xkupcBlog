---
title: java里的排序-Comparator（二）
date: 2017-12-06 15:46:07
tags: [java]
categories: 排序算法
---
### 前言
上一篇讲到了list集合排序用到的Cellection.sort使用了归并排序，今天我们来研究一下java里是怎么样实现的归并排序。这里使用jdk的版本是jdk1.8的版本。
### 源码
我们看到Cellection.sort回调了list的sort方法，该方法在java.util里面：
```
default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```
这里有调用了Array.sort方法进行排序，Array是什么样的存在呢。我们看这个类的说明：
```
 * This class contains various methods for manipulating arrays (such as
 * sorting and searching). This class also contains a static factory
 * that allows arrays to be viewed as lists.
```
我们稍稍可以窥探一点list集合底层的实现原理-数组实现。Array提供了数组的排序和搜索操作，同时提供了工厂方法让数组以列表的形式展示。
<!--more--->
继续往下走:
```
 public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) {
            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c);
            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);
        }
    }
```
这里有一个标示位,我们看看这个标示位含义：
```
 /**
     * Old merge sort implementation can be selected (for
     * compatibility with broken comparators) using a system property.
     * Cannot be a static boolean in the enclosing class due to
     * circular dependencies. To be removed in a future release.
     */
    static final class LegacyMergeSort {
        private static final boolean userRequested =
            java.security.AccessController.doPrivileged(
                new sun.security.action.GetBooleanAction(
                    "java.util.Arrays.useLegacyMergeSort")).booleanValue();
    }
```
是的，已经很明显了，旧版本的归并排序要在将来的版本淘汰了，这个标示位还是通过配置文件才生效的。知道这些之后，我们就想问那么legacyMergeSort和TimSort有什么区别呢。
TimSort在归并排序的基础上做了大量的优化，是一种复杂的排序算法，既然legacyMergeSort是老版本，那么相对低阶一点，OK,我们先从低阶学起。
### legacyMergeSort
直接进入legacyMergeSort里调用的mergeSort方法：
```
/**
     * Src is the source array that starts at index 0
     * Dest is the (possibly larger) array destination with a possible offset
     * low is the index in dest to start sorting
     * high is the end index in dest to end sorting
     * off is the offset into src corresponding to low in dest
     * To be removed in a future release.
     */
    @SuppressWarnings({"rawtypes", "unchecked"})
    private static void mergeSort(Object[] src,
                                  Object[] dest,
                                  int low, int high, int off,
                                  Comparator c) {
        int length = high - low;

        // Insertion sort on smallest arrays
        if (length < INSERTIONSORT_THRESHOLD) {
            for (int i=low; i<high; i++)
                for (int j=i; j>low && c.compare(dest[j-1], dest[j])>0; j--)
                    swap(dest, j, j-1);
            return;
        }

        // Recursively sort halves of dest into src
        int destLow  = low;
        int destHigh = high;
        low  += off;
        high += off;
        int mid = (low + high) >>> 1;
        mergeSort(dest, src, low, mid, -off, c);
        mergeSort(dest, src, mid, high, -off, c);

        // If list is already sorted, just copy from src to dest.  This is an
        // optimization that results in faster sorts for nearly ordered lists.
        if (c.compare(src[mid-1], src[mid]) <= 0) {
           System.arraycopy(src, low, dest, destLow, length);
           return;
        }

        // Merge sorted halves (now in src) into dest
        for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
            if (q >= high || p < mid && c.compare(src[p], src[q]) <= 0)
                dest[i] = src[p++];
            else
                dest[i] = src[q++];
        }
    }
```
我们可以看看这个方法的描述，很明显这是一个归并排序，但和归并排序有那么一些不一样，我们具体来分析分析。
首先src是目标集合dest的一个拷贝，初始偏移量off为0，可以看到这里对归并排序做了一些优化。
优化点1：第一个if：当集合的大小在INSERTIONSORT_THRESHOLD =7以下时，优先使用插入排序。
优化点2：int mid = (low + high) >>> 1;划分两个子序列使用移位运算符，右移移位，相当于除2,但比除法快
优化点3：判断序列是否已有序，有序则直接将子序列拷贝到目标集合
优化点4：直接拷贝整个集合代替递归分配的两个子序列，减少了递归的中的空间分配
优化点5：合并子序列的循环中，p,q分别代表两个子序列的起始值。当一个序列元素全部填充到目标集合后，直接将另一个序列剩余元素填充到目标集合。
### 归并排序优化
根据java里低阶的归并优化版本，将上一篇的归并排序进行优化
```
   private static final int INSERTSORT_THRESHOLD = 7;

    public int[] legacyMergeSore(int[] num, int left, int right) {
        //拷贝目标数组
        int[] cloneNum = num.clone();
        return mergeSort(cloneNum, num, left, right);
    }

    private int[] mergeSort(int[] cloneNum, int[] num, int left, int right) {
        int length = right - left;
        if (length < INSERTSORT_THRESHOLD) {
            //采用插入排序算法
            InsertSort insertSort = new InsertSort();
            insertSort.insertSort(num, left, right);
            return num;
        }
        int baseIndex = (left + right) >>> 1;
        mergeSort(num, cloneNum, left, baseIndex);
        mergeSort(num, cloneNum, baseIndex, right);
        merge(num, cloneNum, left, baseIndex, right);
        return num;
    }

    /**
     * 合并序列
     *
     * @param num
     * @param cloneNum
     * @param left
     * @param baseIndex
     * @param right
     */
    private void merge(int[] num, int[] cloneNum, int left, int baseIndex, int right) {
        //校验是否有序，因为baseIndex为第二个子序列的起始值，且两个子序列已有序
        if (cloneNum[baseIndex - 1] <= cloneNum[baseIndex]) {
            //两子序列之间已有序，直接拷贝到目标序列
            for (int i = left; i < right; i++) {
                num[i] = cloneNum[i];
            }
            return;
        }
        for (int j = left, p = left, q = baseIndex; j < right; j++) {
            if (q >= right || p < baseIndex && cloneNum[p] < cloneNum[q]) {
                num[j] = cloneNum[p++];
            } else {
                num[j] = cloneNum[q++];
            }
        }
    }

    public static void main(String[] args) {
        LegacyMergeSort legacyMergeSort = new LegacyMergeSort();
        int[] num = new int[]{6, 7, 7, 5, 10, 2, 8, 2, 9, 1, 3};
        legacyMergeSort.legacyMergeSore(num,0,num.length);
    }
```
有时间可以研究一下TimeSort，再来写笔记吧。