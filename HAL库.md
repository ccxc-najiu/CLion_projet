在进行STM32系列芯片的开发时，有三种开发方式

1.寄存器开发

直接配置各个寄存器，这种开发方式在处理51单片机时比较方便，因为寄存器比较少，但在处理32系列就比较麻烦，因为32系列的单片寄存器数量非常之多，难以记忆，在开发时常常要翻阅芯片的数据手册，这样就浪费了大量时间

2.库函数开发

STM32的每一款芯片都有对应的标准库，即工程文件里的stm32F1xx...之类的。在这些.c .h文件中，包含一些常用量的宏定义，把一些外设的也通过结构体变量封装起来，我们只需要配置结构体就可以修改外设的寄存器，从而实现不同功能

但问题在于不同芯片的库函数之间难以移植，且即使是使用了库函数，其开发也是十分繁琐，例如初始化一个串口

```c
USART_InitTypeDef USART_InitStructure;

USART_InitStructure.USART_BaudRate = bound;//串口波特率
USART_InitStructure.USART_WordLength = USART_WordLength_8b;//字长为8位数据格式
USART_InitStructure.USART_StopBits = USART_StopBits_1;//一个停止位
USART_InitStructure.USART_Parity = USART_Parity_No;//无奇偶校验位
USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;//无硬件数据流控制
USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx; //收发模式

USART_Init(USART3, &USART_InitStructure); //初始化串口1
```

3.HAL库

下面的评价战且作废，没有真正使用过的东西无法做出真正的评价

要做到程序通用，所需耗费的精力是庞大，有些东西看起来复杂，实际上是为了把硬件封装起来，只留下接口，在不同芯片中硬件封装不同，但接口是一致的

HAL库是ST公司目前主力推荐的开发方式，HAL（Hardware Abstraction Layer），抽象硬件层，将不同芯片的硬件抽象出来，使开发者不必过分拘泥于每款芯片本身的各种参数，外设，单纯的对硬件这一概念进行开发

和标准库一样，其目的是为了节省程序的开发周期。标准库主要是把实现功能所需要的寄存器集成在一个结构体内，HAL库则只用一些函数就做到一些特定功能的集成。同样的功能，标准库需要好几句代码，HAL库只需要一个函数

而HAL库也很好的解决了程序移植问题，对HAL库而言，只要是使用相通的外设，程序可以复制粘贴在不同的芯片上（这些芯片需要有该外设，不可无中生有）

**最方便的是，结合ST公司的STMcube软件，可以直接生成对应芯片的HAL库工程文件（这样就不用抄正点原子的工程了）**

下面介绍HAL库

按照实践论，认知的两个阶段都是在实践过程中进行的，所以要介绍HAL库，必须自己用这个库真正开发过项目。

认知两个阶段，第一阶段感性阶段，这一阶段主要认识事物的外在，事物与事物的外在练习，映射在hal库的学习上就是学习其库工程结构，各个`.c`, `.h`文件的作用。这一阶段需要构建在实践中，你需要看hal库的文档，以及hal库的一些示例来帮助你理解各个文件

认知第二阶段，论理阶段。仅仅依靠感性阶段所认识的hal库终究只是片面的，不具体的，而通过反复认知，反复实践，就能够产生认知上的飞跃，进入认知第二阶段，论理阶段。此处，我们要将hal库的结构抽象开来，分析为什么要这么设计，其优点是什么，缺点是什么，各个层次的作用是什么，彻底将hal库转化为概念上的东西，进行思维推导。

# 认知第一阶段

首先来看其外部结构，hal库本身是一个函数库，下面以stm32CubeMX生成的F1系列芯片的工程为例讲解，工程文件目录如下。

