---
title: java里的排序-Comparator（三）
date: 2017-12-07 10:16:55
tags: [java]
categories: 排序算法
---
## 前言
上一篇研究了List.sort()采用的老版归并排序方法-legacyMergeSort，今天研究一下经过复杂优化的排序方法-TimSort，算法有点复杂，慢慢磨。思路最重要。
## 源码
直接进入TimSort.sort方法：
```
static <T> void sort(T[] a, int lo, int hi, Comparator<? super T> c,
                         T[] work, int workBase, int workLen) {
        assert c != null && a != null && lo >= 0 && lo <= hi && hi <= a.length;

        int nRemaining  = hi - lo;
        if (nRemaining < 2)
            return;  // Arrays of size 0 and 1 are always sorted

        // If array is small, do a "mini-TimSort" with no merges
        if (nRemaining < MIN_MERGE) {
            int initRunLen = countRunAndMakeAscending(a, lo, hi, c);
            binarySort(a, lo, hi, lo + initRunLen, c);
            return;
        }

        /**
         * March over the array once, left to right, finding natural runs,
         * extending short natural runs to minRun elements, and merging runs
         * to maintain stack invariant.
         */
        TimSort<T> ts = new TimSort<>(a, c, work, workBase, workLen);
        int minRun = minRunLength(nRemaining);
        do {
            // Identify next run
            int runLen = countRunAndMakeAscending(a, lo, hi, c);

            // If run is short, extend to min(minRun, nRemaining)
            if (runLen < minRun) {
                int force = nRemaining <= minRun ? nRemaining : minRun;
                binarySort(a, lo, lo + force, lo + runLen, c);
                runLen = force;
            }
            // Push run onto pending-run stack, and maybe merge
            ts.pushRun(lo, runLen);
            ts.mergeCollapse();

            // Advance to find next run
            lo += runLen;
            nRemaining -= runLen;
        } while (nRemaining != 0);

        // Merge all remaining runs to complete sort
        assert lo == hi;
        ts.mergeForceCollapse();
        assert ts.stackSize == 1;
    }
```
<!--more-->
初始参数：
目标数组T[] a
待排序的第一个元素int lo
带排序的最后一个元素int hi
排序规则comparator c
我们先来通过源码理清是如何实现的排序的，再来分析TimeSort的实现原理。
### 1.校验是否满足binarySort规则
```
int nRemaining  = hi - lo;
        if (nRemaining < 2)
            return;
        if (nRemaining < MIN_MERGE) {
            int initRunLen = countRunAndMakeAscending(a, lo, hi, c);
            binarySort(a, lo, hi, lo + initRunLen, c);
            return;
        }
```
判断中待排序数据元素：
    （1）当元素数量为0或者1时，则不必排序
    （2）当元素数量小于MIN_MERGE（为常量32）时，采用binarySort排序。（稍后在分析）
