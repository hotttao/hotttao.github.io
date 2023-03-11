# go 字符集编码/解码


## 1. 字符集原理
Go语言默认源码文件中的字符是采用UTF-8编码方案的Unicode字符。在Go中，每个rune对应一个Unicode字符的码点，而Unicode字符在内存中的编码表示则放在[]byte类型中。从rune类型转换为[]byte类型，称为“编码”（encode），而反过来则称为“解码”（decode）。

