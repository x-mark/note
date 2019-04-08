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

```go

func longestPalindrome(s string) string {
	if len(s) < 2 {
		return s
	}
	var substr string
	for end := 1; end < len(s); end++ {
		begin := end - 1
		for s[end] == s[begin] {
			if len(substr) < end-begin+1 {
				substr = s[begin : end+1]
			}
			if begin > 0 && end < len(s)-1 {
				begin--
				end++
			} else {
				break
			}
		}
	}
	for end := 2; end < len(s); end++ {
		begin := end - 2
		for s[end] == s[begin] {
			if len(substr) < end-begin+1 {
				substr = s[begin : end+1]
			}
			if begin > 0 && end < len(s)-1 {
				begin--
				end++
			} else {
				break
			}
		}
	}

	return substr
}
```

## Review
[Machine Lerning for Humans Part 2.1 Supervised Learning](https://medium.com/machine-learning-for-humans/supervised-learning-740383a2feab)
回归：预测一个连续值
特征(feature)应满足数字的(numerical)或明确的(categorical)
训练集和测试集
supervised learning algorithms. 监督学习算法


## Tip

## Share