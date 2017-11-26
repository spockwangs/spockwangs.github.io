---
layout: post
title: 比特操纵技巧
comments: true
categories:
- programming practice
---

## 计算数据二进制表示中1的个数

比如3（11<sub>2</sub>）有2个，4（100<sub>2</sub>）有1个，0（0<sub>2</sub>）有0个。
<!--more-->

最直接的方法当然是每次将其右移一位，若移出的比特为1则加1。它时间复杂度正比于数据宽度。还有一种更简单的方法如下所示：
``` c
int count_ones(unsigned int x)
{
    int sum = 0;           // Count how many ones in x.
    while (x != 0) {
        sum++;
        x = x & (x-1);     // 这句是关键。
    }
    return sum;
}
```
注意上面第6行代码是关键。它的作用是将x最右边的比特1置为0，直到x变为0。它的时间复杂度正比于x中比特1的个数。

当然，还有更快的算法。我们可以将问题的答案存在一张表里，通过查表法乃解决，相
当于通过空间来换取时间。比如，我们可以把一个字节的所有情况（共256个）存在一
张表格中，这样我们可以每次处理一个字节，这比上面的算法要快得多，当然也要更多
的空间。

另见：[智慧碰撞（求二进制数中1的个数）](http://www.msra.cn/Articles/ArticleItem.aspx?Guid=edb3a02b-6d5e-42c7-b2a5-4ae4a18f4254#).
