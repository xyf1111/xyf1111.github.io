# Go Study 05

<!--more-->
## 寻找最长不含有重复字符的子串
- abcabcbb -> abc
- bbbbb -> b
- pwwkew ->wke
```go
func lenthofNonRepeatingSubStr(s string) int {
	lastOccurred := make(map[rune]int)
	start := 0
	maxLength := 0
	for i, ch := range []rune (s){
		if lastI, ok := lastOccurred[ch]; ok && lastI >= start {
			fmt.Println(lastI)
			fmt.Println(ok)
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i-start+1
		}
		lastOccurred[ch] = i
	}
	return maxLength
}
```
