# 解析字符串 笔记
[TOC]
## C语言语法、函数与数据结构
### memcpy
```c
void* memcpy(void *restrict dst,
        const void *restrict src,size_t n);
```

> DESCRIPTION
     The memcpy() function copies n bytes from memory area src to memory area dst.  If dst and src overlap, behavior is undefined.  Applications in which dst
     and src might overlap should use memmove(3) instead.

1. 使用`memcpy`要注意`dst`和`src`不能有重叠区域，否则结果是未定义的。

### 动态数组
详见缓冲区与堆栈
### 内存分配函数
## 函数与设计
**注意事项**：

1.  在解析过程中，`lept_parse_**`,`lept_context_**`函数和宏`EXPECT`，会对`lept_context`对象的内容作出更改。
2. `lept_parse_**`函数，在解析成功后，必须修改`lept_context`，不能遗漏 
3. 解析过程中，为防止发生异常导致上下文内容无法恢复，需要复制一份进行读取，如果解析成功，再修改`lept_context`上下文
4. 不可更改的字符串要用`const char*`

### JSON中的转义编码
```
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
```
1. 在json中，共支持9种转义序列
2. JSON不以`\0`作为字符串的结束符，字符串中出现`\0`时，后面的字符仍然会解析

### 字符串设置函数与内存管理

```c
void lept_set_string(lept_value *v, const char *s, size_t len) {
  assert(v != NULL && (s != NULL || len == 0));
  lept_free(v);
  v->u.s.s = (char *)malloc(len + 1);
  memcpy(v->u.s.s, s, len);
  v->u.s.s[len] = '\0';
  v->u.s.len = len;
  v->type = LEPT_STRING;
}
```

1. 函数定义为`void`类型，接收3个参数
    * 一个json值指针
    * 一个常量字符串
    * 一个size_t类型的长度(字节数)
2. 断言条件
    * json值指针不能为NULL
    * 字符串不为空
    * 字符串为空但len为0
3. 设置前要先清空json值
4. 分配内存
    * 长度要在len(字符串实际长度)的基础上+1，用来保存`'\0'`
    * `malloc`返回`void*`型指针，转换成`char*`
5. 按字节拷贝
6. 给json值加`'\0'`
7. 设置json值中字符串的实际长度
8. 设置数据类型为`LEPT_STRING`

### json值释放函数
```c
void lept_free(lept_value *v) {
  assert(v != NULL);
  if (v->type == LEPT_STRING)
    free(v->u.s.s);
  v->type = LEPT_NULL;
}
```

1. 返回值类型为`void`，接收一个json值指针作为参数
2. 断言条件为：json值指针不能为NULL
3. 如果该值为`LEPT_STRING`类型，需要先释放内存
4. 设置类型为`LEPT_NULL`，可以避免对一块内存的重复释放

### json值初始化函数

```c
#define lept_init(v)                                                           \
  do {                                                                         \
    (v)->type = LEPT_NULL;                                                     \
  } while (0)
```

1. 初始化即将json值的初始类型设置为`LEPT_NULL`
2. 使用`do{...}while(0)`的目的是为了保证语句能够正确执行，并不会与条件语句等相互影响

### json值置空函数
```c
#define lept_set_null(v) lept_free(v)
```
1. 本质上就是释放函数，所以只用一个宏来定义

### 缓冲区与堆栈
我们解析字符串时，需要：
1. 先把解析的结果存在一个临时的缓冲区
2. 用`lept_set_string()`把缓冲区的结果设置到`lept_value`中
在完成解析一个字符串之前，缓冲区的大小是不能预知的，因此我们需要一个能自动扩展空间的数据结构，即**动态数组(dynamic array)**

如果每次解析字符串时，都重新建一个动态数组，那么是比较耗时的。我们可以重用这个动态数组，每次解析 JSON 时就只需要创建一个。而且我们将会发现，无论是解析字符串、数组或对象，我们也只需要以先进后出的方式访问这个动态数组。换句话说，我们需要一个动态的堆栈数据结构。

```c
typedef struct {
    const char* json;
    char* stack;
    size_t size, top;
}lept_context;
```
#### 修改`lept_parse()`函数
```c
int lept_parse(lept_value *v, const char *json) {
  lept_context c;
  int ret;
  assert(v != NULL);
  c.json = json;
  c.stack = NULL;
  c.size = c.top = 0;
  lept_init(v);
  lept_parse_whitespace(&c);
  if ((ret = lept_parse_value(&c, v)) == LEPT_PARSE_OK) {
    lept_parse_whitespace(&c);
    if (*c.json != '\0') {
      v->type = LEPT_NULL;
      ret = LEPT_PARSE_ROOT_NOT_SINGULAR;
    }
  }
  assert(c.top == 0);
  free(c.stack);
  return ret;
}
```
1. 返回值为`int`，接收参数`lept_value*`和`const char*`
2. 创建一个`lept_context`的json上下文,初始化返回值
3. 断言条件`lept_value*`不为空
4. 初始化栈和栈顶指针
5. 初始化`lept_value`
6. ...
7. 断言检查，栈顶为0
8. 释放栈内存