```
│  .cproject
│  .mxproject
│  .project
│  CLion_projet.ioc
│  CMakeLists.txt
│  CMakeLists_template.txt
│  HAL库.md
│  STM32F103CBTX_FLASH.ld
│
├─.idea
│  │  .gitignore
│  │  misc.xml
│  │  modules.xml
│  │  workspace.xml
│  │
│  └─runConfigurations
│          OCD_CLion_projet.xml
│
├─cmake-build-debug
│  │  .ninja_deps
│  │  .ninja_log
│  │  build.ninja
│  │  CLion_projet.bin
│  │  CLion_projet.elf
│  │  CLion_projet.hex
│  │  CLion_projet.map
│  │  CMakeCache.txt
│  │  cmake_install.cmake
│  │
│  ├─.cmake
│  │  └─api
│  │      └─v1
│  │          ├─query
│  │          │      cache-v2
│  │          │      cmakeFiles-v1
│  │          │      codemodel-v2
│  │          │      toolchains-v1
│  │          │
│  │          └─reply
│  │                  cache-v2-947b6310a35a7d40d65d.json
│  │                  cmakeFiles-v1-d56d74da120a817eff3b.json
│  │                  codemodel-v2-443b3292fcd0bc95fcf8.json
│  │                  directory-.-Debug-d0094a50bb2071803777.json
│  │                  index-2022-08-13T02-04-58-0325.json
│  │                  target-CLion_projet.elf-Debug-05cbcc68b29d06060ed6.json
│  │                  toolchains-v1-4e859f9b1526a9561593.json
│  │
│  ├─CMakeFiles
│  │  │  clion-environment.txt
│  │  │  clion-log.txt
│  │  │  cmake.check_cache
│  │  │  CMakeError.log
│  │  │  CMakeOutput.log
│  │  │  rules.ninja
│  │  │  TargetDirectories.txt
│  │  │
│  │  ├─3.21.1
│  │  │  │  CMakeASMCompiler.cmake
│  │  │  │  CMakeCCompiler.cmake
│  │  │  │  CMakeCXXCompiler.cmake
│  │  │  │  CMakeDetermineCompilerABI_C.bin
│  │  │  │  CMakeDetermineCompilerABI_CXX.bin
│  │  │  │  CMakeSystem.cmake
│  │  │  │
│  │  │  ├─CompilerIdASM
│  │  │  ├─CompilerIdC
│  │  │  │  │  CMakeCCompilerId.c
│  │  │  │  │  CMakeCCompilerId.o
│  │  │  │  │
│  │  │  │  └─tmp
│  │  │  └─CompilerIdCXX
│  │  │      │  CMakeCXXCompilerId.cpp
│  │  │      │  CMakeCXXCompilerId.o
│  │  │      │
│  │  │      └─tmp
│  │  ├─CLion_projet.elf.dir
│  │  │  ├─Core
│  │  │  │  ├─Src
│  │  │  │  │      main.c.obj
│  │  │  │  │      stm32f1xx_hal_msp.c.obj
│  │  │  │  │      stm32f1xx_it.c.obj
│  │  │  │  │      syscalls.c.obj
│  │  │  │  │      sysmem.c.obj
│  │  │  │  │      system_stm32f1xx.c.obj
│  │  │  │  │
│  │  │  │  └─Startup
│  │  │  │          startup_stm32f103cbtx.s.obj
│  │  │  │
│  │  │  └─Drivers
│  │  │      └─STM32F1xx_HAL_Driver
│  │  │          └─Src
│  │  │                  stm32f1xx_hal.c.obj
│  │  │                  stm32f1xx_hal_cortex.c.obj
│  │  │                  stm32f1xx_hal_dma.c.obj
│  │  │                  stm32f1xx_hal_exti.c.obj
│  │  │                  stm32f1xx_hal_flash.c.obj
│  │  │                  stm32f1xx_hal_flash_ex.c.obj
│  │  │                  stm32f1xx_hal_gpio.c.obj
│  │  │                  stm32f1xx_hal_gpio_ex.c.obj
│  │  │                  stm32f1xx_hal_pwr.c.obj
│  │  │                  stm32f1xx_hal_rcc.c.obj
│  │  │                  stm32f1xx_hal_rcc_ex.c.obj
│  │  │                  stm32f1xx_hal_tim.c.obj
│  │  │                  stm32f1xx_hal_tim_ex.c.obj
│  │  │
│  │  └─CMakeTmp
│  └─Testing
│      └─Temporary
│              LastTest.log
│
├─Core
│  ├─Inc
│  │      main.h
│  │      stm32f1xx_hal_conf.h
│  │      stm32f1xx_it.h
│  │
│  ├─Src
│  │      main.c
│  │      stm32f1xx_hal_msp.c
│  │      stm32f1xx_it.c
│  │      syscalls.c
│  │      sysmem.c
│  │      system_stm32f1xx.c
│  │
│  └─Startup
│          startup_stm32f103cbtx.s
│
└─Drivers
    ├─CMSIS
    │  │  LICENSE.txt
    │  │
    │  ├─Device
    │  │  └─ST
    │  │      └─STM32F1xx
    │  │          │  License.md
    │  │          │
    │  │          ├─Include
    │  │          │      stm32f103xb.h
    │  │          │      stm32f1xx.h
    │  │          │      system_stm32f1xx.h
    │  │          │
    │  │          └─Source
    │  │              └─Templates
    │  └─Include
    │          cmsis_armcc.h
    │          cmsis_armclang.h
    │          cmsis_compiler.h
    │          cmsis_gcc.h
    │          cmsis_iccarm.h
    │          cmsis_version.h
    │          core_armv8mbl.h
    │          core_armv8mml.h
    │          core_cm0.h
    │          core_cm0plus.h
    │          core_cm1.h
    │          core_cm23.h
    │          core_cm3.h
    │          core_cm33.h
    │          core_cm4.h
    │          core_cm7.h
    │          core_sc000.h
    │          core_sc300.h
    │          mpu_armv7.h
    │          mpu_armv8.h
    │          tz_context.h
    │
    └─STM32F1xx_HAL_Driver
        │  License.md
        │
        ├─Inc
        │  │  stm32f1xx_hal.h
        │  │  stm32f1xx_hal_cortex.h
        │  │  stm32f1xx_hal_def.h
        │  │  stm32f1xx_hal_dma.h
        │  │  stm32f1xx_hal_dma_ex.h
        │  │  stm32f1xx_hal_exti.h
        │  │  stm32f1xx_hal_flash.h
        │  │  stm32f1xx_hal_flash_ex.h
        │  │  stm32f1xx_hal_gpio.h
        │  │  stm32f1xx_hal_gpio_ex.h
        │  │  stm32f1xx_hal_pwr.h
        │  │  stm32f1xx_hal_rcc.h
        │  │  stm32f1xx_hal_rcc_ex.h
        │  │  stm32f1xx_hal_tim.h
        │  │  stm32f1xx_hal_tim_ex.h
        │  │  stm32f1xx_ll_bus.h
        │  │  stm32f1xx_ll_cortex.h
        │  │  stm32f1xx_ll_dma.h
        │  │  stm32f1xx_ll_exti.h
        │  │  stm32f1xx_ll_gpio.h
        │  │  stm32f1xx_ll_pwr.h
        │  │  stm32f1xx_ll_rcc.h
        │  │  stm32f1xx_ll_system.h
        │  │  stm32f1xx_ll_utils.h
        │  │
        │  └─Legacy
        │          stm32_hal_legacy.h
        │
        └─Src
                stm32f1xx_hal.c
                stm32f1xx_hal_cortex.c
                stm32f1xx_hal_dma.c
                stm32f1xx_hal_exti.c
                stm32f1xx_hal_flash.c
                stm32f1xx_hal_flash_ex.c
                stm32f1xx_hal_gpio.c
                stm32f1xx_hal_gpio_ex.c
                stm32f1xx_hal_pwr.c
                stm32f1xx_hal_rcc.c
                stm32f1xx_hal_rcc_ex.c
                stm32f1xx_hal_tim.c
                stm32f1xx_hal_tim_ex.c
```

