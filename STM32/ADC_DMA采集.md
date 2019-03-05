# ADC

#### 对STM32 ADC单次转换模式 连续转换模式 扫描模式的理解  
举例:  

* 用ADC1 规则通道的顺序为CH0,CH1,CH2,CH3，不启动SCAN模式  

在单次转换模式下：	

启动ADC1，则	

1.开始转换CH1（ADC_SQR的第一通道）	

转换完成后停止，等待ADC的下一次启动，继续从第一步开始转换	

在连续转换模式下：	

启动ADC1，则	

1.开始转换CH0（ADC_SQR的第一通道）

转换完成后回到第一步。



* 启动SCAN扫描模式下

在单次转换模式下：

启动ADC1，则

1.开始转换CH0、

2.转换完成后自动开始转换CH1

3.转换完成后自动开始转换CH2

4.转换完成后自动开始转换CH3

5.转换完成后停止，等待ADC的下一次启动下一次ADC启动后从第一步开始转换

在连续转换模式下：

启动ADC1，则1.开始转换CH0、

2.转换完成后自动开始转换CH1

3.转换完成后自动开始转换CH2

4.转换完成后自动开始转换CH3	 

5.转换完成后返回第一步 


**开启扫描模式后，必须搭配DMA功能才能实现ADC的数据处理（软件读取不及时，用DMA保证数据不丢失）**
--------------------- 
原文：https://blog.csdn.net/kiti1013/article/details/44172161 

#### ADC规则组和注入组的区别

STM32的ADC可以对一组指定的通道，按照指定的顺序，逐个转换这组通道，转换结束后，再从头循环；这指定的通道组就称为规则组。但是实际应用中，有可能需要临时中断规则组的转换，对某些通道进行转换，这些需要中断规则组而进行转换的通道组，就称为注入组。

对于ADC模块来说，它按规则转换规则组时，被要求临时转换规则组之外的某些通道，就好像这组通道临时注入了原来的顺序，所以形象地称其为注入组。

有2种划分转换组的方式：规则通道组和注入通道组。通常规则通道组中可以安排最多16个通道，而注入通道组可以安排最多4个通道。

**单通道连续转换，DMA读取（不开扫描）**

使用ADC1的通道11，对应IO口为PC1（需要设置为模拟输入）  

代码如下：


```C
#include "wk_adc.h"

volatile uint16_t ADC_ConvertedValue;//DMA读取AD值暂存SRAM

/**
  * @brief  ADC1 GPIO 初始化
  * @param  无
  * @retval 无
  */
static void ADC1_GPIO_Config(void)	//PC1--CH11
{
	GPIO_InitTypeDef GPIO_InitStructure;

	// 打开 ADC IO端口时钟
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_GPIOC, ENABLE );

	// 配置 ADC IO 引脚模式
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_11;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;

	// 初始化 ADC IO
	GPIO_Init(GPIOC, &GPIO_InitStructure);				
}

/**
  * @brief  配置ADC工作模式
  * @param  无
  * @retval 无
  */
static void ADC1_Mode_Config(void)
{
	DMA_InitTypeDef DMA_InitStructure;
	ADC_InitTypeDef ADC_InitStructure;

	// 打开DMA时钟
	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
	// 打开ADC时钟
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_ADC1, ENABLE );

	// 复位DMA控制器
	DMA_DeInit(DMA1_Channel1);

	// 配置 DMA 初始化结构体
	// 外设基址为：ADC 数据寄存器地址
	DMA_InitStructure.DMA_PeripheralBaseAddr = ( uint32_t ) ( & ( ADC1->DR ) );

	// 存储器地址，实际上就是一个内部SRAM的变量
	DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)&ADC_ConvertedValue;

	// 数据源来自外设
	DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;

	// 缓冲区大小为1，缓冲区的大小应该等于存储器的大小
	DMA_InitStructure.DMA_BufferSize = 1;

	// 外设寄存器只有一个，地址不用递增
	DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;

	// 存储器地址固定
	DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Disable; 

	// 外设数据大小为半字，即两个字节
	DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;

	// 存储器数据大小也为半字，跟外设数据大小相同
	DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;

	// 循环传输模式
	DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;	

	// DMA 传输通道优先级为高，当使用一个DMA通道时，优先级设置不影响
	DMA_InitStructure.DMA_Priority = DMA_Priority_High;

	// 禁止存储器到存储器模式，因为是从外设到存储器
	DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;

	// 初始化DMA
	DMA_Init(DMA1_Channel1, &DMA_InitStructure);

	// 使能 DMA 通道
	DMA_Cmd(DMA1_Channel1 , ENABLE);

	// ADC 模式配置
	// 只使用一个ADC，属于单模式
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;

	// 禁止扫描模式，多通道才要，单通道不需要
	ADC_InitStructure.ADC_ScanConvMode = DISABLE ; 

	// 连续转换模式
	ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;

	// 不用外部触发转换，软件开启即可
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;

	// 转换结果右对齐
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;

	// 转换通道1个
	ADC_InitStructure.ADC_NbrOfChannel = 1;	
		
	// 初始化ADC
	ADC_Init(ADC1, &ADC_InitStructure);

	// 配置ADC时钟为PCLK2的8分频，即9MHz
	RCC_ADCCLKConfig(RCC_PCLK2_Div8); 

	// 配置 ADC 通道转换顺序为1，第一个转换，采样时间为55.5个时钟周期
	ADC_RegularChannelConfig(ADC1, ADC_Channel_11, 1, ADC_SampleTime_55Cycles5);

	// 使能ADC DMA 请求
	ADC_DMACmd(ADC1, ENABLE);

	// 开启ADC ，并开始转换
	ADC_Cmd(ADC1, ENABLE);

	// 初始化ADC 校准寄存器  
	ADC_ResetCalibration(ADC1);
	// 等待校准寄存器初始化完成
	while(ADC_GetResetCalibrationStatus(ADC1));

	// ADC开始校准
	ADC_StartCalibration(ADC1);
	// 等待校准完成
	while(ADC_GetCalibrationStatus(ADC1));

	// 由于没有采用外部触发，所以使用软件触发ADC转换 
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}

/**
  * @brief  ADC初始化
  * @param  无
  * @retval 无
  */
void ADC1_Init(void)
{
	ADC1_GPIO_Config();
	ADC1_Mode_Config();
}
/*********************************************END OF FILE**********************/
```
获取实际电压值：  
```c
temp_data = (float)ADC_ConvertedValue/4096*3.3;
```



