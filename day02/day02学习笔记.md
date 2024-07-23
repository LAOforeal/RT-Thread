# 实时系统的引入

实习系统是指能在指定时间内完成规定功能并且对外部异步事件进行响应的计算机系统

## 实时系统分为

硬实时系统：结果迟到产生灾难性的后果

FIRM实习系统：结果迟到产生难以接受的质量上的降低

软实时系统：结果迟到产生质量下降，但是系统可以自主恢复

## 衡量实时性的指标

本地响应时间

生存时间

吞吐量

## 嵌入式实时系统的功能介绍

![](C:\Users\LAO\Desktop\14.jpg)

# RT-Thread的引入

## 传统操作系统(Linux&&RTOS)

![](C:\Users\LAO\Desktop\21.png)

> 传统的操作系统Linux重点在于通用泛用，但是存在着移植难度大，开销大，实时性并不保证的缺点
>
> RTOS轻量化，保证实时性，但是其功能较少需要额外的成本或者集成工作，仅能满足部分单片机的全部使用

> 而RT-Thread拥有更好的性能，相比前两家兼具其优，并且配有开发社区预计丰富的外设资源

# 裸机开发模式

## 后台

后台是一个无限的循环，其中调用相应的处理函数，完成相应的操作

ADC

SPI

USB

LCD

Audio_Decode

File_Write

## 前台

```c
void USB_ISR(void)
{
	Clear interrupt;
	Read packet;
}
```

> 逻辑模式的缺陷在于后台循环调用的函数中如果存在延时函数那么对于整个系统的实时性会造成影响

# RT-Thread启动流程

## 多任务系统

![](C:\Users\LAO\Desktop\41.png)

## 启动流程分析

![](C:\Users\LAO\Desktop\42.png)

## 线程控制块

## 构成

线程控制块是一种结构体双向链表，其主要结构体为struct rt_thread定义并形成线程内核对象，再链接到内核对象容器中进行管理

## 线程控制块继承关系

![](C:\Users\LAO\Desktop\51.png)

## 线程控制块的定义

一个线程包括有一个线程栈一个入口函数以及一个线程控制块

下面展示struct rt_thread的定义

```
struct rt_thread
{
	char name[RT_NAME_MAX];
	rt_unit8_t type;
	rt_unit8_t flags;
	
	rt_list_t list;
	rt_list_t tlist;
	
	void *sp;
	void *entry
	void *parameter
	void *stack_addr;
	rt_unit32_t stack_size;
	
	rt_err_t error;
	rt_unit8_t stat;
	
	rt_unit8_t current_priority;
	rt_unit8_t init_priority;
	rt_unit32_t number_mask;
	...
	rt_ubase_t init_tick;
	rt_ubase_t remaining_tick;
	
	struct rt_timer thread_timer;
	
	void(*cleanip)(struct rt_thread*tid);
	rt_unit32_t user_data;
}
```

> 从上往下有线程名称，对象类型，标志位，对象列表，栈指针等待，包含很多方面的结构体描述

## 线程栈

RT-Thread的线程具有独立的空间栈，当进行线程转换时，当前线程的上下文会存在栈中。

线程栈还可以用来存放函数中的局部变量

## 线程状态以及转换

![](C:\Users\LAO\Desktop\52.png)

![](C:\Users\LAO\Desktop\53.png)

## 线程属性

### 优先级

有专门的优先级位，数值越小的优先级越高

### 时间片

每个线程都需要有时间片，当时间片仅对优先级相同的就绪态线程有效。系统对优先级相同的就绪态线程进行调度时，时间片起到的作用为单次运行时间，其单位时一个系统节拍（OSTick)

### 入口函数

![](C:\Users\LAO\Desktop\54.png)

### 错误码

![](C:\Users\LAO\Desktop\55.png)

# 系统线程

## 空闲线程

最低优先级的线程，其状态永远保持为就绪态，当系统无其他就绪态存在时，系统将进入空闲线程，且一般都是死循环，且永远不能被挂起

有着几个重要的作用

1.回收被删除线程的资源

2.提供狗子函数的接口，或者看门狗喂狗

# 线程管理API

## 线程初始化

![](C:\Users\LAO\Desktop\56.png)

![](C:\Users\LAO\Desktop\57.png)