我们主要看Core目录与Drivers目录，其余部分为自动生成，不需要再手动更改

其中Drivers目录下CMSIS也是一个硬件抽象层

> 因为基于Cortex系列芯片采用的内核都是相同的，区别主要为核外的片上外设的差异，这些差异却导致软件在同内核，不同外设的芯片上移植困难。为了解决不同的芯片厂商生产的Cortex微控制器软件 的兼容性问题，ARM与芯片厂商建立了CMSIS标准(Cortex MicroController Software Interface Standard)。
> 
> CMSIS标准中最主要的为CMSIS核心层，它包括了：
> 
> q 内核函数层：其中包含用于访问内核寄存器的名称、地址定义，主要由ARM公司提供。
> 
> q 设备外设访问层：提供了片上的核外外设的地址和中断定义，主要由芯片生产商提供。
> 
> 可见CMSIS层位于硬件层与操作系统或用户层之间，提供了与芯片生产商无关的硬件抽象层，可以为接口外设、实时操作系统提供简单的处理器软件接口，屏蔽了硬件差异，这对软件的移植是有极大的好处的。STM32的库，就是按照CMSIS标准建立的。


而我们所使用的HAL库就是基于CMSIS标准开发的。这里只介绍HAL库，不介绍CMSIS。Drivers下`STM32F1xx_HAL_Driver`就是F1系列的HAL库，显然，还有其它系类的HAL库，不过基本结构是一致的。`STM32F1xx_HAL_Driver`下有Src目录与Inc目录，在其中分别存放`.c`文件与`.h`文件，这一处主要各种外设的驱动文件与外设驱动的拓展（`stm32f1xx_hal_ppp/_ex.c/h`），此外还有用于初始化hal固件库的源文件（`stm32f1xx_hal.c/h`），HAL固件库宏定义文件（`stm32f1xx_hal_def.h`）