**多通道连续转换，DMA读取（开启扫描）**

使用ADC1的通道10~15，对应IO口为PC0~PC5（需要设置为模拟输入）

代码如下：


```C
#include "wk_adc.h"

volatile uint16_t ADC_ConvertedValue[NOFCHANEL]={0,0,0,0};//DMA读取AD值暂存SRAM  NOFCHANEL：转换通道个数，这里为6

/**
  * @brief  ADC GPIO 初始化
  * @param  无
  * @retval 无
  */
static void ADC1_GPIO_Config(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;

	// 打开 ADC IO端口时钟
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_GPIOC, ENABLE );

	// 配置 ADC IO 引脚模式
	GPIO_InitStructure.GPIO_Pin = 	GPIO_Pin_0|
									GPIO_Pin_1|
									GPIO_Pin_2|
									GPIO_Pin_3|
									GPIO_Pin_4|
									GPIO_Pin_5;

	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;

	// 初始化 ADC IO   PC0-PC5对应通道CH10-CH15
	GPIO_Init(GPIOC, &GPIO_InitStructure);				
}

/**
  * @brief  配置ADC工作模式
  * @param  无
  * @retval 无
  */
static void ADC1_Mode_Config(void)
{
	DMA_InitTypeDef DMA_InitStructure;
	ADC_InitTypeDef ADC_InitStructure;

	// 打开DMA时钟
	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
	// 打开ADC时钟
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_ADC1, ENABLE );

	// 复位DMA控制器
	DMA_DeInit(DMA1_Channel1);

	// 配置 DMA 初始化结构体
	// 外设基址为：ADC 数据寄存器地址
	DMA_InitStructure.DMA_PeripheralBaseAddr = ( u32 ) ( & ( ADC1->DR ) );//ADC只有这一个数据寄存器

	// 存储器地址
	DMA_InitStructure.DMA_MemoryBaseAddr = (u32)ADC_ConvertedValue;	//DMA把AD值读出来存放的位置

	// 数据源来自外设
	DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;

	// 缓冲区大小，应该等于数据目的地的大小
	DMA_InitStructure.DMA_BufferSize = NOFCHANEL;	//缓存区等于转换通道个数，这里为6

	// 外设寄存器只有一个，地址不用递增
	DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;

	// 存储器地址递增
	DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable; 

	// 外设数据大小为半字，即两个字节
	DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;

	// 内存数据大小也为半字，跟外设数据大小相同
	DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;

	// 循环传输模式
	DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;	

	// DMA 传输通道优先级为高，当使用一个DMA通道时，优先级设置不影响
	DMA_InitStructure.DMA_Priority = DMA_Priority_High;

	// 禁止存储器到存储器模式，因为是从外设到存储器
	DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;

	// 初始化DMA
	DMA_Init(DMA1_Channel1, &DMA_InitStructure);

	// 使能 DMA 通道
	DMA_Cmd(DMA1_Channel1 , ENABLE);

	// ADC 模式配置
	// 只使用一个ADC，属于单模式
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;

	// 扫描模式
	ADC_InitStructure.ADC_ScanConvMode = ENABLE ; 

	// 连续转换模式
	ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;

	// 不用外部触发转换，软件开启即可
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;

	// 转换结果右对齐
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;

	// 转换通道个数
	ADC_InitStructure.ADC_NbrOfChannel = NOFCHANEL;	//转换通道个数，这里为6
		
	// 初始化ADC
	ADC_Init(ADC1, &ADC_InitStructure);

	// 配置ADC时钟预分频CLK2的8分频，即9MHz
	RCC_ADCCLKConfig(RCC_PCLK2_Div8); 

	// 配置ADC 通道的转换顺序和采样时间
	ADC_RegularChannelConfig(ADC1, ADC_Channel_10, 1, ADC_SampleTime_55Cycles5);
	ADC_RegularChannelConfig(ADC1, ADC_Channel_11, 2, ADC_SampleTime_55Cycles5);
	ADC_RegularChannelConfig(ADC1, ADC_Channel_12, 3, ADC_SampleTime_55Cycles5);
	ADC_RegularChannelConfig(ADC1, ADC_Channel_13, 4, ADC_SampleTime_55Cycles5);
	ADC_RegularChannelConfig(ADC1, ADC_Channel_14, 5, ADC_SampleTime_55Cycles5);
	ADC_RegularChannelConfig(ADC1, ADC_Channel_15, 6, ADC_SampleTime_55Cycles5);

	// 使能ADC DMA 请求
	ADC_DMACmd(ADC1, ENABLE);

	// 开启ADC ，并开始转换
	ADC_Cmd(ADC1, ENABLE);

	// 初始化ADC 校准寄存器  
	ADC_ResetCalibration(ADC1);
	// 等待校准寄存器初始化完成
	while(ADC_GetResetCalibrationStatus(ADC1));

	// ADC开始校准
	ADC_StartCalibration(ADC1);
	// 等待校准完成
	while(ADC_GetCalibrationStatus(ADC1));

	// 由于没有采用外部触发，所以使用软件触发ADC转换 
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}

/**
  * @brief  ADC初始化
  * @param  无
  * @retval 无
  */
void ADC1_Init(void)
{
	ADC1_GPIO_Config();
	ADC1_Mode_Config();
}
/*********************************************END OF FILE**********************/
```
获取实际电压值：  
```c
temp_data[0] =(float) ADC_ConvertedValue[0]/4096*3.3;
temp_data[1] =(float) ADC_ConvertedValue[1]/4096*3.3;
temp_data[2] =(float) ADC_ConvertedValue[2]/4096*3.3;
temp_data[3] =(float) ADC_ConvertedValue[3]/4096*3.3;
temp_data[4] =(float) ADC_ConvertedValue[4]/4096*3.3;
temp_data[5] =(float) ADC_ConvertedValue[5]/4096*3.3;
```

