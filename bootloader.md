# Mcu IAP

## IAP简介	

​	一般情况下对芯片进行固件烧写都是使用芯片定义好的编程口SWD或者JTAG，如果想要使用串口烧写固件需要修改BOOT0和BOOT1的引脚接线，关于BOOT引脚不同接线下芯片复位后如何启动可参考这篇文章。{[关于stm32程序烧写BOOT1和BOOT0的设置问题_什么样的芯片需要烧写boot1-CSDN博客](https://blog.csdn.net/hurryuptowang/article/details/116567589)}

​	IAP的全程为In application programing，即在应用程序编程。主要作用就是在产品开发完成以后方便对产品的固件进行升级，一般情况下产品都是使用外壳封装好的，极少数会将程序烧录调试口SWD和JTAG开放出来，这时候就需要通过预留的通信接口对固件进行升级，如果产品有无线通信协议可以使用更为高级的OTA对固件升级。

​	IAP一般需要两段程序实现，第一段叫Bootloader，即引导加载程序，第二段叫APP，即我们的实现业务功能应用程序。所以要在产品实现IAP功能需要我们在出厂前先烧录Bootloader程序到产品中，一般情况下是烧录在芯片的flash上。

​	Bootloader程序主要就是做三件事：

- 接收要更新的程序暂存到RAM
- 将RAM的数据写入到flash（当然也可以将APP拷贝在SRAM上运行，但是这样每次开机都要重新刷固件，而且一般MCU的SRAM都比较小，对于要实现业务功能的APP不推荐在SRAM上运行）
- 程序跳转到APP

## 芯片闪存和程序启动

### 	闪存和启动

​	要实现IAP必须要对芯片的闪存设计有一定的了解。我们以STM32F10X为例看一下芯片的闪存模块和正常程序启动流程。

![STM32F10X闪存](D:\嵌入式资料\My_Girant\嵌入式笔记\STM32F10X闪存.png)

![stm32正常程序启动流程](D:\嵌入式资料\My_Girant\嵌入式笔记\stm32正常程序启动流程.png)

​	当烧录一个程序，程序文件一般是写入flash起始地址0x08000000中的。当单片机上电以后，首先会将flash的前四个字节写入栈顶地址，也就是堆栈指针。这样做的目的是初始化堆栈指针，使其能够安全调用函数和中断服务。接下来会将0x08000004这里写入复位中断向量的地址，当单片机上电复位后，会触发复位中断，此时PC会根据中断向量表中的地址去执行中断服务函数，如上图步骤一所示；接下来在复位中断服务函数中会进行必要的时钟初始化，然后调用用户程序的main函数，此时会进入用户程序的main函数处继续执行程序。

​	我们打开项目工程的启动文件在向量表可以看到和上图是对应的和复位中断可以看到和上面的流程是对应的![image-20250920103030690](C:\Users\caojl\AppData\Roaming\Typora\typora-user-images\image-20250920103030690.png)

![image-20250920103058611](C:\Users\caojl\AppData\Roaming\Typora\typora-user-images\image-20250920103058611.png)

### 	向量表偏移

​	但是，有一个新问题摆在我们面前，那就是当我们程序跳转过后如果APP触发了中断就会去Bootloader的向量表执行中断服务函数，当执行完中断服务函数后，再回到APP会因为栈地址被破坏导致上下文不匹配，程序就直接跑飞了。

​	因此跳转到APP我们第一个要做的事情就是将中断向量表映射到另一个地址或者直接偏移到APP程序首地址。这里Cortex-M0由于厂商没有定义向量偏移寄存器VTOR，无法动态修改向量表的地址，只能使用向量表映射的方法。

```c
//下面是两种方法的实现
//向量表偏移
SCB->VTOR == 0x08008000;	//0x08008000就是APP的地址

//向量表映射
void JumpToApp_SetVectorTable(void)
{
	__disable_irq();
	memcpy((void *)RAM_START_ADDR, (void *)ApplicationAddress, VECTOR_SIZE);
	SYSCFG_MemoryRemapConfig(SYSCFG_MemoryRemap_SRAM);
	set_boot_flag(BOOT_APP);
	__enable_irq();
}
```

在KEIL工程中还需要做下面的选项配置

![image-20250920113149851](C:\Users\caojl\AppData\Roaming\Typora\typora-user-images\image-20250920113149851.png)

## Bootloader程序实现

```c
//程序跳转
//设置栈指针、设置PC寄存器为Reset_Handle
typedef void (*pFunction)(void);

void jump_code(uint32_t is_app)
{
	DBG("Jump to %s!\n",is_app?"app":"bootloader");	
	if(is_app) HAL_UART_DeInit(&huart5);
	uint32_t JumpAddress = *(__IO uint32_t*) ((is_app?APPLICATION_ADDRESS:BOOTLOADER_ADDRESS) + 4);	
	pFunction JumpToApplication = (pFunction) JumpAddress;
	 __set_MSP(*(__IO uint32_t*) (is_app?APPLICATION_ADDRESS:BOOTLOADER_ADDRESS));	
	JumpToApplication();
}
```

### 指南IAP流程

![image-20250920115226650](C:\Users\caojl\AppData\Roaming\Typora\typora-user-images\image-20250920115226650.png)