在Core目录下是需要自己编写的文件，其中也有Src与Inc两个目录，分别用于存放`.c`与`.h`文件。此处包含了hal库中的模板配置文件，中断配置文件(`stm32f1xx_hal_it.c/.h`)

其中，主要有以下几类文件

<div align="center">

| 源文件 | 说明|
| ---- | ---- |
| stm32f1xx_hal_ppp.c/h | 主要外设/模块的`.c`与`.h`驱动程序源文件，其中包含对应外设/驱动通用的API。该文件中ppp是代指，如`stm32f1xx_hal_adc.c/h`就是adc外设的驱动源文件 | 
| stm32f1xx_hal_ppp_ex.c/h | 外设/模块驱动程序拓展的`.c`与`.h`源文件，例如`stm32f1xx_hal_adc_ex.c/h`，其通常用于定义某个指定型号独有的API，重点在于独有 | 
| stm32f1xx_hal.c/h | 用于初始化HAL固件库的源文件，包含有`DBGMCU`, `Remap`和基于`SysTick`的时间延迟函数 |
| stm32f1xx_hal_msp_template.c| 该文件需要复制到自身的工程目录中，其是一个模板文件，包含了外设的主堆栈指针 (MAP, MainStackPointer)的初始化和反向初始化|
| stm32f1xx_hal_conf_template.c| 和上一个文件类似，也是模板文件，用于配置指定应用驱动 |
| stm32f4xx_hal_def.h | 通用的HAL固件库资源，例如通常的语句、枚举、结构体、宏等定义 |
</div>

以上文件虽然多，但并非每一个文件都要用到，我们需要理清各个文件相对于hal库的关系与作用。首先介绍组成hal库的最小文件集合

<div align="center">


| 源文件 | 说明 |
| ---- | ---- |
| **system_stm32f1xx.c** | 包含有系统启动时调用的`SystemInit()`方法，该方法允许重新定位内部的SRAM中的向量表，并且配置FSMC/FMC（如果可用）使用配置的SRAM或SDRAM作为数据存储器 |
| startup_stm32f1xx.s | 包含有`重置处理器`与异常向量在内的，工具链指定文件；它的后缀是.s，说明它是一个汇编文件 |
| stm32f1xx_flash.icf | (可选)EWARM工具链的链接器文件，允许调整堆/栈大小以适应程序的需求 |
| stm32f1xx_hal_msp.c | 包含用户应用程序当中使用到的外设**主堆栈指针**的MSP的`初始化`与`反向初始化`（主程序与回调函数），它的编写需要参考`stm32f1xx_hal_msp_template.c` |
| stm32f1xx_hal_conf.h | 该文件允许用户为特定的应用程序编写HAL驱动程序，它的编写需要参考`stm32f1xx_hal_conf_template.c` |
| stm32f1xx_it.c/h | 包含异常处理与外设中断服务程序 |
| main.c/h | 用户主程序，除了放置用户程序代码之外，还会调用`HAL_Init()`、实现`assert_failed()`、配置系统时钟，初始化指定外设
</div>

综上所述，要构建一个HAL库最小系统，需要以下文件
1.系统启动相关
- system_stm32f1xx.c
- startup_stm32f1xx.s
- stm32f1xx_flash.icf
2.用户自定义堆栈与驱动
- stm32f1xx_hal_msp.c
- stm32f1xx_hal_conf.h
3.中断处理
- stm32f1xx_it.c
4.主程序调用HAL_Init。实现`assert_failed()`、配置系统时钟，初始化指定外设
- main.c



## 认知第一阶段的实践

### HAL数据类型

与标准库类似，HAL库使用结构体来进行相关外设的配置

每一个 HAL 固件驱动程序都会包含有如下三种类型的数据结构：