![](C:\Users\LAO\Desktop\58.png)

## 线程启动

rt_err_t rt_thread_startup(rt_thread_t thread)

调用这个函数时，将把线程的状态更改为就绪状态，并根据优先级队列中等待调度。

## 线程延时

抢占时系统线程在不使用CPU时需要让出CPU的使用权，让其他线程得以运行。

通常有如下三个函数

```c
rtt_err_t rt_thread_sleep(rt_tick_t tick);//OS Tick为单位
rtt_err_t rt_thread_delay(rt_tick_t tick);//OS Tick为单位
rtt_err_t rt_thread_mdelay(rt_int32_t ms);//以毫秒为单位，这个最小
```

# 学习示例

## 程序流程

分别定义一个静态线程和一个动态线程

在两者成功建立时向电脑发送ready确认

而后在进程中有进行GPIO引脚亮灭的任务，两个进程控制的GPIO引脚各不相同

## 初始化部分(头文件和GPIO)

```c
#include <board.h>
#include <rtthread.h>
#include <drv_gpio.h>
#ifndef RT_USING_NANO
#include <rtdevice.h>
#endif /* RT_USING_NANO */

#define GPIO_LED_B    GET_PIN(F, 11)
#define GPIO_LED_R    GET_PIN(F, 12)
```

## 初始化部分（静态线程）

```c
//静态线程任务
static rt_uint8_t thread1_stack[512];   		  //线程栈
static struct rt_thread static_thread;			  //线程控制块
static void static_entry(void *param)
{
	static int cnt = 0;
    while (cnt++<2)
    {
		rt_kprintf("static_thread is run:%d\n",cnt);
        rt_pin_write(GPIO_LED_B, PIN_LOW);       
        rt_thread_mdelay(1000);
        rt_pin_write(GPIO_LED_B, PIN_HIGH);
        rt_thread_mdelay(500);
    }
}

```

## 初始化部分（动态线程）

```c
//动态线程任务
static void dynamic_entry(void *param)
{
	static int cnt = 0;
    while (cnt++<2)
    {
		rt_kprintf("dynamic_thread is run:%d\n",cnt);
        rt_pin_write(GPIO_LED_R, PIN_LOW);       
        rt_thread_mdelay(1000);
        rt_pin_write(GPIO_LED_R, PIN_HIGH);
        rt_thread_mdelay(500);
    }
}
```

## 建立线程

```c
int thread_sample(void)
{
    static rt_thread_t thread_id = RT_NULL;
    thread_id = rt_thread_create("dynamic_th",    //名称
                                 dynamic_entry,   //线程代码
                                 RT_NULL,         //参数
                                 1024,            //栈大小
                                 15,              //优先级
                                 20);             //时间片

    if (thread_id != RT_NULL)
    {
        rt_thread_startup(thread_id);			  //线程进入就绪态
        rt_kprintf("dynamic_th ready!");

    }
	thread_id=rt_thread_init(&static_thread,							//线程handle
								 "static_thread",			//线程名称
								 static_entry,				//线程入口函数
								 RT_NULL,					//线程入口参数
								 &thread1_stack[0],       	//线程栈地址
								 sizeof(thread1_stack),	  	//线程栈大小
								 15, 	 					//线程优先级							    		
								 5);						//线程时间片		 						/.	  		
    if (thread_id != RT_NULL)
    {
        rt_thread_startup(thread_id);			  //线程进入就绪态
        rt_kprintf("static_th ready!");
    }														 
    return RT_EOK;									 
}
/* 导出到 msh 命令列表中 */
MSH_CMD_EXPORT(thread_sample, thread sample);
```

## 主函数部分

```c
int main(void)
{  
    rt_err_t ERR;
    rt_pin_mode(GPIO_LED_R, PIN_MODE_OUTPUT);
    rt_pin_mode(GPIO_LED_B, PIN_MODE_OUTPUT);

    rt_pin_write(GPIO_LED_R, PIN_HIGH);
    rt_pin_write(GPIO_LED_B, PIN_HIGH);

    ERR=thread_sample();
    while(ERR==RT_EOK);
    while(1);
    return RT_EOK;
}

```

## 结果展示

开发板上的灯正常亮灭，但是串口没有发送数据不知为何仍在努力中