# ARTS第三周

## Algorithm

>Given a string s, find the longest palindromic substring in s. You may assume that the maximum length of s is 1000.
>
>Example 1:
>
>Input: "babad"
>Output: "bab"
>Note: "aba" is also a valid answer.
>Example 2:
>
>Input: "cbbd"
>Output: "bb"

思路：通过中心点向两边扩张，偶数长度子串时中心点在两个字符之间，奇数长度子串时中心点为单个字符
时间复杂度：
$$\sum_{\mathclap{1\le x\le n/2}} x => O(n^2)$$

```go

func longestPalindrome(s string) string {
	if len(s) < 2 {
		return s
	}
	subBegin, subEnd := 0, 0
	for pos := 0; pos < 2*(len(s)-1); pos++ {
		begin, end := 0, 0
		if pos%2 == 0 {
			if pos == 0 {
				begin = 0
				end = 0
			} else {
				begin = pos/2 - 1
				end = pos/2 + 1
			}
		} else {
			begin = (pos - 1) / 2
			end = (pos + 1) / 2
		}

		for s[end] == s[begin] {
			if subEnd-subBegin < end-begin {
				subBegin = begin
				subEnd = end
			}
			if begin > 0 && end < len(s)-1 {
				begin--
				end++
			} else {
				break
			}
		}
	}
	return s[subBegin : subEnd+1]
}
//执行时间12ms
```

## Review
[Machine Lerning for Humans Part 2.1 Supervised Learning](https://medium.com/machine-learning-for-humans/supervised-learning-740383a2feab)
回归：预测一个连续值
特征(feature)应满足数字的(numerical)或明确的(categorical)
训练集和测试集
supervised learning algorithms. 监督学习算法

## Tip

C++ std::shared_ptr和std::weak_ptr配合使用
最近在使用std::shared_ptr来管理资源的时候遇到了两个对象相互持有对方的shared_ptr的情况，这时需要小心的处理释放逻辑，避免环状引用带来的内存泄漏。这里受到教训，在这样写之前就知道这样使用会形成环状引用，释放的时候会比较麻烦，但是当时实现的时候考虑到情况不多，控制好释放逻辑，顺序释放就ok，weak_ptr每次需要转换成shared_ptr使用觉得太麻烦就没有使用。随后程序加入的外部销毁释放的逻辑，打乱了原来的释放逻辑，情况变的复杂，决定还是使用weak_ptr来维护最简单。

>std::weak_ptr 是一种智能指针，它对被 std::shared_ptr 管理的对象存在非拥有性（“弱”）引用。在访问所引用的对象前必须先转换为 std::shared_ptr。
>
>std::weak_ptr 用来表达临时所有权的概念：当某个对象只有存在时才需要被访问，而且随时可能被他人删除时，可以使用 std::weak_ptr 来跟踪该对象。需要获得临时所有权时，则将其转换为 std::shared_ptr，此时如果原来的 std::shared_ptr 被销毁，则该对象的生命期将被延长至这个临时的 std::shared_ptr 同样被销毁为止。
>
>std::weak_ptr 的另一用法是打断 std::shared_ptr 所管理的对象组成的环状引用。若这种环被孤立（例如无指向环中的外部共享指针），则 shared_ptr 引用计数无法抵达零，而内存被泄露。能令环中的指针之一为弱指针以避免此情况。

我的需求里需要打断环状引用。

## Share

分享一个VS Code的markdown插件[markdown-preview-enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/zh-cn/vscode-installation)
教程很详细，各种图 数学公式等等支持十分完备，日常用来写文档很方便