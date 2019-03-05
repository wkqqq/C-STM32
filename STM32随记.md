# STM32知识点随记
## 自定义结构体与全局变量用法--2019.3.5
>前提：已有自定义文件库wk_usart.c和wk_usart.h
>
* 举例：定时器输入捕获用户自定义变量结构体声明

### 在wk_usart.h文件中：
首先进行结构体的声明：

```C
typedef struct
{   
	uint8_t   Capture_FinishFlag;   // 捕获结束标志位
	uint8_t   Capture_StartFlag;    // 捕获开始标志位
	uint16_t  Capture_CcrValue;     // 捕获寄存器的值
	uint16_t  Capture_Period;       // 自动重装载寄存器更新标志 
}TIM_ICUserValueTypeDef;
```

`如果在wk_usart.h文件的最后加上：

```C
extern TIM_ICUserValueTypeDef TIM_ICUserValueStructure;
```
则结构体TIM_ICUserValueStructure变成了全局变量，任何包含了wk_usart.h的文件中都可以直接使用

### 在wk_usart.h文件中：

```C
// 定时器输入捕获用户自定义变量结构体定义
TIM_ICUserValueTypeDef TIM_ICUserValueStructure = {0,0,0,0};
```
进行参数初始化