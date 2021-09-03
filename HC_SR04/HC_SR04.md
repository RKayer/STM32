# HC_SR04超声波测距
## HCSR04模块介绍
超声波测距模块

![超声波测距模块实物图](https://gitee.com/RKayer/blogimage/raw/master/img/HC01.png)
四个引脚
|引脚|功能|
|-|-|
|VCC|电源正极|
|Trig|发射引脚(单片机向模块发送信号)|
|Echo|接收引脚(单片机接收信号)|
|GND|接地|

该模块有两个大的发射管，像摄像头一样的，一个是发射超声波，一个是接收超声波，发射和接收之间的时间差值，再根据声音的传播速度就可以计算出距离

**原理**
![模块原理时序图](https://gitee.com/RKayer/blogimage/raw/master/img/HC02.png)
1. 单片机向Trig引脚发送一个不少于10us的高电平信号，模块接收到该信号就会开启
2. 模块会自动向前方发射8个40KHz的声波，与此同时回波引脚（echo）端的电平会自动由低电平变为高电平
3. 当模块接收到反射回来的声波时就会将Echo引脚电平拉低

所以由此我们只要测量Echo的高电平持续时间就可以算出最后的距离

此模块的测量范围为2cm-400cm，声速340m/s，所以一般高电平持续时间10ms这种就比较合理

## Stm32+HC-SR04测距

### 硬件连接
我选择的开发板是STM32f103rb评估板

首先选择四个IO来连接到模块
|模块|开发板|
|-|-|
|Trig|PA1|
|Echo|PA0|
|VCC|+5V|
|GND|GND|

选择USART1作为串口打印distance结果，所以将ST-Link的RX和PA9连接，TX与PA10连接

### STM32Cubemx
1. 设置RCC和时钟树（TIM1为72MHZ）
2. PA0为GPIO_Input,PA1为GPIO_Output
3. 设置TIM1的clocksource时钟源为内部时钟
4. 计数器的PSC设置为71，ARR设置为999，然后使能auto-reload自动重装载（因为我想达到计数器每个CNT加一一次就是1us，计满1000次就是1ms，这样设置的计数周期正好是1ms，每个周期计数1000次）；NVIC settings选择TIM1的更新中断（每次计满一个周期就进行一次中断）
5. 设置USART1为异步模式，波特率默认115200
6. 生成工程

### keil编写代码
按照步骤
	  /*
       *向模块发送10us高电平开启模块超声波发送
       *开始检测
	   *等待Echo检测出高电平
	   *开启定时器2，开始计数
	   *检测出低电平就停止计数
	   *读出并计算计数值和距离
	  */

1. **在main.c文件主函数while(1)中(trig所连接IO初始化输出低电平)**
```c
//先定义三个全局变量
uint16_t Ms_Count = 0;
uint16_t Us_Count = 0;
uint16_t distance = 0;
//向Trig输出10us高电平信号
	  HAL_GPIO_WritePin(Trig_GPIO_Port, Trig_Pin, GPIO_PIN_SET);
	  delay_us(10);
	  HAL_GPIO_WritePin(Trig_GPIO_Port, Trig_Pin, GPIO_PIN_RESET);
```

2. ** 开启/关闭计数 **
```c
  //等待echo输出高电平到单片机
    while(HAL_GPIO_ReadPin(Echo_GPIO_Port,Echo_Pin) != SET);
	  
	  //开始计数，开启定时器中断
	  HAL_TIM_Base_Start_IT(&htim1);
	  
	  //等待计数器计数完成(读取到Echo为低电平表示计数完成)
	  while(HAL_GPIO_ReadPin(Echo_GPIO_Port,Echo_Pin) != RESET);
```
3. **中断逻辑处理**
中断函数中的回调函数逻辑代码主要是标志出进入了几次中断函数（意味着计满多少次，计满一次是1ms）
这里要注意的是我们并不是直接在中断函数中写逻辑代码，而是在更新中断回调函数中编写逻辑代码。因为在HAL库中是中断函数执行之后再进入具体的回调函数中去执行

```c
//更新中断回调函数
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	if(htim == &htim1)
	{
		Ms_Count ++;  //ms + 1
	}
}

```

4. **读取和计算**
```c
//先读取定时器中的us值，再关闭定时器(如果先关闭再读取貌似不行)
Us_Count = __HAL_TIM_GET_COUNTER(&htim1);
	  //关闭定时器
	  __HAL_TIM_SET_COUNTER(&htim1,0);
	  HAL_TIM_Base_Stop_IT(&htim1);

      distance =( Ms_Count * 1000 + Us_Count ) / 58;
      /*
       *time(us) * speed(0.034cm/us) / 2 = distance
       *为了不产生小数计算过程中带来的误差，我们将后面的常量变成整数就是/58
       */
	  printf("distance =  %d \r\n",distance);
//	  printf("Ms_Count =  %d \r\n",Ms_Count);
//	  printf("Us_Count =  %d \r\n",Us_Count);

     // 每次测量一次后要将参数清零以备下一次测量
	  Us_Count = 0;
	  Ms_Count = 0;
	  distance = 0;
	  HAL_Delay(1000); //延时1s等待下一次测量距离
```

5. **在main.c文件中加入串口打印函数的定义**
就可以使用printf来从串口输出数据
```c
#include "stdio.h"
int fputc(int ch, FILE *f){
    uint8_t temp[1] = {ch};
    HAL_UART_Transmit(&huart1, temp, 1, 2);//huart1 根据配置修改
    return ch;
}
```

## 串口现象
![](https://gitee.com/RKayer/blogimage/raw/master/img/HC_distance.png)

**这里的单位是cm，并没有精确到mm**
**串口是每1s测量一次，打印一次**


