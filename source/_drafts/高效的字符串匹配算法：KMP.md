## 问题

给定一个字符串text，以及一个模式串pattern，请你给出pattern在text中第一个匹配项的下标，如果pattern不属于text的一部分，则返回-1。

例1：

输入： text = "ababcababd ", pattern = "ababd"

输出： 5

例2:

输入 text = "hello world", pattern = "no"

输出：-1

## 朴素解法

### 代码

```go
func strStr(haystack string, needle string) int {
    if len(haystack) < len(needle) {
        return -1
    }
    originLen, patternLen := len(haystack), len(needle)
    for i := 0; i <= originLen - patternLen; i++ {
        if haystack[i] != needle[0] {
            continue
        }
        p1, p2 := i, 0
        for p1 < len(haystack) && p2 < len(needle) && haystack[p1] == needle[p2] {
            p1++
            p2++
        }
        if p2 >= len(needle) {
            return i
        }
    }
    return -1
}
```



## KMP



### next数组计算方式

令字符串为s，对于下标i，next[i]的值就是str[0,i)的最长相等**真前后缀**长度，。

当然我们可以在字符串匹配的过程中，每遍历到一个字符就计算它前面子串的最长相等前后缀，但是这样的效率相比暴力并没有提升。

因此我们需要进一步观察next数组的特点，看看有没有更高效的办法。next数组是从前往后计算的，那么可以对每个next[i]暴力计算，然后字符串匹配过程用next数组即可，但是暴力计算一遍next数组的效率也比较低下，有什么更好的办法呢？

我们说到，next数组是从前往后计算的，那么如果能用前面的值推导出后面的值，那就很好了！

![img](https://oss.seineo.cn/images/202406151707714.png)

以next[8]为例，next[8]的含义是str[0]到str[7]的最长相等前后缀长度，那么next[7]=2，也就意味str[0:1] == str[5:6]，那么如果str[7] == str[2] 我们就可以得到next[8] = next[7] + 1。

因此，可以得到一个结论：

>结论1:
>
>如果 array[ next[m] ] == array[ m ]，那么next[ m+1 ] = next[ m ] + 1；



那如果str[next[m]] != str[m]呢？比如下图中next[10]的计算，str[9] != str[next[9]] = str[4]：

![img](https://oss.seineo.cn/images/202406151722346.png)



那么说明没有5那么长的相同前后缀，可以向下尝试（其实这里就可以直接说str[0:3]等于str[5:8]，要想计算最长相同前后缀，也就是要找到str[0:3]的最长前后缀如str[0:i]，如果str[i+1] == str[9]那么就找到了），4的话也就是str[0:3]与str[6:9]，但是我们并不知道他们是否相等，我们只知道str[0:3] == str[5:8]，可以继续尝试利用相同前后缀来推导。要判断str[0:3]与str[6:9]是否相等，也就是判断str[3]是否等于str[9] 且 str[0:2] 是否等于str[6:8]。

**由next[9]=4可知,str[0:3] == str[5:8]， 这两个字符串完全相等，那么如果str[0:2] == str[6:8]那也就说明str[0:2]就是str[0:3]相等前后缀。**

那我们要检查str[0:3]的最长相等前后缀，也就是next[4]，我们发现next[4] == 2，因此str[0:2] != str[6:8]。因为str[0:3] == str[6:8]， 因此str[0:3]的最长相等前后缀就可以用来构造next[10]的值。（可以试试反证法？）

那么next[4] == 2，也就是str[0:1] == str[2:3]，那么也就是str[0:1] == str[7:8]，我们就只需要进一步判断str[2] 是否等于 str[9]即可，我们发现等于， 因此next[10] = next[4] + 1。

那么以此类推，我们可以得到一个新的结论：

> 结论2:
>
> 令m为当前下标，t = next[m]
>
> While array[t] != array[m]:
>
> ​	t = next[t]
>
> next[m+1] = t + 1

根据上述结论，如果array[t]一直不等于array[m]会怎么样？ t会为0，而next[0] = 0，这样就无限循环了，因此我们可以将next[0] = -1，来巧妙地解决这个问题。

### 代码

```go
func getNext(needle string) []int {
	next := make([]int, len(needle))
    if len(needle) == 0 {
        return []int{}
    }
    if len(needle) == 1 {
        return []int{-1}
    }
	next[0] = -1
	next[1] = 0
	for i := 1; i < len(needle)-1; i++ {
		t := next[i]
		for t >= 0 && needle[i] != needle[t] {
			t = next[t]
		}
		next[i+1] = t + 1
	}
	return next
}

func strStr(haystack string, needle string) int {
    if len(haystack) < len(needle) {
        return -1
    }
    originLen, patternLen := len(haystack), len(needle)
    next := getNext(needle)
    j := 0
    for i := 0; i < originLen; i++ {
        for j > 0 && haystack[i] != needle[j] {  // j > 0 是处理第一个字符，此时如果不等，next[0] = -1 无法进一步比较
            j = next[j]
        }
        if haystack[i] == needle[j] {
            j++
        }
        if j == patternLen {
            return i-patternLen+1
        }
    }
    return -1
}
```



## 参考

1. https://www.cnblogs.com/aninock/p/13796006.html