- **外设操作结构体：** PPP_HandleTypeDef 是HAL驱动程序中主要的实现结构，其用于配置外设，注册和嵌入式外设相关的结构体与变量，例如usart这一外设的`USART_HandleTypeDef`结构体
```c
typedef struct {
    USART_TypeDef *Instance;               /* USART 寄存器基地址 */
    USART_InitTypeDef Init;                /* USART 通信参数 */
    uint8_t *pTxBuffPtr;                   /* 指向 USART Tx 发送缓冲区的指针 */
    uint16_t TxXferSize;                   /* USART 发送大小 */
    __IO uint16_t TxXferCount;             /* USART 发送计数器 */
    uint8_t *pRxBuffPtr;                   /* 指向 USART Rx 传输缓冲区的指针 */
    uint16_t RxXferSize;                   /* USART Rx 传输大小 */
    __IO uint16_t RxXferCount;             /* USART Rx 传输计数器 */
    DMA_HandleTypeDef *hdmatx;             /* USART Tx 的 DMA 处理参数 */
    DMA_HandleTypeDef *hdmarx;             /* USART Rx 的 DMA 处理参数 */
    HAL_LockTypeDef Lock;                  /* 对象锁定 */
    __IO HAL_USART_StateTypeDef State;     /* USART 通信状态 */
    __IO HAL_USART_ErrorTypeDef ErrorCode; /* USART 错误代码 */
} USART_HandleTypeDef;
```
可以看到，上诉代码能够配置USART通信参数，缓冲区参数，DMA相关参数，基本涵盖与USART相关的操作

- **初始化与配置结构体：** PPP_InitTypeDef结构体定义在通用固件驱动程序的`.h`文件中，其用于初始化外设相关参数，还是以USART为例

```c
typedef struct {
    uint32_t BaudRate;     /* 配置 UART 通信波特率 */
    uint32_t WordLength;   /* 指定接收或者发送数据的长度 */
    uint32_t StopBits;     /* 指定传输的停止位数 */
    uint32_t Parity;       /* 指定校验模式 */
    uint32_t Mode;         /* 启用或者禁用收发模式 */
    uint32_t HwFlowCtl;    /* 启用或者禁用硬件流控制模式 */
    uint32_t OverSampling; /* 启用或者禁用过采样，以达到更高的速度（可以达到 fPCLK/8）*/
} UART_InitTypeDef;

```

可以看到，上诉代码初始化串口通信的波特率，数据长度，停止位数，校验模式等参数

除此之外，配置结构体 HAL_PPP_Config 用于初始化子模块或者子实例，例如下面 ADC 模数转换器外设示例：

`HAL_ADC_ConfigChannel (ADC_HandleTypeDef* hadc, ADC_ChannelConfTypeDef* sConfig)`

- 指定流程结构体


指定流程结构体：HAL_PPP_Process 用于通用 API 当中的特定流程，通常被定义在通用固件驱动程序的 .h 头文件当中；
HAL_PPP_Process (PPP_HandleTypeDef* hadc,PPP_ProcessConfig* sConfig)
HAL 库 API 的分类

### HAL库API的分类

HAL固件库的API可以被划分为**通用**与**扩展**两种类型

- 通用API：适用于所有STM32微控制器，主要出现在HAL固件库的**通用**驱动程序文件当中

以ADC外设的通用API为例

```c
HAL_StatusTypeDef HAL_ADC_Init(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_DeInit(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_Start(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_Stop(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_Start_IT(ADC_HandleTypeDef* hadc);
HAL_StatusTypeDef HAL_ADC_Stop_IT(ADC_HandleTypeDef* hadc);
void HAL_ADC_IRQHandler(ADC_HandleTypeDef* hadc)
```

在上述代码中，出现多个API，对应着ADC外设的通用操作，其输入参数为外设操作结构体变量，是之前HAL固件驱动程序所讲述的三种结构体类型之一

- 扩展API：可以进一步划分为**指定系列**与**指定型号**两种类型，处于HAL固件库的**扩展**驱动程序源文件（`stm32f1xx_hal_ppp_ex.c/h`）当中

一般而言只需要通用API即可，目前还未找到 需要拓展API的例子，若是之后找到例子可以在这里再做补充

### HAL库驱动规范

#### HAL API命名规则

HAL固件库当中所使用的驱动程序命名规则如下所示

<div align="center">

|      | 通用 | 系列指定 | 具体型号指定 |
| --- | --- | --- | --- |
| 模块驱动文件名称 | `stm32f1xx_hal_ppp (c/h)` | stm32f1xx_hal_ppp_ex (c/h) | stm32f1xx_hal_ppp_ex (c/h) |
| 函数名称 | HAL_PPP_MODULE | HAL_PPP_MODULE | HAL_PPP_MODULE |
| 头文件名称 | HAL_PPP_Function / HAL_PPP_FeatureFunction_MODE | HAL_PPPEx__Function / HAL_PPPEx_FeatureFunction_MODE | HAL_PPPEx_Function / HAL_PPPEx_FeatureFunction_MODE |
| 指针名称 | PPP_HandleTypeDef | NA | NA |
| 初始化结构体名称 | PPP_InitTypeDef | NA | PPP_InitTypeDef |
| 枚举名称 | HAL_PPP_StructnameTypeDef | NA | NA |

