---
title: 理解KMP回溯
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-03 15:28:02
password:
summary:
tags:
categories: 数据结构
---

#### 理解KMP回溯

相信大家都看过KMP算法，但是对于它的回溯确是难以理解。我们先来看一下KMP中的next数组生成代码：

```java
//用于生成next数组
private static int[] get_next(String target){
    int[] next = new int[target.length()];
    next[0] = -1;
    int i = 0, j = -1;
    while(i < target.length() - 1){
        if (j == -1 || target.charAt(i) == target.charAt(j)) {
            ++i;
            ++j;
            next[i] = j;
        } else {
            j = next[j];
        }
    }
    return next;
}
```
- 其中数组的next中的值计算方式是：

next[j] = Max{k | 1<k<j,且‘p1p2...pk’=‘p(j-k)...p(j - 1)’}

- ##### 概念

*简单来说next[j]表示的就是两个相等的字符串的长度，这两个字符串分别是从头开始记的长度为next[j]的和以next[j]的前一个字符结尾的长度为next[j]。*

- ##### 例子

例如：字符串"ababaaaba" next = [-1,0,0,1,2,3,1,1,2]
其中的回溯环节就是从next[5] = 3 到 next[6] = 1;

其中next[5]时：是"ababa"中前缀"aba"与后缀"aba"的长度，当i = 6时，"ababaa"中"a"不等于"b",所以回溯到j = next[j],其中j为现在next[5]的值。

- ##### 理解

**我开始也是很不明白为什么就可以直接回到j = next[next[5]] = 0处开始向后比较，后来仔细研究发现原因是，通过前面的比较它已经排除了所有的前缀字符串等于后缀字符串的长度大于回溯到当前j的可能 。**
就拿上面的“ababa”到“ababaa”举例：
其实我们想不通的无非就是它是怎么排除“aba”!="baa"转而直接去判断前缀“ab”是否等于后缀“aa”,后来我仔细分析才发现因为如果前面的“aba” = "baa"要成立，必须有“前缀ab”等于后缀“ba”,而得到next[5]=3的时候已经隐式的得到的第一个“ba”等于第二个“ba”(当时是“aba” = "aba")
从而有“aba”中三个值都应该相等，与前面矛盾。可能你早就看不懂我在说什么了，来一点数学表达式比较实际：

- ##### 数学证明

②开始有p1p2....pj = pi - j ....pi-1，可以得出pj = pj-1  j = 1,2,...
假设 next[j] = k 就有 p1p2...pk = pj-k ...pj-1	k = 1,2...
若加入pi != pj + 1,则需要回溯到判断pk 是否等于pj;
首先证明：pi-j+1...pi ！= p1p2...pj,反证：假设：pi-j+1...pi = p1p2...p，又p1p2....pj = pi - j ....pi-1
所以有pi - j ....pi-1 =pi-j+1...pi ,得到pi-j=pi-j+1=...=pi;与前面矛盾，所以有pi-j+1...pi ！= p1p2...pj
同理可以得出pi-j+2...pi ！= p1p2...pj-1  。。。。。pi-j+k...pi ！= p1p2...pj-k+1  。。。。。
所以可以直接回溯到j = next[j]继续向后判断

KMP完整代码

```java
private static int[] get_next(String target){
    int[] next = new int[target.length()];
    next[0] = -1;
    int i = 0, j = -1;
    while(i < target.length() - 1){
        if (j == -1 || target.charAt(i) == target.charAt(j)) {
            ++i;
            ++j;
            next[i] = j;
        } else {
            j = next[j];
        }
    }
    return next;
}

int kmp(String s, String pattern) {
    int i = 0,j = 0;
    int slen = s.length(), plen = pattern.length();
    int[] next = get_next(pattern);
    while (i < slen && j < plen) {
        if (s.charAt(i) == pattern.charAt(j)) {
            i++;
            j++;
        } else {
            if (next[j] == -1) {
                i++;
                j = 0;
            } else {
                j = next[j];
            }
        }
        if (j == plen) {
            return i - j;
        }
    }
    return -1;
}
```