**双ADC连续转换（同步规则），DMA读取（必须DMA使能）**

使用ADC1的通道11和ADC2的通道14，对应IO口为PC1和PC4（需要设置为模拟输入）

代码如下：

```C
// 双模式时，ADC1和ADC2转换的数据都存放在ADC1的数据寄存器，
// ADC1的在低十六位，ADC2的在高十六位
// 双ADC模式的第一个ADC，必须是ADC1
#include "wk_adc.h"

volatile uint32_t ADC_ConvertedValue[NOFCHANEL] = {0};

/**
  * @brief  ADC GPIO 初始化
  * @param  无
  * @retval 无
  */
static void ADCx_GPIO_Config(void) 	//ADC1的PC1，对应CH11
									//ADC2的PC4，对应CH14
{
	GPIO_InitTypeDef GPIO_InitStructure;

	// ADCx_1 GPIO 初始化
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_GPIOC, ENABLE );
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_Init(GPIOC, &GPIO_InitStructure);

	// ADCx_2 GPIO 初始化
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_GPIOC, ENABLE );
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_Init(GPIOC, &GPIO_InitStructure);	
}

/**
  * @brief  配置ADC工作模式
  * @param  无
  * @retval 无
  */
static void ADCx_Mode_Config(void)
{
	DMA_InitTypeDef DMA_InitStructure;
	ADC_InitTypeDef ADC_InitStructure;

	// 打开DMA时钟
	RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);
	// 打开ADC时钟
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_ADC1, ENABLE );
	RCC_APB2PeriphClockCmd ( RCC_APB2Periph_ADC2, ENABLE );

	/* ------------------DMA模式配置---------------- */	
	// 复位DMA控制器
	DMA_DeInit(DMA1_Channel1);	
	// 配置 DMA 初始化结构体
	// 外设基址为：ADC 数据寄存器地址
	DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)(&( ADC1->DR ));	
	// 存储器地址
	DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)ADC_ConvertedValue;	
	// 数据源来自外设
	DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;	
	// 缓冲区大小，应该等于数据目的地的大小
	DMA_InitStructure.DMA_BufferSize = NOFCHANEL;	
	// 外设寄存器只有一个，地址不用递增
	DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
	// 存储器地址递增
	DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable; 	
	// 外设数据大小
	DMA_InitStructure.DMA_PeripheralDataSize = 
									  DMA_PeripheralDataSize_Word;	
	// 内存数据大小，跟外设数据大小相同
	DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_Word;	
	// 循环传输模式
	DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;	
	// DMA 传输通道优先级为高，当使用一个DMA通道时，优先级设置不影响
	DMA_InitStructure.DMA_Priority = DMA_Priority_High;	
	// 禁止存储器到存储器模式，因为是从外设到存储器
	DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;	
	// 初始化DMA
	DMA_Init(DMA1_Channel1, &DMA_InitStructure);	
	// 使能 DMA 通道
	DMA_Cmd(DMA1_Channel1 , ENABLE);

	/* ----------------ADCx_1 模式配置--------------------- */
	// 双ADC的规则同步
	ADC_InitStructure.ADC_Mode = ADC_Mode_RegSimult;	
	// 扫描模式
	ADC_InitStructure.ADC_ScanConvMode = ENABLE ; 
	// 连续转换模式
	ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;
	// 不用外部触发转换，软件开启即可
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
	// 转换结果右对齐
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;	
	// 转换通道个数
	ADC_InitStructure.ADC_NbrOfChannel = NOFCHANEL;			
	// 初始化ADC
	ADC_Init(ADC1, &ADC_InitStructure);	
	// 配置ADC时钟Ｎ狿CLK2的8分频，即9MHz
	RCC_ADCCLKConfig(RCC_PCLK2_Div8); 	
	// 配置ADC 通道的转换顺序和采样时间
	ADC_RegularChannelConfig(ADC1, ADC_Channel_11, 1, 
							 ADC_SampleTime_239Cycles5);	
	// 使能ADC DMA 请求
	ADC_DMACmd(ADC1, ENABLE);


		/* ----------------ADCx_2 模式配置--------------------- */
	// 双ADC的规则同步
	ADC_InitStructure.ADC_Mode = ADC_Mode_RegSimult;	
	// 扫描模式
	ADC_InitStructure.ADC_ScanConvMode = ENABLE ; 
	// 连续转换模式
	ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;
	// 不用外部触发转换，软件开启即可
	ADC_InitStructure.ADC_ExternalTrigConv = 
							   ADC_ExternalTrigConv_None;
	// 转换结果右对齐
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;	
	// 转换通道个数
	ADC_InitStructure.ADC_NbrOfChannel = NOFCHANEL;			
	// 初始化ADC
	ADC_Init(ADC2, &ADC_InitStructure);	
	// 配置ADC时钟为PCLK2的8分频，即9MHz
	RCC_ADCCLKConfig(RCC_PCLK2_Div8); 	
	// 配置ADC 通道的转换顺序和采样时间
	ADC_RegularChannelConfig(ADC2, ADC_Channel_14, 1, 
							 ADC_SampleTime_239Cycles5);
	/* 使能ADCx_2的外部触发转换 */
	ADC_ExternalTrigConvCmd(ADC2, ENABLE);

	/* ----------------ADCx_1 校准--------------------- */
	// 开启ADC ，并开始转换
	ADC_Cmd(ADC1, ENABLE);	
	// 初始化ADC 校准寄存器  
	ADC_ResetCalibration(ADC1);
	// 等待校准寄存器初始化完成
	while(ADC_GetResetCalibrationStatus(ADC1));	
	// ADC开始校准
	ADC_StartCalibration(ADC1);
	// 等待校准完成
	while(ADC_GetCalibrationStatus(ADC1));

	/* ----------------ADCx_2 校准--------------------- */
		// 开启ADC ，并开始转换
	ADC_Cmd(ADC2, ENABLE);	
	// 初始化ADC 校准寄存器  
	ADC_ResetCalibration(ADC2);
	// 等待校准寄存器初始化完成
	while(ADC_GetResetCalibrationStatus(ADC2));	
	// ADC开始校准
	ADC_StartCalibration(ADC2);
	// 等待校准完成
	while(ADC_GetCalibrationStatus(ADC2));

	// 由于没有采用外部触发，所以使用软件触发ADC转换 
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}

/**
  * @brief  ADC初始化
  * @param  无
  * @retval 无
  */
void ADCx_Init(void)
{
	ADCx_GPIO_Config();
	ADCx_Mode_Config();
}
/*********************************************END OF FILE**********************/
```