</div>

对于上述表格所描述的驱动程序命名规则，需要特别注意以下几个事项

- PPP前缀指代的是外设的功能模式，而非外设本身，因此，使用USART串口时，该前缀可以是`USART`、`IRDA`、`UART`、`SMARTCARD`; 这很容易理解，因为无论是外设初始化还是外设操作所对应的操作对象都不是物理意义上的外设，而是在逻辑意义上外设的某一功能模式
- 一个源文件当中定义的常量，就定义在该源文件内部，而多个源文件所共用的常量则定义在头文件当中; 除外设驱动的函数参数以外，所有常量都需要大写
- `typedef`类型的变量名称都应当以`_TypeDef`作为后缀
- HAL固件库认为**寄存器**属于常量, 大多数情况下常量名称是大写的，并且使用于官方参考手册当中相同的首字母缩写
- 外设寄存器被声明在`stm32f1xx_hal_PPP.h`头文件的`PPP_TypeDef`，例如`ADC_TypeDef`
- 外设函数的名称以`HAL_`作为前缀，然后是相应外设的首字母缩写（大写），然后再跟上一条下划线，接下来每一个首字母都大写. e.g. `HAL_UART_Transmit()`
- 包含指定PPP外设初始化参数的结构体名称被命名为`PPP_InitTypeDef`, e.g `ADC_InitTypeDef`
- 包含指定PPP外设配置参数的结构体被命名为`PPP_xxxxConfTypeDef`, 例如`ADC_ChannelConfTypeDef`
- 外设指针结构体被命名为`PPP_HandleTypeDef`，例如`DMA_HandleTypeDef`
- 根据`PPP_InitTypeDef`当中的参数，用于初始化PPP外设的函数被命名为`HAL_PPP_Init`，例如`HAL_TIM_Init()`
- 采用默认值重置**PPP**外设寄存器的函数被命名为`HAL_PPP_DeInit`, 例如`HAL_TIM_DeInit()`
- 后缀`MODE`是指处理模式（轮询、中断、DMA），例如在本地资源以外使用DMA时，就应当调用`HAL_PPP_Funtion_DMA()`函数
- 前缀`Feature`是指新的特性，例如`HAL_ADCEx_InjectedStart()`表示ADC开始注入通道

#### HAL通用命名规则

对于共有的系统外设，无需使用指针或者实例对象，这个规则适用于GPIO、SYSTICK、NVIC、RCC、FLASH等外设，例如函数`HAL_GPIO_Init()`只需要GPIO的地址及其配置参数

```c
HAL_StatusTypeDef HAL_GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* Init) {
}

```

这很容易理解，如果不是通用外设，就需要做一些配置，无法直接定义GPIOx

每个外设驱动程序当中都定义有处理中断和特定时钟配置的宏，这些宏会被导出到外设驱动的**头文件**，以便于拓展文件使用，这些用于处理中断和特定时钟配置的宏如下所示

<div align="center">

| 宏定义 | 功能描述 |
| --- | --- |
| __HAL_PPP_ENABLE_IT(__HANDLE__, __INTERRUPT__) | 使能一个特定的外设中断 |



</div>

### SystemClock_Config

系统时钟配置

单纯的看代码并不能搞懂，得结合时钟树来看

![20220822183536](https://cdn.jsdelivr.net/gh/ccxc-najiu/imageBed@main/note/images20220822183536.png)

其中用到了HSE，故有代码

```c
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
```
然后未使用PLL分频器输出作为sysClk

故不开启PLL
```c
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;
```

以上就是RCC_OscInitStruct的内容，其用于初始化SysCLK之前的时钟配置选项

接下来是RCC_ClkInitStruct的配置，其中最重要的语句是

```c
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSE;
```
其表示SysCLK的时钟源是HSE （看时钟树，另外两个选择是HSI与PLL输出）

然后SysCLK再经过分频得到AHB, APB1, APB2, 下面的代码配置其分频倍率

```c
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
```
总体代码如下
```c
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
  RCC_OscInitStruct.HSEState = RCC_HSE_ON;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSE;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV2;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0) != HAL_OK)
  {
    Error_Handler();
  }
}
```