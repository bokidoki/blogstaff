---
title: unicode学习以及其它
categories: Android
top: 0
date: 2020-07-13 17:56:55
tags:
thumbnail:
---

unicode 定义了字符与字节码的映射关系  
utf-8 utf-16 utf-32 定义了字节码在内存中的存储方式  
utf-8 (1-4个字节表示一个符号)  
utf-16(2-4个字节表示一个符号)  
utf-32(4个字节表示一个符号)  

unicode -> utf-编码  

unicode符号范围（十六进制）|UTF-8编码方式(二进制)
---|:--:
0000 0000 - 0000 007F|0xxxxxxx
0000 0080 - 0000 07FF|110xxxxx 10xxxxxx
0000 0800 - 0000 FFFF |1110xxxx 10xxxxxx 10xxxxxx
0001 0000 - 0010 FFFF|11110xxx 10xxxxxx 10xxxxxx 10xxxxxx

举例 汉字严的unicode字符是4E25，对应的范围是0000 0800 - 0000 FFFF 二进制是
01001110 00100101
UTF-8模板为
1110xxxx 10xxxxxx 10xxxxxx
将严的二进制码从左向右填入模板可得
11100100 10111000 10100101
所以严的UTF-8编码的16进制结果为 E4B8A5

码点：所谓码点就是字符在unicode中映射成的一个数字。

java中char
char的长度为2个字节，存储的是unicode的码点（两个字节）

李  
674E  
01100111 01001110  
1110xxxx 10xxxxxx 10xxxxxx  
11100110 10011101 10001110  
E6 9D8E  

Byte.toUnsignedInt() byte只有2个字节8位
-26  
 26 0001 1010  
-26 1110 0110  
230 1110 0110  
负数为正数的补码 + 1

打印出unicode的16进制码点

```java
String s = "xx";x
byte[] bytes = s.getBytes(Charset.defaultCharset());
for (byte b : bytes) {
    String hex = Integer.toHexString(Byte.toUnsignedInt(b));
}
```

英文|简称|中文
---|:--:|--:
binary|BIN|二进制
octal|OCT|八进制
decimal|DEC|十进制
hex|HEX|十六进制

## 参考

[字符编码笔记：ASCII，Unicode 和 UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