#### 栈的压入与弹出

```c
#ifndef LEPT_PARSE_STACK_INIT_SIZE
#define LEPT_PARSE_STACK_INIT_SIZE 256
#endif

static void* lept_context_push(lept_context* c, size_t size) {
    void* ret;
    assert(size > 0);
    if (c->top + size >= c->size) {
        if (c->size == 0)
            c->size = LEPT_PARSE_STACK_INIT_SIZE;
        while (c->top + size >= c->size)
            c->size += c->size >> 1;  /* c->size * 1.5 */
        c->stack = (char*)realloc(c->stack, c->size);
    }
    ret = c->stack + c->top;
    c->top += size;
    return ret;
}

static void* lept_context_pop(lept_context* c, size_t size) {
    assert(c->top >= size);
    return c->stack + (c->top -= size);
}
```
##### 压入前
1. `lept_context_push`接收json上下文指针和字符个数，允许压入任意数量的字符。返回**待压入数据应该存放的起始位置**，`void*`型。
2. 创建返回值指针
3. 断言条件`size>0`
4. 检查栈的剩余空间与内存分配
    * 如果当前为空，则将栈大小设置为初始大小
    * 如果当前不为空，并会溢出，则循环检查溢出状态，按照1.5倍比例重新分配内存
5. 修改栈顶位置，修改为增加若干字符后的位置
6. 返回**保存待压入数据的位置**

#### 单字符压入
我们需要采用对字符的修改进行压入，使用一个宏进行逐字符修改

```c
#define PUTC(c, ch)                                                            \
  do {                                                                         \
    *(char *)lept_context_push(c, sizeof(char)) = (ch);                        \
  } while (0)
```
##### 弹出
1. `lept_context_pop`，返回**待弹出的字符串的起始位置**
2. 待弹出的字符数，必须少于栈内的字符数
3. 修改上下文结构内的栈顶位置

### 解析字符串函数
未做转义处理和不合法字符校验。

1. 接收参数`lept_context*`和`lept_value*`，返回解析结果标记
2. 在此过程中`lept_context.top`会被修改，如果字符串正确（"*"）
    * 备份`lept_context.top`为`head`：
        * 栈顶在解析过程中发生变化，一旦中途错误，恢复栈顶
        * 解析完成后，使用栈顶指针和起始位置`head`计算长度`len`
    * 处理字符，期间`lept_context.top`有改动
    * 语句`len=c->top-head`计算字符串长度
3. 返回parse结果标记

```c
static int lept_parse_string(lept_context *c, lept_value *v) {
  size_t head = c->top, len;
  const char *p;
  EXPECT(c, '\"');
  p = c->json;
  for (;;) {
    char ch = *p++;
    switch (ch) {
    case '\"':
      len = c->top - head;
      lept_set_string(v, (const char *)lept_context_pop(c, len), len);
      c->json = p;
      return LEPT_PARSE_OK;
    case '\0':
      c->top = head;
      return LEPT_PARSE_MISS_QUOTATION_MARK;
    default:
      PUTC(c, ch);
    }
  }
}
```

## 单元测试修改
应用方的代码在调用`lept_parse()`之后，最终也应该调用`lept_free()`来释放内存。
如果不调用`lept_parse()`，我们需要初始化值，就要像以下单元测试，先`lept_init()`再`lept_free()`。

```c
static void test_access_string() {
  lept_value v;
  lept_init(&v);
  lept_set_string(&v, "", 0);
  EXPECT_EQ_STRING("", lept_get_string(&v), lept_get_string_length(&v));
  lept_set_string(&v, "Hello", 5);
  EXPECT_EQ_STRING("Hello", lept_get_string(&v), lept_get_string_length(&v));
  lept_free(&v);
}
```
1. 静态的返回值为`void`型的函数，无参数
2. 创建一个`lept_value`
3. 对其初始化
4. 设置为`LEPT_STRING`类型
5. 调用`EXPECT_EQ_STRING()`进行校验
6. 重新设置为新的字符串
7. 重复第5步
8. 释放该`lept_value`

