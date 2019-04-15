# ARTS第四周

## Algorithm

[ZigZag Conversion](https://leetcode-cn.com/problems/zigzag-conversion/)

>The string "PAYPALISHIRING" is written in a zigzag pattern on a given number of rows like this: (you may want to display this pattern in a fixed font for better legibility)
>
>P   A   H   N
>A P L S I I G
>Y   I   R
>And then read line by line: "PAHNAPLSIIGYIR"
>
>Write the code that will take a string and make this conversion given a number of rows:
>
>string convert(string s, int numRows);
>Example 1:
>
>Input: s = "PAYPALISHIRING", numRows = 3
>Output: "PAHNAPLSIIGYIR"
>Example 2:
>
>Input: s = "PAYPALISHIRING", numRows = 4
>Output: "PINALSIGYAHRPI"
>Explanation:
>
>P     I    N
>A   L S  I G
>Y A   H R
>P     I

思路：对字符串进行z字排序，保存每一行的顺序，最后合并行
时间复杂度$O{n}$  
空间复杂度$O(n)$

```go

import "bytes"
func convert(s string, numRows int) string {
	arr := make(map[int][]byte)
	i, j := 0, 0
	positive := true
	for i = 0; i < len(s); i++ {
		arr[j] = append(arr[j], s[i])

		if positive {
			if j < numRows-1 {
				j++
			} else {
				positive = false
				if j > 0 {
					j--
				}
			}
		} else {
			if j > 0 {
				j--
			} else {
				positive = true
				if j < numRows-1 {
					j++
				}
			}
		}
	}

	var buffer bytes.Buffer
	for k := 0; k < numRows; k++ {
		buffer.Write(arr[k])
	}

	return buffer.String()
}

```

## Review

The Go Blog [Go Slice : useage and internals](https://blog.golang.org/go-slices-usage-and-internals)  
这篇blog讲了go slice 的实现原理和使用，slice的结构是一个 |Elem*|Len |Cap| 这样的结构，elem*指向底层数组的某个元素，len是当前长度 cap是最大容量  
增加slice的容量会申请一个更大的底层数组，然后拷贝数据  
内建的copy函数可以用来拷贝splice的底层数据  
有时候需要自定义实现诸如AppenBytes的函数，这样方便我们控制slice追加的时候控制扩大的容量  
只需要使用大数组中的一小段slice的时候，我们需要copy这个slice，如果直接使用大数组的slice会浪费大量内存

## Tips

C++ std::condition_variable使用注意事项：condition_variable的mutex需要是保护临界资源的mutex，使用另一个的mutex作为线程唤醒的mutex就可能两步获取锁不一致的问题。

```c++
#include <iostream>
#include <string>
#include <thread>
#include <mutex>
#include <condition_variable>

std::mutex mCV;
std::condition_variable cv;

std::mutex m;
int flag = 0;
 
void worker1_thread()
{
    {
        std::lock_guard<std::mutex> lk(m);
        //do something
        flag = 1;
    }

    cv.notify_one();

    {
        std::lock_guard<std::mutex> lk(m);
        //do something
        flag = 0;
    }
}
 
int main()
{
    std::thread worker(worker1_thread);

    std::lock_guard<std::mutex> lk(mCV);
    cv.wait(mCV,[&](){
        std::lock_guard<std::mutex> lk(m);
        return flag != 0;
        });
 
    worker.join();
}
```

## Share

使用 C++ atomic_flag实现TAS模式的自旋锁

```C++

#include <atomic>
class SRTSpinMutex{
public:
    SRTSpinMutex() = default;
    SRTSpinMutex(const SRTSpinMutex&) = delete;
    SRTSpinMutex& operator= (const SRTSpinMutex&) = delete;
    void lock() {
      while(flag.test_and_set(std::memory_order_acquire));
    }
    bool try_lock(){return flag.test_and_set(std::memory_order_acquire);}
    void unlock() {
      flag.clear(std::memory_order_release);
    }
private:
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
};

```