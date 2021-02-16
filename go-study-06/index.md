# Go Study 06

<!--more-->
## rune相当于go的char
### 使用range遍历pos，rune对
```go
    s := "Yes我是你爸爸!"

	for _, b := range []byte(s) {
		fmt.Printf("%X ", b)
	}
	fmt.Println()

	for i, ch := range s {
		fmt.Printf("(%d %X) ", i, ch)
	}
```
### 使用utf8.RuneCountInString获得字符数量
```go
    fmt.Println("Rune count:", utf8.RuneCountInString(s))
```
### 使用len获得字节长度
```go
    fmt.Println(len(s))
```
### 使用[]byte获得字节
```go
    bytes := []byte(s)
	for len(bytes) > 0 {
		ch, size := utf8.DecodeRune(bytes)
		bytes = bytes[size:]
		fmt.Printf("%c ", ch)
	}
```
## 其他字符串操作
### Fields，Split，Join
### Contains，Index
### ToLower，ToUpper
### Trim，TrimRight，TrimLeft
