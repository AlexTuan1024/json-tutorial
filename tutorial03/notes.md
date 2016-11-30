# 解析字符串 笔记
## C语言语法与函数
### `void* memcpy(void *restrict dst,const void *restrict src,size_t n)`

> DESCRIPTION
     The memcpy() function copies n bytes from memory area src to memory area dst.  If dst and src overlap, behavior is undefined.  Applications in which dst
     and src might overlap should use memmove(3) instead.

1. 使用`memcpy`要注意`dst`和`src`不能有重叠区域，否则结果是未定义的。
## 解析函数与设计
### JSON中的转义编码
~~~
string = quotation-mark *char quotation-mark
char = unescaped /
   escape (
       %x22 /          ; "    quotation mark  U+0022
       %x5C /          ; \    reverse solidus U+005C
       %x2F /          ; /    solidus         U+002F
       %x62 /          ; b    backspace       U+0008
       %x66 /          ; f    form feed       U+000C
       %x6E /          ; n    line feed       U+000A
       %x72 /          ; r    carriage return U+000D
       %x74 /          ; t    tab             U+0009
       %x75 4HEXDIG )  ; uXXXX                U+XXXX
escape = %x5C          ; \
quotation-mark = %x22  ; "
unescaped = %x20-21 / %x23-5B / %x5D-10FFFF
~~~
1. 在json中，共支持9种转义序列
2. JSON不以`\0`作为字符串的结束符，字符串中出现`\0`时，后面的字符仍然会解析
