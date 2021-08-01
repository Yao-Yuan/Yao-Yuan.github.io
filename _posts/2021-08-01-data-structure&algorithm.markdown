---
layout: post
title:  "Data Structure & Algorithm"
date:   2021-08-01 21:58:00 +0200
categories: jekyll update
---
系统学习，也复习一下 yeahhhh.

数据结构：
- 线性结构：数据之间一对一关系, e.g. list, stack
    - 顺序
    - 链式
- 非线性结构
    - 多维数组
    - 表
    - 树
    - 图

时间和空间复杂度（取最坏情况）
- 计算时间复杂度
    - 计算时间频度
    - 忽略常量项
    - 忽略低次项
    - 忽略系数（和多少次方有关，e.g.3n^3 6n^3 还是有些区别）

- 常见时间复杂度： O(1) < O（log_2n） < O(n) < O(nlog_2n) < O(n^2) < O(n^k) < O(2^n) 
    - 常数阶O(1): 无论代码多少行，只要没有循环就是常数阶
    - 对数阶O(log_2n)
    - 线性阶O(n)：for循环执行n遍
    - 线性对数阶O(nlogn): 将时间复杂度为O(logn)的代码循环n遍
    - 平方阶O（n^2）:把O(n)的代码再嵌套循环一遍
    - K次方阶O(n^k)