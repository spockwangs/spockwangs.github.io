---
layout: post
title: 随机采样
categories:
- algorithm
---

## 问题描述

从一个预先不知道其大小的集合中随机选出`K`个数字。

## 算法

_Input:_ `K`，和一系列数字。

_Output:_随机选出的`K`个数字。
```
R := [ ];     # 初始化为一个空数组。
N := 0;       # 已经考虑的数字的个数。
for next number i
begin
    N := N + 1;
    if size(R) &lt; K
    then
        R := [ R, i ];      # 将数字i添加到数组R中。
    else
        j := randint(0, N);
        if j &lt; K
        then
            R[j] = i;
        end
    end
end
```
其中`randint(0, N)`等概率地返回`[0, N)`中的一个整数。结束后`R`中的`K`个数字就是我们需要的。
