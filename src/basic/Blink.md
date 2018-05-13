# keil工程之点灯

本节介绍在`Keil`环境下建立的第一个简单工程，完成点灯功能。将从`DEMO`工程解析入手，介绍基于`Keil`的`STM32`开发中的基本操作。



所需文件：
- `Demo`文件`STM32-DEMO.zip`
- 工程配置好后的空工程模板

---

## Demo工程解析

与空工程模板相比，DEMO工程的差异主要在于`port.c/h`, `led.c/h`,当然还有主函数`main.c/h`,在这一部分将逐一介绍。

### main文件
```C
#include "main.h"
#include "port.h"
#include "led.h"
int main(void)
{
    //Start with board specific hardware init.
	Our_Sys_Init();

	// initialization for the led GPIO
	LED_GPIO_Config();

	//Infinite loop to deal with your project.
	while(1)
	{
		// turn on the LED
		LED(ON);
		Delay(1000);
		// turn off the LED
		LED(OFF);
		Delay(1000);
	}
}
```
如上，`Keil`中代码的写法与`Visual Studio`中普通的C语言工程相同，从`main`函数可以看出整个工程的架构：`头文件包含`、`底层系统初始化和端口初始化`、`逻辑层循环（此工程中为LED的亮灭及延时）`
对初学者较为陌生的是`Our_Sys_Init()`、`LED_GPIO_Config()`和`Delay(1000)`这样与硬件层相关的函数，而它们即在`port.c/h`和`led.c/h`中实现。

### LED文件

```C
#include "led.h"
void LED_GPIO_Config(void)
{		
    /*定义一个GPIO_InitTypeDef类型的结构体*/
    GPIO_InitTypeDef GPIO_InitStructure;
	
    /*开启GPIOD的外设时钟*/
    RCC_APB2PeriphClockCmd( RCC_APB2Periph_GPIOD, ENABLE); 

    /*选择要控制的GPIOD引脚*/	
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;	

    /*设置引脚模式为通用推挽输出*/
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;   

    /*设置引脚速率为50MHz*/   
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz; 

    /*调用库函数，初始化GPIOD*/
    GPIO_Init(GPIOD, &GPIO_InitStructure);		  

    /*关闭所有led灯*/
    GPIO_SetBits(GPIOD, GPIO_Pin_2);	 
}
```
`led.c`的功能很简单，只是实现了`LED_GPIO`的初始化，而`GPIO`的初始化也是在后续`STM32`开发中的必需一步，代码变化量很小，主要流程如下：

`定义结构体`->`设置相关参数`->`调用库函数初始化`->`定义初始状态为灯灭`
标准库中有很多结构体的相关定义，`右键单击结构体类型`->`Go to Definition of xxx`即可看到相关参数的信息：
```C
/** 
  * @brief  GPIO Init structure definition  
  */

typedef struct
{
  uint16_t GPIO_Pin;             /*!< Specifies the GPIO pins to be configured.
                                      This parameter can be any value of @ref GPIO_pins_define */

  GPIOSpeed_TypeDef GPIO_Speed;  /*!< Specifies the speed for the selected pins.
                                      This parameter can be a value of @ref GPIOSpeed_TypeDef */

  GPIOMode_TypeDef GPIO_Mode;    /*!< Specifies the operating mode for the selected pins.
                                      This parameter can be a value of @ref GPIOMode_TypeDef */
}GPIO_InitTypeDef;
```
同样的右键选择方式，可以看到各个结构体的枚举变量：
```C
typedef enum
{ GPIO_Mode_AIN = 0x0,
  GPIO_Mode_IN_FLOATING = 0x04,
  GPIO_Mode_IPD = 0x28,
  GPIO_Mode_IPU = 0x48,
  GPIO_Mode_Out_OD = 0x14,
  GPIO_Mode_Out_PP = 0x10,
  GPIO_Mode_AF_OD = 0x1C,
  GPIO_Mode_AF_PP = 0x18
}GPIOMode_TypeDef;
```
根据工程需求进行选择即可，在点灯工程中通常采用推挽输出` GPIO_Mode_Out_PP`。

