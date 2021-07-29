## Learning Golang strings.TrimLeft
```Go
func TrimLeft(s, cutset string) string {
	if s == "" || cutset == "" {
		return s
	}
	return TrimLeftFunc(s, makeCutsetFunc(cutset))
}
```
```Go
func TrimLeftFunc(s string, f func(rune) bool) string {
	i := indexFunc(s, f, false)
	if i == -1 {
		return ""
	}
	return s[i:]
}
```
```Go
func indexFunc(s string, f func(rune) bool, truth bool) int {
	for i, r := range s {
		if f(r) == truth {
			return i
		}
	}
	return -1
}
```
```Go
func makeCutsetFunc(cutset string) func(rune) bool {
    // ut8.RuneSelf = 0x80 = 128
	if len(cutset) == 1 && cutset[0] < utf8.RuneSelf {
		return func(r rune) bool {
			return r == rune(cutset[0])
		}
	}
	if as, isASCII := makeASCIISet(cutset); isASCII {
		return func(r rune) bool {
			return r < utf8.RuneSelf && as.contains(byte(r))
		}
	}
	return func(r rune) bool { return IndexRune(cutset, r) >= 0 }
}
```
```Go
type asciiSet [8]uint32
func makeASCIISet(chars string) (as asciiSet, ok bool) {
	for i := 0; i < len(chars); i++ {
		c := chars[i]
		if c >= utf8.RuneSelf {
			return as, false
		}
		as[c>>5] |= 1 << uint(c&31)
	}
	return as, true
}
```
```Go
func (as *asciiSet) contains(c byte) bool {
	return (as[c>>5] & (1 << uint(c&31))) != 0
}
```

1. 如果 cutset 是一个字符，并且该字符属于 ascii 字符，则遍历 string s，一旦遍历的字符(i, r)不等于 cutset，就返回 s[i:]
2. 如果 cutset 的长度大于 1且字符均为 ascii 字符，则生成 asciiSet 用于判断 contains。遍历 string s，一旦遍历的字符(i, r)不存在于 asciiSet 中，就返回 s[i:]
3. ...上述逻辑很好理解，关键在于 asciiSet 的生成

asciiSet 的定义为 长度为8的 uint32 数据

as[c>>5] |= 1 << uint(c&31)   =>   as[c>>5] = as[c>>5] | 1 << uint(c&31)

as 的初始状态为 {0，0，0，0，0，0，0，0}

我们假设 c = 'b', rune('b') = 98 = 0110 0010

c >> 5 = 0110 0010 >> 5 = 0000 0011 即只保留高三位，而在 ascii table 中，高三位的范围是 000 ~ 111 = 0 ~ 7， 对应了 asciiSet 的长度为 8

c & 31 = 0110 0010 & 0001 1111 = 0000 0010, 即只保留 c 的低五位， 低五位的范围是 00000 ~ 11111 = 0 ~ 31

1 << uint(c&31) = 1 << uint(0000 0010) = 1 << 2 = 0100, 因为 c & 31 的范围为 0 ~ 31， 所以此处的 0100 表示为 0000 0000 0000 0000 0000 0000 0000 0100

as[c>>5] = as[0000 0011] = as[3] = 0 | 0000 0000 0000 0000 0000 0000 0000 0100 = 0000 0000 0000 0000 0000 0000 0000 0100

依此推理，假设 c = 'a', rune('a') = 97 = 01100001, 则 1 << uint(c&31) = 0000 0000 0000 0000 0000 0000 0000 0010

假设 c 的高三位固定为 011， 1 << uint(c&31) 可表示 ascii table 中 0110 0000 ~ 0111 1111 的所有字符，共32个，分别表示为 0000 0000 0000 0000 0000 0000 0000 0000 ~ 1000 0000 0000 0000 0000 0000 0000 0000

假设 c 的值先为 'b'，后为 'a'，则可得出 as[3] = 0000 0000 0000 0000 0000 0000 0000 0110, 此时如果要判断 'a' 是否存在于 asciiSet 中，则只需判断 1 << uint('a'&31) & as['a'>>5] != 0 为 true 即可

1 << uint('a'&31) = 0000 0000 0000 0000 0000 0000 0000 0010

as['a'>>5] = as[3] = 0000 0000 0000 0000 0000 0000 0000 0110

0000 0000 0000 0000 0000 0000 0000 0010 & 0000 0000 0000 0000 0000 0000 0000 0110 = 0000 0000 0000 0000 0000 0000 0000 0010 != 0 为 true，所以 'a' 存在于 asciiSet 中
