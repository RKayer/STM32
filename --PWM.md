# PWM
51里面没有关于PWM的硬件，所以只能自己用定时器去配置产生PWM。
但是在32里面是有PWM外设的。

STM32 的定时器除了 TIM6 和 7。其他的定时器都可以用来产生 PWM 输出。其中高级定
时器 TIM1 和 TIM8 可以同时产生多达 7 路的 PWM 输出。而通用定时器也能同时产生多达 4
路的 PWM 输出，这样，STM32 最多可以同时产生 30 路 PWM 输出

TIMx_ CCRx寄存器中存放着CCR值，这个值通过与CNT的值(定时器的计数值)进行比较，从而做出输出高/低电平的操作，产生PWM。

PWM输出步骤：
```c
//1. 使能定时器(外设)和相关IO口时钟
//RCC函数调用

//2. 初始化IO口的输出模式->复用推挽输出
GPIo_Init()

//3. 初始化定时器，设置ARR和PSC等
TIM_TimeBaseInit()
//4.）设置 TIM1_CH1 的 PWM 模式及通道方向, 使能 TIM1 的 CH1 输出。
void TIM_OC1Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)；
//5. 使能PWM的时钟
TIM_Cmd(TIM1, ENABLE);
//6. 修改占空比
void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1)；
```
在库函数中，我们需要用TIM_SetCompare1()函数去设置占空比，但是在HAL中没有这个函数，只能在初始化中进行设置。

## HAL库实现PWM输出控制LED呼吸灯

打开STM32CubeMX

选择RCC选择HSE高速外部时钟源为晶振

引脚图上选择GPIO配置PA8(LED灯)作为TIM1_CH1输出通道，软件会自动将该IO口设置为复用推挽输出
(输出通道和定时器都不是随便选的，而是要查找手册中的重映射表来选取，比如这里的PA8可以作为定时器1的PWM的CH1输出)

选择TIM1，将Clock sourse配置为内部时钟(internal clock),CH1选择PWM Generatio CH1

然后在下面的Parameter Settings选择关于PWM的一些配置。
> Prescaler : PSC预分频值 --> 72-1
Counter Period: ARR计数最大值 --> 初始化为0就可以
auto-reloa preloa: 自动重装载预装载值 --> ENABLE
PWM Mode : 设置为Mode1
CH Polarity : PWM的极性设置为LOW低有效
(在PWM mode1下当CNT < ARR则为有效电平)

然后配置一下工程名和IDE基本就可以生成工程文件

CubeMX虽然帮我们生成了初始化配置，但是这里我们还需要使能PWM通道，在main.c中可以通过
```c
HAL_StatusTypeDef HAL_TIM_PWM_Start(TIM_HandleTypeDef *htim, uint32_t Channel)
```
实现

接下来是业务代码的编写

ARR的值我们初始化为0，那么要改变这个值我们怎么做呢。这个ARR的值是通过TIMx_CRRx寄存器配置的，所以我们可以直接操作寄存器:
```c
htim1.Instance->CCR1 = ARR；
```
或者通过HAL库中的自带的宏定义来实现，在stm32f1xx_hal_tim.h中有如下定义:
```c
#define __HAL_TIM_SET_COMPARE(__HANDLE__, __CHANNEL__, __COMPARE__) \
  (((__CHANNEL__) == TIM_CHANNEL_1) ? ((__HANDLE__)->Instance->CCR1 = (__COMPARE__)) :\
   ((__CHANNEL__) == TIM_CHANNEL_2) ? ((__HANDLE__)->Instance->CCR2 = (__COMPARE__)) :\
   ((__CHANNEL__) == TIM_CHANNEL_3) ? ((__HANDLE__)->Instance->CCR3 = (__COMPARE__)) :\
   ((__HANDLE__)->Instance->CCR4 = (__COMPARE__)))
```
所以我们可以
```c
__HAL_TIM_SET_COMPARE(&htim1,TIM_CHANNEL_1,ARR);
```
这样来实现改变ARR的值

要想实现呼吸灯，ARR先不断增加-->灯越来越亮
ARR不断减小-->灯越来越暗

那么我们可以定义一个PWM_CRR变量来存储ARR的值

main.c代码：
```c
/**
  * @brief  The application entry point.
  * @retval int
  */
int main(void)
{
  /* USER CODE BEGIN 1 */
	uint8_t flag = RESET;
  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_TIM1_Init();
  MX_USART1_UART_Init();
  /* USER CODE BEGIN 2 */
	HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_1);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
		while(flag == RESET){
		    __HAL_TIM_SET_COMPARE(&htim1,TIM_CHANNEL_1,PWM_CRR);
			//htim1.Instance->CCR1 = PWM_CRR;
			PWM_CRR ++;
			HAL_Delay(1);
			if(PWM_CRR == 700){
			  flag = SET;
			}
		}
		while(flag == SET){
			__HAL_TIM_SET_COMPARE(&htim1,TIM_CHANNEL_1,PWM_CRR);
			//htim1.Instance->CCR1 = PWM_CRR;
			PWM_CRR --;
			HAL_Delay(1);
			if(PWM_CRR == 0){
			  flag = RESET;
			}
		}
		
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```

每次写ARR到寄存器之后要HAL_Delay一段时间，不然灯不会出现亮暗的变化，因为没有Delay的话，每次变化会非常快，从而看不到显著的变化。

另外对于PWM频率的设定，不要低于50HZ(这里设定的是1KHZ)，不然会出现肉眼可见的闪烁(本人亲测可信).

