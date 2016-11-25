# 头文件,API和数据类型设计
## 数据类型
```
typedef enum {
  LEPT_NULL,
  LEPT_FALSE,
  LEPT_TRUE,
  LEPT_NUMBER,
  LEPT_STRING,
  LEPT_ARRAY,
  LEPT_OBJECT
} lept_type;
```
1. 用枚举表示数据类型
2. 使用typedef给枚举别名`lept_type`\
## 分割结果
```
enum {
  LEPT_PARSE_OK = 0,
  LEPT_PARSE_EXPECT_VALUE,
  LEPT_PARSE_INVALID_VALUE,
  LEPT_PARSE_ROOT_NOT_SINGULAR
};
```
1. 使用枚举表示分割json数据结果的标识
## json数据结构
```
typedef struct { lept_type type; } lept_value;
```
1. 使用`lept_type`表示json数据的类型
2. 本🌰只包含数据类型，数据的值字段带添加
### json字符串上下文
```
typedef struct { const char *json; } lept_context;
```
1. 使用一个结构体封装*json字符串*
## 分割函数
```
int lept_parse(lept_value *v, const char *json);
```
1. 外部调用该库的入口
	* 接受两个参数，*json数据结构*和*json字符串*
	* 实现会调用其它函数，但不对外体现
2. 为防止对*json*字符串进行更改，使用`const`关键字
## 获取类型
```
lept_type lept_get_type(const lept_value *v);
```
1. 接收一个参数*json数据结构*，返回数据类型