而在外设时钟开启和引脚的选择上，需参照`Datasheet`。
在`DEMO`文件夹中可参照相关文件夹。打开`103RCT6-Datasheet.pdf`。`P31-37`可以找到各个端口的相关信息。此处我们使用`PD2`端口控制的`LED`灯。`p12-13`是芯片的时钟树，可以找到`GPIOD`是挂载在外围总线`APB2`上，故需开启对应的时钟并初始化对应的端口。同样，关于上述`GPIO`模式的设置，在`103RCT6-Ref.pdf`及其中文版中也可以找到相关寄存器和端口位配置说明，至此上述`LED.c`中的代码均可读懂了。

```C
#ifndef __LED_H
#define	__LED_H
#include "stm32f10x.h"

#define ON  1
#define OFF 0

#define LED(a)	if (a)	\
                GPIO_SetBits(GPIOD,GPIO_Pin_2);\
                else	\
                GPIO_ResetBits(GPIOD,GPIO_Pin_2)

void LED_GPIO_Config(void);
#endif /*__LED_H*/
```
在`led.h`中包含了标准库头文件、利用宏定义简化控制灯亮灭的函数的表达，并对`led.c`中的函数作了声明。
值得注意的是在工程中的每一个头文件开头和结尾都会有`#ifndef __xxx`、`#define __xxx`、`#endif /* __xxx */`以避免头文件的重定义，在多文件工程中更是必要操作，定义的内容没有要求，只需满足唯一性即可。


### Port文件
剩下的关于时钟和系统初始化的操作均在这部分文件中进行。在这一部分只对port文件的框架加以介绍，在以后的工程建立中，可再利用此框架做进一步的开发。
```C
void Our_Sys_Init(void)
{
	RCC_init();
	GPIO_init();
	NVIC_init();
	EXTI_init();
	Periph_init();
}
```
该文件实现的功能就是在主文件调用的`Our_Sys_Init()`函数，如上可见，该函数按顺序实现`系统时钟配置`、`GPIO初始化`、`NVIC中断配置`、`EXTI外部中断配置`、`外设初始化`。
  * `RCC_init()`此函数与`RCC_ClocksTypeDef RCC_Clocks`一起实现功能，相关时钟设置较为复杂，可参考`103RCT6-Ref.pdf`，在很多工程中直接复制即可。
  * `GPIO_init()`端口初始化，其实上一部分LED相关的端口初始化即可在此函数中实现。
  * `NVIC_init()`、`EXTI_init()`涉及有关于时钟树、中断的操作，会在后续的文档中加以介绍。
  * `Periph_init()`外设初始化，在点灯工程中实现系统定时器初始化`Systick_init()`，同样关于`NVIC`的设置会在后续文档中介绍。`port`文件中也有`Delay`延时函数的相关实现，需在`stm32f10x_it.c`文件的` SysTick_Handler()`中添加`TimingDelay_Decrement()`调用。


## Keil相关操作说明
能够看懂DEMO文件，就可以开始基本`Keil`工程的开发了。除了上文所提到的头文件中防止重定义的方式，`Keil`环境下的开发还有一些习惯上的基本操作：
* 该环境下的新建文件、加入文件等操作与`Visual Studio`类似，故不再赘述。值得注意的一点是`.c`文件和`.h`文件是同样的添加方式，但是`.h`文件添加后不会在左方的目录中显示，在编译中后会添加到`.c`文件目录下。
* 每个代码文件最后一行请留一行空行，否则会出现`Warning`
* 更改文件后`Keil`不会立即去掉报错的红叉或是波浪线，在编译后或者剪切粘贴后相关报错信息会自行消失。
* 最后延续一个传统，如果遇到疑问，可以尝试下几种解决方案：
  1. 参照`Datasheet`,可自行百度或者 `Google`或者`CSDN`……
  2. 与同学讨论 （包括但不限于同班同学、 同年级同学、 学长学姐）
  3. 到电设相关网站、github等平台提问
  4. 如果上述方案均未能解决你的问题，那……我也无为力了……
