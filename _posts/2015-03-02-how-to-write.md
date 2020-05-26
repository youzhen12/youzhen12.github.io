---
layout: post
title: 阿里笔试题取物资的go语言解法
date: 2020-5-14
categories: blog
tags: [算法]
description: 快速排序。
---

#

# 阿里笔试题取物资

## 题目

在一个二维地图上有很多军营，每个军营的坐标为（x,y），有一条平行于y轴且长度无限的路，补给车经过这条路时会将每个军营的物资放到离他们军营最近的路边(这条马路所在的位置使得所有军营到达马路的距离和最小)，请问所有军营到马路的最短距离和是多少
样例
输入:
[[-10,0],[0,0],[10,0]]
输出:20
说明:假设路在x=0上，最短路径和为10+0+10=20

## 分析

这个题目实际上就是求x坐标的中位值

1.首先提取出x坐标的数组
2.找出中位数，这里使用快速排序算法
3.做差值

## 代码

```
import (
	"math"
)
func quickSort(begin int,end int,arr []int){
	if (begin - end == 1 || begin - end == 0) {
		return
	}
	mark := begin
	for i := (begin + 1); i < end; i++ {
		if arr[i]<arr[mark] {
			tmp := arr[i]
			arr[i] = arr[mark + 1]
			arr[mark + 1] = arr[mark]
			arr[mark] = tmp
			mark++
		}
	}
	quickSort(begin, mark, arr);
	quickSort(mark + 1, end, arr);
}
func Fetchsupplies(coordinates [][]int)  int{
	b := len(coordinates)
	arr := make([]int, b)
	for k, v:=range coordinates{
		arr[k]=v[0]
	}
	quickSort(0,b,arr)
	mid := arr[b/2]
	var s float64
	for _, v:=range arr{
		s=s+(math.Abs(float64(v-mid)))
	}
	return int(s)
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200513183812862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMzk0MzIx,size_16,color_FFFFFF,t_70)

## 快速排序

从数列中挑出一个元素，称为 “基准”（pivot）;

重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作；

递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序；

关于第二步的具体代码步骤

取第一个数作为基准数
每个数和第一个数对比，大的不变，小的和标志位交换位置，这时候标志位加+1，这样如果有比基准数大的数，就会被放到标志位的地方（如果前面没有比它大的数，则和它自己交换位置，即不变）这时候标志位加+1。最后交换标志位-1和第一个数，标志位减一即为标志数，所对应的值为基准值。

还有另外一种写法，开始没有基准数
先取标志位为0，从一开始每个数和标志位对比，大则不变，小则这位=标志位+1，标志位+1=标志位，标志位=这位，这时候标志位加一。最后标志位所在的数就是基准数。
就是一个位移操作，把小于标志数插入标志位前面，然后把标志位后移。










