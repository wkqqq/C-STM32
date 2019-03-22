# STM32知识点随记
## 1、自定义结构体与全局变量用法--2019.3.5
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

如果在wk_usart.h文件的最后加上：

```C
extern TIM_ICUserValueTypeDef TIM_ICUserValueStructure;//TIM_ICUserValueStructure结构体在wk_usart.C文件中定义
```
则结构体TIM_ICUserValueStructure变成了全局变量，任何包含了wk_usart.h的文件中都可以直接使用

### 在wk_usart.C文件中：

```C
// 定义一个TIM_ICUserValueTypeDef类型的TIM_ICUserValueStructure结构体并初始化
TIM_ICUserValueTypeDef TIM_ICUserValueStructure = {0,0,0,0};
```

### 总结：
结构体类型定义和全局变量声明在.h文件中，变量的定义在.C文件中。

## 2、通信相关知识点--2019.3.11
`波特率：每秒传输的数据位数`
* 举例：波特率为115200，1S/115200 = 8.68us,则一个位的时间为8.68us，字长为8,0个校验位，1个停止位，则一帧数据位1+8+0+1=10位，发送时间需要86.8us

`上位机串口调试助手：总是默认显示ASCII码字符，对于单片机发送，总是发送对应的二进制数`