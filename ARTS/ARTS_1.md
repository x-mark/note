# ARTS第一周

## Algorithm
[leetcode无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/solution/)

>Given a string, find the length of the longest substring without repeating characters.
>
>Example 1:
>
>Input: "abcabcbb"
>Output: 3 
>Explanation: The answer is "abc", with the length of 3. 
>Example 2:
>
>Input: "bbbbb"
>Output: 1
>Explanation: The answer is "b", with the length of 1.
>Example 3:
>
>Input: "pwwkew"
>Output: 3
>Explanation: The answer is "wke", with the length of 3. 
>             Note that the answer must be a substring, "pwke" is a subsequence and not a substring.

最近正准备学习一下golang所以就用go来练习解答。[详细记录](https://xmark.xyz/?p=46)
```go
func lengthOfLongestSubstring(s string) int {
   	var window []rune
	max := 0
	for _, r := range s {
		for i, vr := range window {
			if vr == r {
				window = window[i+1:]
				break
			}
		}

		window = append(window, r)
		if max < len(window) {
			max = len(window)
		}
	}

	return max
}
//leetcode提交结果：执行用时12ms 内存消耗3.2M
```
## Review
Review了medium的这篇[WordPress over HTTPS with Docker (SSL)](https://medium.com/today-i-solved/wordpress-over-https-with-docker-ssl-ecaf02a47fea?tdsourcetag=s_pctim_aiomsg)。
这篇blog主要记录了怎么在docker下通过wordpress nginx letsencrypt来搭建https的站点，刚好我自己的wordpress站点需要升级，所以就参考这篇文章的步骤重新升级了我的wordpress站点。[xmark.xyz](https://xmark.xyz/)

## Tip

今天的Tip记录一个遇到的C++语法解析问题
考虑如下代码

```C++
class background_task
{
public:
  void operator()() const
  {
    do_something();
    do_something_else();
  }
};

background_task f;
std::thread my_thread(f);
```

而如果最后两句写成如下形式

```C++
std::thread my_thread(background_task());
```

这里相当与声明了一个名为my_thread的函数，这个函数带有一个参数(函数指针指向没有参数并返回background_task对象的函数)，返回一个std::thread对象的函数，而非启动了一个线程。

使用在前面命名函数对象的方式，或使用多组括号，如1，或使用新统一的初始化语法，如2，可以避免这个问题。
如下所示：

```C++
std::thread my_thread((background_task()));  // 1
std::thread my_thread{background_task()};    // 2
```

## Share

分享一篇最近看的陈硕的[Muduo 设计与实现之一：Buffer 类的设计](https://blog.csdn.net/Solstice/article/details/6329080)，最近正在做一个基于UDP的文件传输工具，处理Buffer的时候看了这篇文章。在这里的设计是使用```std::Vector<char>```存储，int类型的index来跟踪当前容量。而没有采用fixed size array的方案。好处是既可以动态增长又能保证连续存储。使用很方便同时也不会因为大量连接而浪费太多内存。代价是vector扩容带来的数据拷贝。