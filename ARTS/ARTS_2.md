# ARTS第一周

## Algorithm

[leetcode  寻找两个有序数组的中位数](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/)

>There are two sorted arrays nums1 and nums2 of size m and n respectively.
>
>Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).
>
>You may assume nums1 and nums2 cannot be both empty.
>
>Example 1:
>
>nums1 = [1, 3]
>nums2 = [2]
>
>The median is 2.0
>Example 2:
>
>nums1 = [1, 2]
>nums2 = [3, 4]
>
>The median is (2 + 3)/2 = 2.5

看到这个题目很容易联想到归并排序的marge过程，这里需要通过marge过程找到最中间的一个数（总数为奇数时）或者两个数（总数为偶数时）

```Go
func findMedianSortedArrays(nums1 []int, nums2 []int) float64 {
	i, j := 0, 0
	iMax := len(nums1)
	jMax := len(nums2)
	p, q := 0, 0
	if (iMax+jMax)%2 == 0 {
		q = (iMax + jMax) / 2
		p = q - 1
	} else {
		q = (iMax + jMax) / 2
		p = q
	}

	index := 0
	sum := 0
	value := 0
	for {
		switch {
		case i < iMax && j < jMax:
			if nums1[i] < nums2[j] {
				value = nums1[i]
				i++
			} else {
				value = nums2[j]
				j++
			}
		case i < iMax && j >= jMax:
			value = nums1[i]
			i++
		case i >= iMax && j < jMax:
			value = nums2[j]
			j++
		}

		if index == p {
			sum += value
		}

		if index == q {
			sum += value
			break
		}

		index++
	}

    return float64(sum) / 2.0
}
//执行时间32ms 内存消耗5.5MB
```

## Review

[Why Data Scientists love Gaussian?](https://towardsdatascience.com/why-data-scientists-love-gaussian-6e7a7b726859)
 这篇文章主要介绍高斯分布（正态分布）
 高斯分布的重要特点：
 1. 两个高斯分布的乘积是高斯分布
 2. 两个独立高斯随机变量的和是高斯分布
 3. 一个高斯分布与另一高斯的卷积是高斯分布的
 3. 高斯分布的傅里叶变换是高斯分布的

 对于一个高斯近似模型可能存在其它的复杂的多参数分布模型可以提供更好的近似，但是高斯模型依然是首选的，因为高斯模型是简洁的
 1. 均值(Mean)，中位数(Median)和众数(Mode)是相同的
 2. 可以通过两个参数均值和方差来描述整个分布

 ## Tip

vs提供的dumpbin工具可以很方便的帮助我们分析程序发布时会遇到的各种奇葩问题
**常用参数**
```c++
dumpbin /headers xxx.dll //查看dll header 常用区分32/64

dumpbin /dependents xxx.dll //查看dll依赖关系

dumpbin /exports xxx.dll //查看dll导出表 常用于解决LINK 200 无法找到特定符号 错误

```

 ## Share
LZW压缩算法

LZW压缩算法基于初始字典，逐步新增子串编码扩充字典，来实现压缩编码。  
关键点是发送双方基于相同的初始字典可以逐步建立起完全一样的子串字典。

**发送方**
1. 取一个待发送字符，与上一次发送的子串合并检查合并串是否在字典中，不在则扩充字典。记录待发子串在字典中的编码，直到待发子串不在字典中
2. 发送最后一次记录的待发子串编码
3. 重复直到数据发完

**接收方**

1. 接收到一个编码，查找字典解码（字典中不存在则错误）
2. 用上一次接收到的子串和当前接收子串扩充字典