### 2.分配空间
构造TimeSort实例，初始化分片的内存空间和归并时的临时空间，这里分配空间是也根据目标数组的长度做了最适合的分配。
```
 TimSort<T> ts = new TimSort<>(a, c, work, workBase, workLen);
  
  private TimSort(T[] a, Comparator<? super T> c, T[] work, int workBase, int workLen) {
        this.a = a;
        this.c = c;
        int len = a.length;
        int tlen = (len < 2 * INITIAL_TMP_STORAGE_LENGTH) ?
            len >>> 1 : INITIAL_TMP_STORAGE_LENGTH;
        if (work == null || workLen < tlen || workBase + tlen > work.length) {
            @SuppressWarnings({"unchecked", "UnnecessaryLocalVariable"})
            T[] newArray = (T[])java.lang.reflect.Array.newInstance
                (a.getClass().getComponentType(), tlen);
            tmp = newArray;
            tmpBase = 0;
            tmpLen = tlen;
        }
        else {
            tmp = work;
            tmpBase = workBase;
            tmpLen = workLen;
        }
        int stackLen = (len <    120  ?  5 :
                        len <   1542  ? 10 :
                        len < 119151  ? 24 : 49);
        runBase = new int[stackLen];
        runLen = new int[stackLen];
    }

```
### 3.计算最小分片长度
通过minRunLength计算最小分片长度，低于这个长度，采用binarySort排序
```
int minRun = minRunLength(nRemaining);
```
### 4.do while
do while循环里每次获取一个升序的分片长度，判断该长度是否小于最小分片长度，小于最小长度，使用binarySort排序。
分析一下binarySort排序：
```
private static <T> void binarySort(T[] a, int lo, int hi, int start,
                                       Comparator<? super T> c) {
        assert lo <= start && start <= hi;
        if (start == lo)
            start++;
        for ( ; start < hi; start++) {
            T pivot = a[start];

            // Set left (and right) to the index where a[start] (pivot) belongs
            int left = lo;
            int right = start;
            assert left <= right;
            /*
             * Invariants:
             *   pivot >= all in [lo, left).
             *   pivot <  all in [right, start).
             */
            while (left < right) {
                int mid = (left + right) >>> 1;
                if (c.compare(pivot, a[mid]) < 0)
                    right = mid;
                else
                    left = mid + 1;
            }
            assert left == right;

            /*
             * The invariants still hold: pivot >= all in [lo, left) and
             * pivot < all in [left, start), so pivot belongs at left.  Note
             * that if there are elements equal to pivot, left points to the
             * first slot after them -- that's why this sort is stable.
             * Slide elements over to make room for pivot.
             */
            int n = start - left;  // The number of elements to move
            // Switch is just an optimization for arraycopy in default case
            switch (n) {
                case 2:  a[left + 2] = a[left + 1];
                case 1:  a[left + 1] = a[left];
                         break;
                default: System.arraycopy(a, left, a, left + 1, n);
            }
            a[left] = pivot;
        }
    }
```
我们可以看到binarySort对一个升序的分片进行了扩展，使它扩展到最小分片长度，并使用二分法对扩展分片进行排序。进入binarySort时，目标数组从lo到start是有序的，只需要将start到hi的元素通过二分法定位插入到已有序的序列中，这样整个从lo到hi就有序了。
接着通过runBase和runLen记录该分片的起始位置和长度
```
    ts.pushRun(lo, runLen);

    private void pushRun(int runBase, int runLen) {
        this.runBase[stackSize] = runBase;
        this.runLen[stackSize] = runLen;
        stackSize++;
    }

```
然后就可以尝试着将多个分片进行归并了：
```
    ts.mergeCollapse();

    lo += runLen;
    nRemaining -= runLen;
```
我们来看mergeCollapse这个函数，这是TimSort的核心算法
```
 private void mergeCollapse() {
        while (stackSize > 1) {
            int n = stackSize - 2;
            if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1]) {
                if (runLen[n - 1] < runLen[n + 1])
                    n--;
                mergeAt(n);
            } else if (runLen[n] <= runLen[n + 1]) {
                mergeAt(n);
            } else {
                break; // Invariant is established
            }
        }
    }
```
这里有两个条件的判断：当多个分片长度不满足以下条件是，合并分片被执行：
```
  1.runLen[i - 3] > runLen[i - 2] + runLen[i - 1]    
  2.runLen[i - 2] > runLen[i - 1]
```
我们可以这么理解，当条件1不被满足时，runLen[i - 2] ，runLen[i - 1]合并，合并之后，那么条件2也就不能被满足，继续合并。
继续看mergeAt的归并过程：
```
private void mergeAt(int i) {
        assert stackSize >= 2;
        assert i >= 0;
        assert i == stackSize - 2 || i == stackSize - 3;

        int base1 = runBase[i];
        int len1 = runLen[i];
        int base2 = runBase[i + 1];
        int len2 = runLen[i + 1];
        assert len1 > 0 && len2 > 0;
        assert base1 + len1 == base2;

        /*
         * Record the length of the combined runs; if i is the 3rd-last
         * run now, also slide over the last run (which isn't involved
         * in this merge).  The current run (i+1) goes away in any case.
         */
        runLen[i] = len1 + len2;
        if (i == stackSize - 3) {
            runBase[i + 1] = runBase[i + 2];
            runLen[i + 1] = runLen[i + 2];
        }
        stackSize--;

        /*
         * Find where the first element of run2 goes in run1. Prior elements
         * in run1 can be ignored (because they're already in place).
         */
        int k = gallopRight(a[base2], a, base1, len1, 0, c);
        assert k >= 0;
        base1 += k;
        len1 -= k;
        if (len1 == 0)
            return;

        /*
         * Find where the last element of run1 goes in run2. Subsequent elements
         * in run2 can be ignored (because they're already in place).
         */
        len2 = gallopLeft(a[base1 + len1 - 1], a, base2, len2, len2 - 1, c);
        assert len2 >= 0;
        if (len2 == 0)
            return;

        // Merge remaining runs, using tmp array with min(len1, len2) elements
        if (len1 <= len2)
            mergeLo(base1, len1, base2, len2);
        else
            mergeHi(base1, len1, base2, len2);
    }
```
在对入参做了一些常规校验以后，更新了存储分片信息数组，因为两个分片已经有序，所以可以采用一些小技巧完成合并。一直贴代码还是不好，我们继续看gallopLeft和gallopRight，
可以理解的是gallopRight通过二分法，找到了分片2的头在分片1的位置k，gallopLeft则通过二分法，找到分片1的尾在分片2的位置len2，这样我们只需要对a[base1+k]~a[base1+len1]与a[base2]~a[base2+len2]进行归并算法，新的大分片的头拷贝分片1的剩余的元素，尾拷贝分片2剩余的元素。
