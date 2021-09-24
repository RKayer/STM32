# 酒精检测自创实验

![](https://gitee.com/RKayer/blogimage/raw/master/img/zc01.png)
## MQ-3酒精检测模块
MQ-3酒精检测模块可以将酒精浓度的变化转换为电压值从而通过单片机的ADC转换读取出来
![](https://gitee.com/RKayer/blogimage/raw/master/img/zc03.png)

这里有AO和DO两个输出引脚，AO是指模拟量输出，0-5V的模拟量。而DO是指TTL输出，0.1v对应数字量0，5V高电平对应数字量1.
这里用模拟量输出就行

## 压力传感器模块
这个模块实质是一个压敏电阻，压力的大小可以改变阻值从而影响电流的电压的大小，然后通过压力电压转换模块可以转换为模拟量输出。

## 继电器
继电器其实相当于一个开关，COM口，NC端(常闭端)和NO端(常开端)，COM口和NO端连接到电路，一开始COM口和NC端连接，当满足触发条件COM口就会和NO端连接。触发条件是根据另外一个IN引脚决定的，如果设置的是低电平触发，那么当IN口输出低电平时就会将COM口和NO端闭合，从而实现开关闭合的功能。
继电器一般是用来低电压控制高电压电路的器件，从而降低直接控制高电压的风险。
![](https://gitee.com/RKayer/blogimage/raw/master/img/zc02.png)

