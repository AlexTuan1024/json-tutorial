# 阅读笔记
## 语法
1. `fprintf("%.17g")`的含义：参数以`%e`或`%f`的格式打印，取决于它的值
	* 如果指数大于等于`-4`但小于精度字段，就使用`%f`格式
	* 否则使用`%e`格式打印

## 数据结构修改
### `lept_value`的修改
```  
typedef struct {
  double n;
  lept_type type;
} lept_value;
```

## 函数
### 函数说明
1. `lept_parse`是分割的入口，
	* 负责封装json字符串
	* 调用`lept_parse_whitespace`和`lept_parse_value`
	* 实际分割后的检查，如果分割动作完成后，指向的字符并非'\0'，则表示非法值导致分割不完全
2. `lept_parse_value`会根据第一个字母将任务分发给不同的分割函数
3. `EXPECT`和每次分割，不管是whitespace还是value，都会移动首字符指针的位置

### 新增函数`lept_get_number`
```
double lept_get_number(const lept_value *v){
	assert(v != NULL && v->type == LEPT_NUMBER);
	return v->n;
}
```
### 新增函数`lept_parse_number`
```
static int lept_parse_number(lept_context *c,lept_value *v){
	char *end;
	v->n = strtod(c->json,&end);
	if(c->json == end)
		return LEPT_PARSE_INVALID_VALUE;
	c->json = end;
	v->type = LEPT_NUMBER;
	return LEPT_PARSE_OK;
}
```
1. 调用库函数`double strtod(char const *restrict nptr,char **restrict endptr)`
	* 扫描ntpr字符串，忽略空格
	* 遇到非数字字符停止
	* 十进制或十六进制
	* 如果endptr不为NULL，则会将遇到不合条件而中止的nptr中的字符指针由endptr传回。
	* 之所以传递一个二级指针，是因为如果只传递一级指针，函数内部只能得到该指针指向的地址，
	只能对该内存地址进行修改，而传递一个二级指针，则可以修改*endptr（一级指针）的值，
	将其赋值为中止处的地址。
	* 除了该函数外，同系列的还有`strtof,strtodl`，分别返回`float`和`long double`类型
	* 测试代码
	
	```
	#include <stdio.h>
	#include <stdlib.h>

	int main(){
		char testArray[] = "1234.5678dfs\0";
		char *end;
		double result;
		result = strtod(testArray,&end);
		printf("result:%.10g\n",result);
		printf("end:%s\n",end);
		printf("start:\t%p\n",testArray);
		printf("end:\t%p\n",end);
		printf("difference:%ld\n",(end-testArray)/sizeof(char));
		return 0;
	}
	```
	* 输出结果
	
	```
	result:1234.5678
	end:dfs
	end:	0x7fff58950903
	start:	0x7fff589508fa
	difference:9
	```
	
2. `if(c->json == end)`判断错误
	如果`c->json`和`end`指向同一处地址，则证明第一个非空字符（在本例中为第一个字符）就是不合法字符。