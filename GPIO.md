# STM32学习笔记

## GPIO

STM32F103RCT6共有51个GPIO：

PA(B,C,D) 0-15 和PD 0-2

### GPIO相关寄存器

#### 端口配置寄存器(GPIOx_CRL和GPIOx_CRH)

端口配置位寄存器用来设置GPIO每个端口的模式

GPIOx_CRL 和 GPIOx_CRH分别控制每组GPIO的低8位和高8位

每个端口由端口配置位寄存器的四个位进行模式控制，分别为CNF1,CNF0,MODE1,MODE0(从高到低位)，故一个端口位配置寄存器是4 * 8 = 32位

| 配置模式 | CNF1 | CNF0 | MODE1 | MODE0 |
|    -    |  -   |   -  |    -   |  -    |
|模拟输入 | 0| 0| 0| 0|
|浮空输入| 0 | 1 | 0 | 0 |
| 下拉输入 | 1 | 0 | 0 | 0|
|上拉输入 | 1 |0| 0 | 0 |
|通用推挽|0 | 0| -|-|
|复用推挽|1|0|-|-|
|通用开漏|0|1|-|-|
|复用开漏|1|1|-|-|

MODE两个位设置为00时为输入模式，其他为输出模式

01 -> 最大输出速度10MHZ

10 -> ..2 MHZ

11 -> ..50MHZ

不过在寄存器中位从高到低每四位是CNF1 MODE1 CNF0 MODE0，在参考手册中进行查找即可

#### 端口输出/输入数据数据寄存器GPIOx_ODR和GPIOx_IDR

GPIOx_ODR中共有32个位，但只用到了低16位，高16位保留，通过向其中写入位值就可以设置对应端口的输出电平高低。

一般我们通过位操作：比如要设置PA8的输出电平位高电平/低电平

```c
GPIOA_ODR |= 1 << 8 //输出高电平
GPIOA_ODR &= ~(1 << 8) //输出低电平
```

对于GPIOx_IDR则一般用于读取电平状态

```c
if(GPIOA_IDR & (1 << 8 ) == 0) // 真对应低电平,假对应高电平
```

#### 端口位设置/清除寄存器GPIOx_BSRR

此寄存器共有32位，高16位为BRy(y=0..15),对应位写1，清除该位为0，写0，该位不变

低16位为BSy(y=0..15),对应位写1，设置该位为1，写0，对应位不变

比如要设置PA8的输出电平位高电平/低电平

```c
GPIOA_BSRR = 1 << (16 + 8)  //设置为高电平
GPIOA_BSRR = 1 << 8  //设置为低电平
```

#### 端口位清除寄存器GPIOx_BRR

32位，高16位保留，低16位用于操作对应端口

对应位写1，清除该位为0，写0，该位不变

这个和GPIOx_BSRR清除位操作功能一致

对于STM32的固件库来说，其中提供了封装好的函数给我们使用，就不需要通过配置寄存器去操作了

比如GPIO_Init()函数用来初始化配置端口的模式

```c
/*
 *@param GPIOx 可以选择GPIOA-G
 *@param GPIO_InitStruct 此结构体中进行配置Pin和模式选择
 */
void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct);

typedef struct
{
  uint16_t GPIO_Pin;     //选择Pin0 - Pin15         
  GPIOSpeed_TypeDef GPIO_Speed; //输出频率
  GPIOMode_TypeDef GPIO_Mode;   //模式选择,对应的8种模式有宏定义选择
}GPIO_InitTypeDef;

void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);  //设置Pin为高电平
void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);  //设置对应位为低电平
```

通过这些固件库函数，我们不需要去手动设置GPIO的寄存器，只需要调用这些函数即可

***示例***
比如我们要实现PA8上面接的LED的闪烁
在keil工程中中我们首先要在加入一个led.c和led.h

```c
//led.h中的代码
//首先要进行一个头文件的初始化定义
# ifndef __LED_H
# define __LED_H
void LED_Init();//函数声明
# endif


//led.c中的代码
# include "led.h"  //
# include "stm32f10x.h" //GPIO相关函数在这里面可以找到

void LED_Init()
{
    GPIO_InitTypeDef GPIO_InitStruct; //定义一个结构体变量

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8; //LED0-->PA.8 端口配置

    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;  //推挽输出

    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; //IO口速度为50MHz

    GPIO_Init(GPIOA, &GPIO_InitStructure);	//根据设定参数初始化GPIOA.8

    GPIO_SetBits(GPIOA,GPIO_Pin_8);	 //PA.8 输出高
}


//在main.c中的代码
#include "led.h"
#include "delay.h"
#include "sys.h"

 int main(void)
 {	
	delay_init();	   //延时函数初始化	  
	LED_Init();	   //初始化与LED连接的硬件接口
	while(1)
	{
        GPIO_SetBits(GPIOA,GPIO_Pin_8); //设置高电平-灭
		delay_ms(300);	 //延时300ms
        GPIO_ResetBits(GPIOA,GPIO_Pin_8); //设置低电平-亮
		delay_ms(300);	//延时300ms
	}
 }
```

## 按键

按键和51的差不多，也是分为消抖，检测和松手检测

RCT6这个板子上与三个按键和一个wakeup按键

对于按键检测，寄存器方面是通过读取GPIO_IDR寄存器来获取端口的电平状态；库函数方面调用了uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)；函数进行读取，返回值就是0/1;



# 关于GPIO端口重映射的理解
STM32的端口可以进行端口复用，所谓复用功能，就是指该端口可以进行普通的GPIO，也可以连接在外设上，作为外设的输入输出。但并不是每个端口都可以随意连接在任何一个外设上，在STM32中有规定特定的端口可以进行哪些功能的复用，对于外设则会规定哪些端口可以作为其端口，这叫做端口的重映射。具体的规范可以查阅STM32官方的参考手册。
比如对于定时器一：
通道所对应的IO口是已经被规范好的，所以我们在使用时要首先进行选择
|外设|-|-|-
|-|-|-|-
|TIM1_ETR  |  PA12  |  PE7 |
TIM1_CH1 |   PA8   | PE9 
TIM1_CH2  |  PA9  |  PE11 
TIM1_CH3   | PA10  |  PE13 
TIM1_CH4  |  PA11 |   PE14 
TIM1_BKIN  |  PB12 |   PA6  |  PE15 
TIM1_CH1N   | PB13  |PA7   | PE8 
TIM1_CH2N   | PB14 | PB0 |PE10 
TIM1_CH3N   | PB15  |  PB1|    PE12 








