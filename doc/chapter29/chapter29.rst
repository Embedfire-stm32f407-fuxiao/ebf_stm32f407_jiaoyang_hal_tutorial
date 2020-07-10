ADC—电压采集
------------

本章参考资料：《STM32F4xx中文参考手册》ADC章节。

学习本章时，配合《STM32F4xx中文参考手册》ADC章节一起阅读，效果会更佳，特别是涉及到寄存器说明的部分。

ADC简介
~~~~~~~

STM32F407IGT6有3个ADC，每个ADC有12位、10位、8位和6位可选，每个ADC有16个外部通道。
另外还有两个内部ADC源和VBAT通道挂在ADC1上。ADC具有独立模式、双重模式和三重模式，对于不同AD转换要求几乎都有合适的模式可选。
ADC功能非常强大，具体的我们在功能框图中分析每个部分的功能。

ADC功能框图剖析
~~~~~~~~~~~~~~~

.. image:: media/image2.png
   :align: center
   :alt: 图 29‑1 单个ADC功能框图
   :name: 图29_1

图 29‑1 单个ADC功能框图

掌握了ADC的功能框图，就可以对ADC有一个整体的把握，在编程的时候可以做到了然如胸，不会一知半解。框图讲解采用从左到右的方式，跟ADC采集数据，转换数据，传输数据的方向大概一致。

①电压输入范围
'''''''''''''

ADC输入范围为：V\ :sub:`REF-` ≤ V\ :sub:`IN` ≤
V\ :sub:`REF+`\ 。由V\ :sub:`REF-`\ 、V\ :sub:`REF+` 、V\ :sub:`DDA`
、V\ :sub:`SSA`\ 、这四个外部引脚决定。

我们在设计原理图的时候一般把V\ :sub:`SSA`\ 和V\ :sub:`REF-`\ 接地，
把V\ :sub:`REF+`\ 和V\ :sub:`DDA`
接3V3，得到ADC的输入电压范围为：0~3.3V。

如果我们想让输入的电压范围变宽，去到可以测试负电压或者更高的正电压，我们可以在外部加一个电压调理电路，把需要转换的电压抬升或者降压到0~3.3V，这样ADC就可以测量了。

②输入通道
'''''''''

我们确定好ADC输入电压之后，那么电压怎么输入到ADC？这里我们引入通道的概念，STM32的ADC多达19个通道，
其中外部的16个通道就是框图中的ADCx_IN0、ADCx_IN1...ADCx_IN5。这16个通道对应着不同的IO口，
具体是哪一个IO口可以从手册查询到。其中ADC1/2/3还有内部通道： ADC1的通道ADC1_IN16连接到内部的VSS，
通道ADC1_IN17连接到了内部参考电压VREFINT 连接，通道ADC1_IN18连接到了芯片内部的温度传感器或者备用电源VBAT。
ADC2和ADC3的通道16、17、18全部连接到了内部的VSS。

.. image:: media/image3.png
   :align: center
   :alt: 图 29‑2 STM32F407IGT6 ADC 通道
   :name: 图29_2

图 29‑2 STM32F407IGT6 ADC 通道

外部的16个通道在转换的时候又分为规则通道和注入通道，其中规则通道最多有16路，注入通道最多有4路。那这两个通道有什么区别？在什么时候使用？

规则通道
=============

规则通道：顾名思意，规则通道就是很规矩的意思，我们平时一般使用的就是这个通道，或者应该说我们用到的都是这个通道，没有什么特别要注意的可讲。

注入通道
===========

注入，可以理解为插入，插队的意思，是一种不安分的通道。它是一种在规则通道转换的时候强行插入要转换的一种。如果在规则通道转换过程中，有注入通道插队，那么就要先转换完注入通道，等注入通道转换完成后，再回到规则通道的转换流程。这点跟中断程序很像，都是不安分的主。所以，注入通道只有在规则通道存在时才会出现。

③转换顺序
'''''''''

规则序列
===========

规则序列寄存器有3个，分别为SQR3、SQR2、SQR1。SQR3控制着规则序列中的第一个到第六个转换，对应的位为：SQ1[4:0]~SQ6[4:0]，第一次转换的是位4:0
SQ1[4:0]，如果通道16想第一次转换，那么在SQ1[4:0]写16即可。SQR2控制着规则序列中的第7到第12个转换，对应的位为：SQ7[4:0]~SQ12[4:0]，如果通道1想第8个转换，则SQ8[4:0]写1即可。SQR1控制着规则序列中的第13到第16个转换，对应位为：SQ13[4:0]~SQ16[4:0]，如果通道6想第10个转换，则SQ10[4:0]写6即可。具体使用多少个通道，由SQR1的位L[3:0]决定，最多16个通道。

.. image:: media/image4.png
   :align: center
   :alt: 图 29‑3 规则序列寄存器
   :name: 图29_3

图 29‑3 规则序列寄存器

注入序列
===========

注入序列寄存器JSQR只有一个，最多支持4个通道，具体多少个由JSQR的JL[2:0]决定。如果JL的
值小于4的话，则JSQR跟SQR决定转换顺序的设置不一样，第一次转换的不是JSQR1[4:0]，而是JCQRx[4:0]
，x =
（4-JL），跟SQR刚好相反。如果JL=00（1个转换），那么转换的顺序是从JSQR4[4:0]开始，而不是从JSQR1[4:0]开始，这个要注意，编程的时候不要搞错。当JL等于4时，跟SQR一样。

.. image:: media/image5.png
   :align: center
   :alt: 图 29‑4 注入序列寄存器
   :name: 图29_4

图 29‑4 注入序列寄存器

④触发源
'''''''

通道选好了，转换的顺序也设置好了，那接下来就该开始转换了。ADC转换可以由ADC控制寄存器2: ADC_CR2的ADON这个位来控制，
写1的时候开始转换，写0的时候停止转换，这个是最简单也是最好理解的开启ADC转换的控制方式，理解起来没啥技术含量。

除了这种庶民式的控制方法，ADC还支持外部事件触发转换，这个触发包括内部定时器触发和外部IO触发。触发源有很多，具体选择哪一种触发源，由ADC控制寄存器2:ADC_CR2的EXTSEL[2:0]和JEXTSEL[2:0]位来控制。EXTSEL[2:0]用于选择规则通道的触发源，JEXTSEL[2:0]用于选择注入通道的触发源。选定好触发源之后，触发源是否要激活，则由ADC控制寄存器2:ADC_CR2的EXTTRIG和JEXTTRIG这两位来激活。

如果使能了外部触发事件，我们还可以通过设置ADC控制寄存器2:ADC_CR2的EXTEN[1:0]和JEXTEN[1:0]来控制触发极性，可以有4种状态，分别是：禁止触发检测、上升沿检测、下降沿检测以及上升沿和下降沿均检测。

⑤转换时间
'''''''''

ADC时钟
=========

ADC输入时钟ADC_CLK由PCLK2经过分频产生，最大值是36MHz，典型值为30MHz，分频因子由ADC通用控制寄存器ADC_CCR的ADCPRE[1:0]设置，
可设置的分频系数有2、4、6和8，注意这里没有1分频。对于STM32F407IGT6我们一般设置PCLK2=HCLK/2=84MHz。
所以程序一般使用4分频或者6分频。

采样时间
==========

ADC需要若干个ADC_CLK周期完成对输入的电压进行采样，采样的周期数可通过ADC
采样时间寄存器ADC_SMPR1和ADC_SMPR2中的SMP[2:0]位设置，ADC_SMPR2控制的是通道0~9，ADC_SMPR1控制的是通道10~17。每个通道可以分别用不同的时间采样。其中采样周期最小是3个，即如果我们要达到最快的采样，那么应该设置采样周期为3个周期，这里说的周期就是1/ADC_CLK。

ADC的总转换时间跟ADC的输入时钟和采样时间有关，公式为：

Tconv = 采样时间 + 12个周期

当ADCCLK =
30MHz，即PCLK2为60MHz，ADC时钟为2分频，采样时间设置为3个周期，那么总的转换时为：Tconv
= 3 + 12 = 15个周期 =0.5us。

一般我们设置PCLK2=84MHz，经过ADC预分频器能分频到最大的时钟只能是21M，采样周期设置为3个周期，算出最短的转换时间为0.7142us，这个才是最常用的。

⑥数据寄存器
'''''''''''

一切准备就绪后，ADC转换后的数据根据转换组的不同，规则组的数据放在ADC_DR寄存器，注入组的数据放在JDRx。
如果是使用双重或者三重模式那规矩组的数据是存放在通用规矩寄存器ADC_CDR内的。

规则数据寄存器ADC_DR
======================

ADC规则组数据寄存器ADC_DR只有一个，是一个32位的寄存器，只有低16位有效并且只是用于独立模式存放转换完成数据。因为ADC的最大精度是12位，ADC_DR是16位有效，这样允许ADC存放数据时候选择左对齐或者右对齐，具体是以哪一种方式存放，由ADC_CR2的11位ALIGN设置。假如设置ADC精度为12位，如果设置数据为左对齐，那AD转换完成数据存放在ADC_DR寄存器的[4:15]位内；如果为右对齐，则存放在ADC_DR寄存器的[0:11]位内。

规则通道可以有16个这么多，可规则数据寄存器只有一个，如果使用多通道转换，那转换的数据就全部都挤在了DR里面，前一个时间点转换的通道数据，就会被下一个时间点的另外一个通道转换的数据覆盖掉，所以当通道转换完成后就应该把数据取走，或者开启DMA模式，把数据传输到内存里面，不然就会造成数据的覆盖。最常用的做法就是开启DMA传输。

如果没有使用DMA传输，我们一般都需要使用ADC状态寄存器ADC_SR获取当前ADC转换的进度状态，进而进行程序控制。

注入数据寄存器ADC_JDRx
======================

ADC注入组最多有4个通道，刚好注入数据寄存器也有4个，每个通道对应着自己的寄存器，不会跟规则寄存器那样产生数据覆盖的问题。ADC_JDRx是32位的，低16位有效，高16位保留，数据同样分为左对齐和右对齐，具体是以哪一种方式存放，由ADC_CR2的11位ALIGN设置。

通用规则数据寄存器ADC_CDR
=========================

规则数据寄存器ADC_DR是仅适用于独立模式的，而通用规则数据寄存器ADC_CDR是适用于双重和三重模式的。独立模式就是仅仅适用三个ADC的其中一个，双重模式就是同时使用ADC1和ADC2，而三重模式就是三个ADC同时使用。在双重或者三重模式下一般需要配合DMA数据传输使用。

⑦中断
'''''

转换结束中断
======================

数据转换结束后，可以产生中断，中断分为四种：规则通道转换结束中断，注入转换通道转换结束中断，模拟看门狗中断和溢出中断。其中转换结束中断很好理解，跟我们平时接触的中断一样，有相应的中断标志位和中断使能位，我们还可以根据中断类型写相应配套的中断服务程序。

模拟看门狗中断
===================

当被ADC转换的模拟电压低于低阈值或者高于高阈值时，就会产生中断，前提是我们开启了模拟看门狗中断，其中低阈值和高阈值由ADC_LTR和ADC_HTR设置。例如我们设置高阈值是2.5V，那么模拟电压超过2.5V的时候，就会产生模拟看门狗中断，反之低阈值也一样。

溢出中断
===================

如果发生DMA传输数据丢失，会置位ADC状态寄存器ADC_SR的OVR位，如果同时使能了溢出中断，那在转换结束后会产生一个溢出中断。

DMA请求
===================

规则和注入通道转换结束后，除了产生中断外，还可以产生DMA请求，把转换好的数据直接存储在内存里面。要注意的是只有ADC1和ADC3可以产生DMA请求。有关DMA请求需要配合《STM32F10X-中文参考手册》DMA控制器这一章节来学习。一般我们在使用ADC的时候都会开启DMA传输。

⑧电压转换
'''''''''

模拟电压经过ADC转换后，是一个12位的数字值，如果通过串口以16进制打印出来的话，可读性比较差，那么有时候我们就需要把数字电压转换成模拟电压，也可以跟实际的模拟电压（用万用表测）对比，看看转换是否准确。

我们一般在设计原理图的时候会把ADC的输入电压范围设定在：0~3.3v，因为ADC是12位的，那么12位满量程对应的就是3.3V，12位满量程对应的数字值是：2^12。数值0对应的就是0V。如果转换后的数值为  X
，X对应的模拟电压为Y，那么会有这么一个等式成立：  2^12 / 3.3 = X / Y，=>
Y = (3.3 \* X ) / 2^12。

ADC初始化结构体详解
~~~~~~~~~~~~~~~~~~~

HAL库函数对每个外设都建立了一个初始化结构体xxx _HandleTypeDef (xxx为外设名称)，
结构体成员用于设置外设工作参数，并由HAL库函数HAL_xxx_Init()调用这些设定参数进入设置外设相应的寄存器，达到配置外设工作环境的目的。

结构体xxx__HandleTypeDef和库函数HAL_xxx_Init配合使用是HAL库精髓所在，
理解了结构体xxx_HandleTypeDef每个成员意义基本上就可以对该外设运用自如了。
结构体xxx_HandleTypeDef定义在stm32f4xx_hal_xxx.h文件中，库函数HAL_xxx_Init定义在stm32f4xx_hal_xxx.c文件中，
编程时我们可以结合这两个文件内注释使用。

ADC_HandleTypeDef结构体
''''''''''''''''''''''''

ADC_HandleTypeDef结构体定义在stm32f4xx_adc.h文件内，具体定义如下：

.. code-block:: c

   typedef struct {
      ADC_TypeDef          *Instance; /*寄存器基地址指针*/
      ADC_InitTypeDef      Init; /*ADC初始化参数结构体*/
      __IO uint32_t        NbrOfCurrentConversionRank;//正在转换序列的ADC数目
      DMA_HandleTypeDef    *DMA_Handle; /* DMA处理程序指针 */
      HAL_LockTypeDef      Lock; /*ADC锁定对象 */
      __IO uint32_t        State; /*ADC通信状态*/
      __IO uint32_t        ErrorCode; /*ADC错误码 */
   } ADC_HandleTypeDef;

\*Instance：ADC寄存器基地址指针，所有参数都是指定基地址后才能正确写入寄存器。

Init：ADC初始化结构体，下面会详细讲解每一个成员。

\*DMA_Handle：DMA处理程序指针。

Lock：ADC锁定对象。

State：ADC转换状态。

ErrorCode：ADC错误码。

ADC_InitTypeDef结构体
''''''''''''''''''''''''

ADC_InitTypeDef初始化结构体被ADC_HandleTypeDef结构体引用。

ADC_InitTypeDef结构体定义在stm32f4xx_hal_adc.h文件内，具体定义如下：

.. code-block:: c

   typedef struct {
      uint32_t ClockPrescaler;        /*ADC时钟分频系数 */
      uint32_t Resolution;            /*ADC分辨率选择 */
      uint32_t DataAlign;             /*输出数据对齐方式 */
      uint32_t ScanConvMode;          /*扫描转换模式 */
      uint32_t EOCSelection;          /*转换结束标志使用轮询或者中断*/
      uint32_t ContinuousConvMode;    /*连续转换模式 */
      uint32_t NbrOfConversion;       /*规格转换序列数目 */
      uint32_t DiscontinuousConvMode; /*不连续采样模式 */
      uint32_t NbrOfDiscConversion;   /*不连续采样通道 */
      uint32_t ExternalTrigConv;      /*外部事件触发选择 */
      uint32_t ExternalTrigConvEdge;  /*外部事件触发极性 */
      uint32_t DMAContinuousRequests; /*DMA连续请求转换 */
   } ADC_InitTypeDef;

ADC_Prescaler：ADC时钟分频系数选择，ADC时钟是有PCLK2分频而来，分频系数决定ADC时钟频率，可选的分频系数为2、4、6和8。ADC最大时钟配置为36MHz。

ADC_Resolution：配置ADC的分辨率，可选的分辨率有12位、10位、8位和6位。分辨率越高，AD转换数据精度越高，转换时间也越长；分辨率越低，AD转换数据精度越低，转换时间也越短。

ADC_DataAlign：转换结果数据对齐模式，可选右对齐ADC_DataAlign_Right或者左对齐ADC_DataAlign_Left。一般我们选择右对齐模式。

ScanConvMode：可选参数为ENABLE和DISABLE，配置是否使用扫描。如果是单通道AD转换使用DISABLE，如果是多通道AD转换使用ENABLE。

EOCSelection：可选参数为ENABLE和DISABLE，指定通过轮询和中断来使用EOC（转换结束）标志进行转换。

ContinuousConvMode：可选参数为ENABLE和DISABLE，配置是启动自动连续转换还是单次转换。使用ENABLE配置为使能自动连续转换；使用DISABLE配置为单次转换，转换一次后停止需要手动控制才重新启动转换。

NbrOfConversion：AD规则转换通道数目。

DiscontinuousConvMode：不连续采样模式。一般为禁止模式。

NbrOfDiscConversion：ADC不连续转换通道数目。

ExternalTrigConv：外部触发选择，图29_1_ 中列举了很多外部触发条件，可根据项目需求配置触发来源。实际上，我们一般使用软件自动触发。

ExternalTrigConvEdge：外部触发极性选择，如果使用外部触发，可以选择触发的极性，可选有禁止触发检测、上升沿触发检测、下降沿触发检测以及上升沿和下降沿均可触发检测。

DMAContinuousRequests：DMA请求连续转换，开启DMA传输时用到。

ADC_ChannelConfTypeDef结构体
'''''''''''''''''''''''''''''

ADC_ChannelConfTypeDef结构体定义在stm32f4xx_adc.h文件内，具体定义如下：

.. code-block:: c

   typedef struct {
         uint32_t  	Channel;       	/*ADC转换通道*/
      uint32_t		Rank;		/*ADC序列数目*/
      uint32_t		SamplingTime;    /*ADC采样时间*/
      uint32_t		Offset; 		/*预留未用到，设为0即可*/
   } ADC_HandleTypeDef;

Channel：ADC转换通道。

Rank：ADC序列数目

SamplingTime：ADC采样时间。

独立模式单通道采集实验
~~~~~~~~~~~~~~~~~~~~~~

STM32的ADC功能繁多，我们设计三个实验尽量完整的展示ADC的功能。首先是比较基础实用的单通道采集，实现开发板上电位器电压的采集，并通过串口打印至PC端串口调试助手。单通道采集适用AD转换完成中断，在中断服务函数中读取数据，不使用DMA传输，在多通道采集时才使用DMA传输。

硬件设计
'''''''''''''''

开发板板载一个贴片滑动变阻器，电路设计见 图29_5_。

.. image:: media/image6.png
   :align: center
   :alt: 图 29‑5 开发板电位器部分原理图
   :name: 图29_5

图 29‑5 开发板电位器部分原理图

贴片滑动变阻器的动触点通过连接至STM32芯片的ADC通道引脚。当我们旋转滑动变阻器调节旋钮时，其动触点电压也会随之改变，电压变化范围为0~3.3V，亦是开发板默认的ADC电压采集范围。

软件设计
'''''''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。

我们编写两个ADC驱动文件，bsp_adc.h 和
bsp_adc.c，用来存放ADC所用IO引脚的初始化函数以及ADC配置相关函数。

编程要点
===============

1) 初始ADC用到的GPIO；

2) 设置ADC的工作参数并初始化；

3) 设置ADC工作时钟；

4) 设置ADC转换通道顺序及采样时间；

5) 配置使能ADC转换完成中断，在中断内读取转换完数据；

6) 使能ADC；

7) 使能软件触发ADC转换。

ADC转换结果数据使用中断方式读取，这里没有使用DMA进行数据传输。

代码分析
===========

ADC宏定义
....................

代码清单 29‑1 ADC宏定义

.. code-block:: c

   //ADC GPIO 宏定义
   #define RHEOSTAT_ADC_GPIO_PORT              GPIOB
   #define RHEOSTAT_ADC_GPIO_PIN               GPIO_PIN_0
   #define RHEOSTAT_ADC_GPIO_CLK_ENABLE()      __GPIOB_CLK_ENABLE()
   //ADC 序号宏定义
   #define RHEOSTAT_ADC                        ADC1
   #define RHEOSTAT_ADC_CLK_ENABLE()           __ADC1_CLK_ENABLE()
   #define RHEOSTAT_ADC_CHANNEL                ADC_CHANNEL_8
   //ADC DR寄存器宏定义，ADC转换后的数字值则存放在这里
   #define RHEOSTAT_ADC_DR_ADDR                ((uint32_t)ADC1+0x4c)

使用宏定义引脚信息方便硬件电路改动时程序移植。

ADC GPIO初始化函数
.....................

.. code-block:: c
   :caption: 代码清单 29‑2 ADC GPIO初始化
   :name: 代码清单29_2

   static void Rheostat_ADC_GPIO_Config(void)
   {
      GPIO_InitTypeDef GPIO_InitStructure;
      // 使能 GPIO 时钟
      RHEOSTAT_ADC_GPIO_CLK_ENABLE();
      // 配置 IO
      GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN;
      GPIO_InitStructure.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStructure.Pull = GPIO_NOPULL ; //不上拉不下拉
      HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT, &GPIO_InitStructure);
   }

使用到GPIO时候都必须开启对应的GPIO时钟，GPIO用于AD转换功能必须配置为模拟输入模式。

配置ADC工作模式
.................

.. code-block:: c
   :caption: 代码清单 29‑3 ADC工作模式配置
   :name: 代码清单29_3

   static void Rheostat_ADC_Mode_Config(void)
   {
      // 开启ADC时钟
      RHEOSTAT_ADC_CLK_ENABLE();
      // -------------------ADC Init 结构体 参数 初始化-----------------------
      // ADC1
      ADC_Handle.Instance = RHEOSTAT_ADC;
      // 时钟为fpclk 4分频
      ADC_Handle.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4;
      // ADC 分辨率
      ADC_Handle.Init.Resolution = ADC_RESOLUTION_12B;
      // 禁止扫描模式，多通道采集才需要
      ADC_Handle.Init.ScanConvMode = DISABLE;
      // 连续转换
      ADC_Handle.Init.ContinuousConvMode = ENABLE;
      // 非连续转换
      ADC_Handle.Init.DiscontinuousConvMode = DISABLE;
      // 非连续转换个数
      ADC_Handle.Init.NbrOfDiscConversion   = 0;
      //禁止外部边沿触发
      ADC_Handle.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
      //使用软件触发，外部触发不用配置，注释掉即可
      //ADC_Handle.Init.ExternalTrigConv      = ADC_EXTERNALTRIGCONV_T1_CC1;
      //数据右对齐
      ADC_Handle.Init.DataAlign = ADC_DATAALIGN_RIGHT;
      //转换通道 1个
      ADC_Handle.Init.NbrOfConversion = 1;
      //使能连续转换请求
      ADC_Handle.Init.DMAContinuousRequests = ENABLE;
      //转换完成标志
      ADC_Handle.Init.EOCSelection          = DISABLE;
      // 初始化ADC
      HAL_ADC_Init(&ADC_Handle);
      //-------------------------------------------------------------------
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL;
      ADC_Config.Rank         = 1;
      // 采样时间间隔
      ADC_Config.SamplingTime = ADC_SAMPLETIME_56CYCLES;
      ADC_Config.Offset       = 0;
      // 配置 ADC 通道转换顺序为1，第一个转换，采样时间为3个时钟周期
      HAL_ADC_ConfigChannel(&ADC_Handle, &ADC_Config);
      HAL_ADC_Start_IT(&ADC_Handle);
   }

首先，使用ADC_HandleTypeDef和ADC_ChannelConfTypeDef结构体分别定义一个ADC初始化和ADC通道配置变量，这两个结构体我们之前已经有详细讲解。

我们调用RHEOSTAT_ADC_CLK_ENABLE()开启ADC时钟。

接下来我们使用ADC_HandleTypeDef结构体变量ADC_Handle来配置ADC的寄存器基地址指针、分频系数为4、ADC1为12位分辨率、单通道采集不需要扫描、启动连续转换、使用内部软件触发无需外部触发事件、使用右对齐数据格式、转换通道为1，并调用HAL_ADC_Init函数完成ADC1工作环境配置。

使用ADC_ChannelConfTypeDef结构体变量ADC_Config来配置ADC的通道、转换顺序，可选为1到16；采样周期选择，采样周期越短，ADC转换数据输出周期就越短但数据精度也越低，采样周期越长，ADC转换数据输出周期就越长同时数据精度越高。PC3对应ADC通道ADC_Channel_13，这里我们选择ADC_SampleTime_56Cycles即56周期的采样时间，调用HAL_ADC_ConfigChannel函数完成ADC1的配置。

利用ADC转换完成中断可以非常方便的保证我们读取到的数据是转换完成后的数据而不用担心该数据可能是ADC正在转换时“不稳定”的数据。我们使用HAL_ADC_Start_IT函数使能ADC转换完成中断，并在中断服务函数中读取转换结果数据。

ADC中断配置
.................

.. code-block:: c
   :caption: 代码清单 29‑4 ADC中断配置
   :name: 代码清单29_4

   // 配置中断优先级
   static void Rheostat_ADC_NVIC_Config(void)
   {
      HAL_NVIC_SetPriority(Rheostat_ADC_IRQ, 0, 0);
      HAL_NVIC_EnableIRQ(Rheostat_ADC_IRQ);
   }

在Rheostat_ADC_NVIC_Config函数中我们配置了ADC转换完成中断的优先级分组和优先级配置。

ADC中断服务函数
.................

.. code-block:: c
   :caption: 代码清单 29‑5 ADC中断服务函数
   :name: 代码清单29_5

   void ADC_IRQHandler(void)
   {
      HAL_ADC_IRQHandler(&ADC_Handle);
   }
   /**
   * @brief  转换完成中断回调函数（非阻塞模式）
   * @param  AdcHandle : ADC句柄
   * @retval 无
   */
   void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* AdcHandle)
   {
      /* 获取结果 */
      ADC_ConvertedValue = HAL_ADC_GetValue(AdcHandle);
   }

中断服务函数一般定义在stm32f4xx_it.c文件内，HAL_ADC_IRQHandler是HAL中自带的一个中断服务函数，他处理过程中会指向一个回调函数给我们去添加用户代码，这里我们使用HAL_ADC_ConvCpltCallback转换完成中断，在ADC转换完成后就会进入中断服务函数，在进入回调函数，我们在回调函数内直接读取ADC转换结果保存在变量ADC_ConvertedValue(在main.c中定义)中。

ADC_GetConversionValue函数是获取ADC转换结果值的库函数，只有一个形参为ADC句柄，该函数还返回一个16位的ADC转换结果值。

主函数
..........

.. code-block:: c
   :caption: 代码清单 29‑6 主函数
   :name: 代码清单29_6

   /**
   * @brief  主函数
   * @param  无
   * @retval 无
   */
   int main(void)
   {
      /* 配置系统时钟为180 MHz */
      SystemClock_Config();
      /* 初始化USART1 配置模式为 115200 8-N-1 */
      UARTx_Config();
      Rheostat_Init();
      while (1) {
         ADC_Vol =(float)ADC_ConvertedValue/4096*(float)3.3;//读取转换的AD值
         printf("\r\n The current AD value = 0x%04X \r\n", ADC_ConvertedValue);
         printf("\r\n The current AD value = %f V \r\n",ADC_Vol);
         Delay(0x8fffff);
   }

主函数先调用USARTx_Config函数配置调试串口相关参数，函数定义在bsp_debug_usart.c文件中。

接下来调用Rheostat _Init函数进行ADC初始化配置并启动ADC。Rheostat
_Init函数是定义在bsp_adc.c文件中，它只是简单的分别调用Rheostat_ADC_GPIO_Config
()、Rheostat_ADC_Mode_Config ()和Rheostat_ADC_NVIC_Config()。

Delay函数只是一个简单的延时函数。

在ADC中断服务函数中我们把AD转换结果保存在变量ADC_ConvertedValue中，根据我们之前的分析可以非常清楚的计算出对应的电位器动触点的电压值。

最后就是把相关数据打印至串口调试助手.

下载验证
=============

用USB线连接开发板的“USB转串口”接口跟电脑，在电脑端打开串口调试助手，把编译好的程序下载到开发板。在串口调试助手可看到不断有数据从开发板传输过来，此时我们旋转电位器改变其电阻值，那么对应的数据也会有变化。

独立模式多通道采集实验
~~~~~~~~~~~~~~~~~~~~~~

硬件设计
'''''''''''

开发板已通过排针接口把部分ADC通道引脚引出，我们可以根据需要选择使用。
实际使用时候必须注意保存ADC引脚是单独使用的，不可能与其他模块电路共用同一引脚。

软件设计
'''''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。

跟单通道例程一样，我们编写两个ADC驱动文件，bsp_adc.h 和 bsp_adc.c，用来存放ADC所用IO引脚的初始化函数以及ADC配置相关函数，
实际上这两个文件跟单通道实验的文件是非常相似的。

编程要点
============

1)初始化配置ADC目标引脚为模拟输入模式；

2)使能ADC时钟和DMA时钟；

3)配置DMA从ADC规矩数据寄存器传输数据到我们指定的存储区；

4)配置通用ADC为独立模式，采样4分频；

5)设置ADC为12位分辨率，启动扫描，连续转换，不需要外部触发；

6)设置ADC转换通道顺序及采样时间；

7)使能DMA请求，DMA在AD转换完自动传输数据到指定的存储区；

8)启动ADC转换；

9)使能软件触发ADC转换。

ADC转换结果数据使用DMA方式传输至指定的存储区，这样取代单通道实验使用中断服务的读取方法。实际上，多通道ADC采集一般使用DMA数据传输方式更加高效方便。

代码分析
=============

.. _adc宏定义-1:

ADC宏定义
...............

.. code-block:: c
   :caption: 代码清单 29‑7 多通道ADC相关宏定义
   :name: 代码清单29_7

   #define RHEOSTAT_NOFCHANEL      3

   /*=====================通道1 IO======================*/
   // PC3 通过调帽接电位器
   // ADC IO宏定义
   #define RHEOSTAT_ADC_GPIO_PORT1             GPIOB
   #define RHEOSTAT_ADC_GPIO_PIN1              GPIO_PIN_0
   #define RHEOSTAT_ADC_GPIO_CLK1_ENABLE()     __GPIOB_CLK_ENABLE()
   #define RHEOSTAT_ADC_CHANNEL1               ADC_CHANNEL_8
   /*=====================通道2 IO ======================*/
   // PA4 通过调帽接光敏电阻
   // ADC IO宏定义
   #define RHEOSTAT_ADC_GPIO_PORT2             GPIOB
   #define RHEOSTAT_ADC_GPIO_PIN2              GPIO_PIN_1
   #define RHEOSTAT_ADC_GPIO_CLK2_ENABLE()     __GPIOB_CLK_ENABLE()
   #define RHEOSTAT_ADC_CHANNEL2               ADC_CHANNEL_9
   /*=====================通道3 IO ======================*/
   // PA6 悬空，可用杜邦线接3V3或者GND来实验
   // ADC IO宏定义
   #define RHEOSTAT_ADC_GPIO_PORT3             GPIOA
   #define RHEOSTAT_ADC_GPIO_PIN3              GPIO_PIN_6
   #define RHEOSTAT_ADC_GPIO_CLK3_ENABLE()     __GPIOA_CLK_ENABLE()
   #define RHEOSTAT_ADC_CHANNEL3               ADC_CHANNEL_6

   // ADC 序号宏定义
   #define RHEOSTAT_ADC                        ADC1
   #define RHEOSTAT_ADC_CLK_ENABLE()           __ADC1_CLK_ENABLE()

   // ADC DR寄存器宏定义，ADC转换后的数字值则存放在这里
   #define RHEOSTAT_ADC_DR_ADDR                ((uint32_t)ADC1+0x4c)

   // ADC DMA 通道宏定义，这里我们使用DMA传输
   #define RHEOSTAT_ADC_DMA_CLK_ENABLE()       __DMA2_CLK_ENABLE()
   #define RHEOSTAT_ADC_DMA_CHANNEL            DMA_CHANNEL_0
   #define RHEOSTAT_ADC_DMA_STREAM             DMA2_Stream0

定义多个通道进行多通道ADC实验，并且定义DMA相关配置。

ADC GPIO初始化函数
..............................

.. code-block:: c
   :caption: 代码清单 29‑8 ADC GPIO初始化
   :name: 代码清单29_8

   static void Rheostat_ADC_GPIO_Config(void)
   {
      GPIO_InitTypeDef GPIO_InitStructure;
      /*=====================通道1======================*/
      // 使能 GPIO 时钟
      RHEOSTAT_ADC_GPIO_CLK1_ENABLE();
      // 配置 IO
      GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN1;
      GPIO_InitStructure.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStructure.Pull = GPIO_NOPULL ; //不上拉不下拉
      HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT1, &GPIO_InitStructure);
      /*=====================通道2======================*/
      // 使能 GPIO 时钟
      RHEOSTAT_ADC_GPIO_CLK2_ENABLE();
      // 配置 IO
      GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN2;
      GPIO_InitStructure.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStructure.Pull = GPIO_NOPULL ; //不上拉不下拉
      HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT2, &GPIO_InitStructure);
      /*=====================通道3======================*/
      // 使能 GPIO 时钟
      RHEOSTAT_ADC_GPIO_CLK3_ENABLE();
      // 配置 IO
      GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN3;
      GPIO_InitStructure.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStructure.Pull = GPIO_NOPULL ; //不上拉不下拉
      HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT3, &GPIO_InitStructure);
   }

使用到GPIO时候都必须开启对应的GPIO时钟，GPIO用于AD转换功能必须配置为模拟输入模式。

配置ADC工作模式
.................

.. code-block:: c
   :caption: 代码清单 29‑9 ADC工作模式配置
   :name: 代码清单29_9

   static void Rheostat_ADC_Mode_Config(void)
   {
      // ------------------DMA Init 结构体参数 初始化-------------------------
      // ADC1使用DMA2，数据流0，通道0，这个是手册固定死的
      // 开启DMA时钟
      RHEOSTAT_ADC_DMA_CLK_ENABLE();
      // 数据传输通道
      DMA_Init_Handle.Instance = RHEOSTAT_ADC_DMA_STREAM;
      // 数据传输方向为外设到存储器
      DMA_Init_Handle.Init.Direction = DMA_PERIPH_TO_MEMORY;
      // 外设寄存器只有一个，地址不用递增
      DMA_Init_Handle.Init.PeriphInc = DMA_PINC_DISABLE;
      // 存储器地址固定
      DMA_Init_Handle.Init.MemInc = DMA_MINC_ENABLE;
      // // 外设数据大小为半字，即两个字节
      DMA_Init_Handle.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
      //  存储器数据大小也为半字，跟外设数据大小相同
      DMA_Init_Handle.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
      // 循环传输模式
      DMA_Init_Handle.Init.Mode = DMA_CIRCULAR;
      // DMA 传输通道优先级为高，当使用一个DMA通道时，优先级设置不影响
      DMA_Init_Handle.Init.Priority = DMA_PRIORITY_HIGH;
      // 禁止DMA FIFO ，使用直连模式
      DMA_Init_Handle.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
      // FIFO 大小，FIFO模式禁止时，这个不用配置
      DMA_Init_Handle.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_HALFFULL;
      DMA_Init_Handle.Init.MemBurst = DMA_MBURST_SINGLE;
      DMA_Init_Handle.Init.PeriphBurst = DMA_PBURST_SINGLE;
      // 选择 DMA 通道，通道存在于流中
      DMA_Init_Handle.Init.Channel = RHEOSTAT_ADC_DMA_CHANNEL;
      //初始化DMA流，流相当于一个大的管道，管道里面有很多通道
      HAL_DMA_Init(&DMA_Init_Handle);
      HAL_DMA_Start (&DMA_Init_Handle,RHEOSTAT_ADC_DR_ADDR,
      (uint32_t)&ADC_ConvertedValue,RHEOSTAT_NOFCHANEL);
      // 开启ADC时钟
      RHEOSTAT_ADC_CLK_ENABLE();
      // -------------------ADC Init 结构体 参数 初始化-----------------------
      // ADC1
      ADC_Handle.Instance = RHEOSTAT_ADC;
      // 时钟为fpclk 4分频
      ADC_Handle.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4;
      // ADC 分辨率
      ADC_Handle.Init.Resolution = ADC_RESOLUTION_12B;
      // 扫描模式，多通道采集才需要
      ADC_Handle.Init.ScanConvMode = ENABLE;
      // 连续转换
      ADC_Handle.Init.ContinuousConvMode = ENABLE;
      // 非连续转换
      ADC_Handle.Init.DiscontinuousConvMode = DISABLE;
      // 非连续转换个数
      ADC_Handle.Init.NbrOfDiscConversion   = 0;
      //禁止外部边沿触发
      ADC_Handle.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
      //使用软件触发，外部触发不用配置，注释掉即可
      //ADC_Handle.Init.ExternalTrigConv      = ADC_EXTERNALTRIGCONV_T1_CC1;
      //数据右对齐
      ADC_Handle.Init.DataAlign = ADC_DATAALIGN_RIGHT;
      //转换通道个数
      ADC_Handle.Init.NbrOfConversion = RHEOSTAT_NOFCHANEL;
      //使能连续转换请求
      ADC_Handle.Init.DMAContinuousRequests = ENABLE;
      //转换完成标志
      ADC_Handle.Init.EOCSelection          = DISABLE;
      // 初始化ADC
      HAL_ADC_Init(&ADC_Handle);
      //-------------------------------------------------------------------
      // 配置 ADC 通道1转换顺序为1，第一个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL1;
      ADC_Config.Rank         = 1;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES;// 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle, &ADC_Config);
      // 配置 ADC 通道2转换顺序为2，第二个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL2;
      ADC_Config.Rank         = 2;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES; // 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle, &ADC_Config);
      // 配置 ADC 通道3转换顺序为3，第三个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL3;
      ADC_Config.Rank         = 3;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES;// 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle, &ADC_Config);
      HAL_ADC_Start_DMA(&ADC_Handle, (uint32_t*)&ADC_ConvertedValue,1);
   }

首先，我们使用了DMA_HandleTypeDef定义了一个DMA初始化类型变量，该结构体内容我们在DMA篇已经做了非常详细的讲解；另外还使用ADC_HandleTypeDef和ADC_ChannelConfTypeDef结构体分别定义一个ADC初始化和ADC通道配置变量，这两个结构体我们之前已经有详细讲解。

调用RHEOSTAT_ADC_DMA_CLK_ENABLE()和RHEOSTAT_ADC_CLK_ENABLE()函数开启ADC时钟以及开启DMA时钟。

我们需要对DMA进行必要的配置。首先设置外设基地址就是ADC的规则数据寄存器地址；存储器的地址就是我们指定的数据存储区空间，ADC_ConvertedValue是我们定义的一个全局数组名，它是一个无符号16位含有4个元素的整数数组；ADC规则转换对应只有一个数据寄存器所以地址不能递增，而我们定义的存储区是专门用来存放不同通道数据的，所以需要自动地址递增。ADC的规则数据寄存器只有低16位有效，实际存放的数据只有12位而已，所以设置数据大小为半字大小。ADC配置为连续转换模式DMA也设置为循环传输模式。设置好DMA相关参数后就使用HAL_DMA_Init函数初始化。

接下来我们使用ADC_HandleTypeDef和ADC_ChannelConfTypeDef来配置ADC为独立模式、分频系数为4、不需要设置DMA模式、20个周期的采样延迟，并调用HAL_ADC_ConfigChannel函数完成ADC通道的配置。

我们使用ADC_InitTypeDef结构体变量ADC_InitStructure来配置ADC1为12位分辨率、使能扫描模式、启动连续转换、使用内部软件触发无需外部触发事件、使用右对齐数据格式、转换通道为4，并调用ADC_Init函数完成ADC3工作环境配置。

ADC_ChannelConfTypeDef函数用来绑定ADC通道转换顺序和采样时间。分别绑定四个ADC通道引脚并设置相应的转换顺序，控制是否使能ADC的DMA请求，如果使能请求，并调用HAL_ADC_Start_DMA函数控制ADC转换启动。在ADC转换完成后就请求DMA实现数据传输。

主函数
............

.. code-block:: c
   :caption: 代码清单 29‑10 主函数
   :name: 代码清单29_10

   int main(void)
   {
      /* 配置系统时钟为168 MHz */
      SystemClock_Config();
      /* 初始化USART1 配置模式为 115200 8-N-1 */
      UARTx_Config();
      Rheostat_Init();
      while (1) {
      ADC_ConvertedValueLocal[0] =(float) ADC_ConvertedValue[0]/4096*(float)3.3;
      ADC_ConvertedValueLocal[1] =(float) ADC_ConvertedValue[1]/4096*(float)3.3;
      ADC_ConvertedValueLocal[2] =(float) ADC_ConvertedValue[2]/4096*(float)3.3;
      printf("\r\n CH1_PC3 value = %f V \r\n",ADC_ConvertedValueLocal[0]);
      printf("\r\n CH2_PA4 value = %f V \r\n",ADC_ConvertedValueLocal[1]);
      printf("\r\n CH3_PA6 value = %f V \r\n",ADC_ConvertedValueLocal[2]);
      printf("\r\n\r\n");
      Delay(0xffffff);
      }
   }

主函数先调用USARTx_Config函数配置调试串口相关参数，函数定义在bsp_debug_usart.c文件中。

接下来调用Rheostat_Init函数进行ADC初始化配置并启动ADC。Rheostat_Init函数是定义在bsp_adc.c文件中，
它只是简单的分别调用Rheostat_ADC_GPIO_Config()和Rheostat_ADC_Mode_Config ()。

Delay函数只是一个简单的延时函数。

我们配置了DMA数据传输所以它会自动把ADC转换完成后数据保存到数组ADC_ConvertedValue内，我们只要直接使用数组就可以了。经过简单地计算就可以得到每个通道对应的实际电压。

最后就是把相关数据打印至串口调试助手。

下载验证
============

将待测电压通过杜邦线接在对应引脚上，用USB线连接开发板的“USB转串口”接口跟电脑，在电脑端打开串口调试助手，把编译好的程序下载到开发板。在串口调试助手可看到不断有数据从开发板传输过来，此时我们改变输入电压值，那么对应的数据也会有变化。

三重ADC交替模式采集实验
~~~~~~~~~~~~~~~~~~~~~~~~~~~

AD转换包括采样阶段和转换阶段，在采样阶段才对通道数据进行采集；而在转换阶段只是将采集到的数据进行转换为数字量输出，此刻通道数据变化不会改变转换结果。独立模式的ADC采集需要在一个通道采集并且转换完成后才会进行下一个通道的采集。双重或者三重ADC的机制使用两个或以上ADC同时采样两个或以上不同通道的数据或者使用两个或以上ADC交叉采集同一通道的数据。双重或者三重ADC模式较独立模式一个最大的优势就是转换速度快。

我们这里只介绍三重ADC交替模式，关于双重或者三重ADC的其他模式与之类似，可以参考三重ADC交替模式使用。
三重ADC交替模式是针对同一通道的使用三个ADC交叉采集，就是在ADC1采样完等几个时钟周期后ADC2开始采样，
此时ADC1处在转换阶段，当ADC2采样完成再等几个时钟周期后ADC3就进行采样此时ADC1和ADC2处在转换阶段，
如果ADC3采样完成并且ADC1已经转换完成那么就可以准备下一轮的循环，这样充分利用转换阶段时间达到增快采样速度的效果。
AD转换过程见 图29_6_，利用ADC的转换阶段时间另外一个ADC进行采样，而不用像独立模式必须等待采样和转换结束后才进行下一次采样及转换。

.. image:: media/image7.png
   :align: center
   :alt: 图 29‑6 三重ADC同步规则模式
   :name: 图29_6

图 29‑6 三重ADC同步规则模式

硬件设计
'''''''''''''

三重ADC交叉模式是针对同一个通道的ADC采集模式，这种情况跟前门小节的单通道实验非常类似，
只是同时使用三个ADC对同一通道进行采集，所以电路设计与之相同即可，具体可参考 图29_5_。

软件设计
'''''''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。

跟单通道例程一样，我们编写两个ADC驱动文件，bsp_adc.h 和 bsp_adc.c，用来存放ADC所用IO引脚的初始化函数以及ADC配置相关函数，
实际上这两个文件跟单通道实验的文件非常相似。

编程要点
=============

1)初始化配置ADC目标引脚为模拟输入模式；

2)使能ADC1、ADC2、ADC3以及DMA时钟；

3)配置DMA控制将ADC通用规矩数据寄存器数据转存到指定存储区；

4)配置通用ADC为三重ADC交替模式，采样4分频，使用DMA模式2；

5)设置ADC1、ADC2和ADC3为12位分辨率，禁用扫描，连续转换，不需要外部触发；

6)设置ADC1、ADC2和ADC3转换通道顺序及采样时间；

7)使能ADC1的 DMA请求，在ADC转换完后自动请求DMA进行数据传输；

8)启动ADC1、ADC2和ADC3转换；

9)使能软件触发ADC转换。

ADC转换结果数据使用DMA方式传输至指定的存储区，这样取代单通道实验使用中断服务的读取方法。

代码分析
=============

ADC宏定义
...............

.. code-block:: c
   :caption: 代码清单 29‑11 多通道ADC相关宏定义
   :name: 代码清单29_11

   #define RHEOSTAT_NOFCHANEL      3
   // PC3 通过调帽接电位器
   // ADC IO宏定义
   #define RHEOSTAT_ADC_GPIO_PORT             GPIOC
   #define RHEOSTAT_ADC_GPIO_PIN              GPIO_PIN_2
   #define RHEOSTAT_ADC_GPIO_CLK_ENABLE()     __GPIOC_CLK_ENABLE()
   // ADC 序号宏定义
   #define RHEOSTAT_ADC1                      ADC1
   #define RHEOSTAT_ADC2                      ADC2
   #define RHEOSTAT_ADC3                      ADC3
   #define RHEOSTAT_ADC1_CLK_ENABLE()         __ADC1_CLK_ENABLE()
   #define RHEOSTAT_ADC2_CLK_ENABLE()         __ADC2_CLK_ENABLE()
   #define RHEOSTAT_ADC3_CLK_ENABLE()         __ADC3_CLK_ENABLE()
   #define RHEOSTAT_ADC_CHANNEL               ADC_CHANNEL_13
   // ADC DR寄存器宏定义，ADC转换后的数字值则存放在这里
   #define RHEOSTAT_ADC_DR_ADDR               ((uint32_t)0x40012308)
   // ADC DMA 通道宏定义，这里我们使用DMA传输
   #define RHEOSTAT_ADC_DMA_CLK_ENABLE()       __DMA2_CLK_ENABLE()
   #define RHEOSTAT_ADC_DMA_CHANNEL            DMA_CHANNEL_0
   #define RHEOSTAT_ADC_DMA_STREAM             DMA2_Stream0

双重或者三重ADC需要使用通用规则数据寄存器ADC_CDR，这点跟独立模式不同。
定义电位器动触点引脚作为三重ADC的模拟输入，三个ADC都是采集同一路信号。

ADC GPIO初始化函数
..............................

.. code-block:: c
   :caption: 代码清单 29‑12 ADC GPIO初始化
   :name: 代码清单29_12

    static void Rheostat_ADC_GPIO_Config(void)
   {
      GPIO_InitTypeDef GPIO_InitStructure;
      // 使能 GPIO 时钟
      RHEOSTAT_ADC_GPIO_CLK_ENABLE();
      // 配置 IO
      GPIO_InitStructure.Pin = RHEOSTAT_ADC_GPIO_PIN;
      GPIO_InitStructure.Mode = GPIO_MODE_ANALOG;
      GPIO_InitStructure.Pull = GPIO_NOPULL ; //不上拉不下拉
      HAL_GPIO_Init(RHEOSTAT_ADC_GPIO_PORT, &GPIO_InitStructure);
   }

使用到GPIO时候都必须开启对应的GPIO时钟，GPIO用于AD转换功能必须配置为模拟输入模式。

配置三重ADC交替模式
..............................

.. code-block:: c
   :caption: 代码清单 29‑13 三重ADC交替模式配置
   :name: 代码清单29_13

   static void Rheostat_ADC_Mode_Config(void)
   {
      ADC_MultiModeTypeDef   mode;
      // ------------------DMA Init 结构体参数 初始化-------------------------
      // ADC1使用DMA2，数据流0，通道0，这个是手册固定死的
      // 开启DMA时钟
      RHEOSTAT_ADC_DMA_CLK_ENABLE();
      // 数据传输通道
      DMA_Init_Handle.Instance = RHEOSTAT_ADC_DMA_STREAM;
      // 数据传输方向为外设到存储器
      DMA_Init_Handle.Init.Direction = DMA_PERIPH_TO_MEMORY;
      // 外设寄存器只有一个，地址不用递增
      DMA_Init_Handle.Init.PeriphInc = DMA_PINC_DISABLE;
      // 存储器地址固定
      DMA_Init_Handle.Init.MemInc = DMA_MINC_ENABLE;
      // // 外设数据大小为半字，即两个字节
      DMA_Init_Handle.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
      //  存储器数据大小也为半字，跟外设数据大小相同
      DMA_Init_Handle.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
      // 循环传输模式
      DMA_Init_Handle.Init.Mode = DMA_CIRCULAR;
      // DMA 传输通道优先级为高，当使用一个DMA通道时，优先级设置不影响
      DMA_Init_Handle.Init.Priority = DMA_PRIORITY_HIGH;
      // 禁止DMA FIFO ，使用直连模式
      DMA_Init_Handle.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
      // FIFO 大小，FIFO模式禁止时，这个不用配置
      DMA_Init_Handle.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_HALFFULL;
      DMA_Init_Handle.Init.MemBurst = DMA_MBURST_SINGLE;
      DMA_Init_Handle.Init.PeriphBurst = DMA_PBURST_SINGLE;
      // 选择 DMA 通道，通道存在于流中
      DMA_Init_Handle.Init.Channel = RHEOSTAT_ADC_DMA_CHANNEL;
      //初始化DMA流，流相当于一个大的管道，管道里面有很多通道
      HAL_DMA_Init(&DMA_Init_Handle);
      // 开启ADC时钟
      RHEOSTAT_ADC1_CLK_ENABLE();
      RHEOSTAT_ADC2_CLK_ENABLE();
      RHEOSTAT_ADC3_CLK_ENABLE();
      // -------------------ADC1 Init 结构体 参数 初始化----------------------
      // ADC1
      ADC_Handle1.Instance = RHEOSTAT_ADC1;
      // 时钟为fpclk 4分频
      ADC_Handle1.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4;
      // ADC 分辨率
      ADC_Handle1.Init.Resolution = ADC_RESOLUTION_12B;
      // 禁止扫描模式，多通道采集才需要
      ADC_Handle1.Init.ScanConvMode = DISABLE;
      // 连续转换
      ADC_Handle1.Init.ContinuousConvMode = ENABLE;
      // 非连续转换
      ADC_Handle1.Init.DiscontinuousConvMode = DISABLE;
      // 非连续转换个数
      ADC_Handle1.Init.NbrOfDiscConversion   = 0;
      //禁止外部边沿触发
      ADC_Handle1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
      //使用软件触发，外部触发不用配置，注释掉即可
      //ADC_Handle.Init.ExternalTrigConv      = ADC_EXTERNALTRIGCONV_T1_CC1;
      //数据右对齐
      ADC_Handle1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
      //转换通道个数
      ADC_Handle1.Init.NbrOfConversion = 1;
      //使能连续转换请求
      ADC_Handle1.Init.DMAContinuousRequests = ENABLE;
      //转换完成标志
      ADC_Handle1.Init.EOCSelection          = DISABLE;
      // 初始化ADC
      HAL_ADC_Init(&ADC_Handle1);
      //-------------------------------------------------------------------
      // 配置 ADC1 通道13转换顺序为1，第一个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL;
      ADC_Config.Rank         = 1;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES;// 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle1, &ADC_Config);
      // -------------------ADC2 Init 结构体 参数 初始化----------------------
      // ADC2
      ADC_Handle2.Instance = RHEOSTAT_ADC2;
      // 时钟为fpclk 4分频
      ADC_Handle2.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4;
      // ADC 分辨率
      ADC_Handle2.Init.Resolution = ADC_RESOLUTION_12B;
      // 禁止扫描模式，多通道采集才需要
      ADC_Handle1.Init.ScanConvMode = DISABLE;
      // 连续转换
      ADC_Handle2.Init.ContinuousConvMode = ENABLE;
      // 非连续转换
      ADC_Handle2.Init.DiscontinuousConvMode = DISABLE;
      // 非连续转换个数
      ADC_Handle2.Init.NbrOfDiscConversion   = 0;
      //禁止外部边沿触发
      ADC_Handle2.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
      //使用软件触发，外部触发不用配置，注释掉即可
      //ADC_Handle.Init.ExternalTrigConv      = ADC_EXTERNALTRIGCONV_T1_CC1;
      //数据右对齐
      ADC_Handle2.Init.DataAlign = ADC_DATAALIGN_RIGHT;
      //转换通道个数
      ADC_Handle2.Init.NbrOfConversion = 1;
      //使能连续转换请求
      ADC_Handle2.Init.DMAContinuousRequests = ENABLE;
      //转换完成标志
      ADC_Handle2.Init.EOCSelection          = DISABLE;
      // 初始化ADC
      HAL_ADC_Init(&ADC_Handle2);
      // 配置 ADC2 通道13转换顺序为1，第一个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL;
      ADC_Config.Rank         = 1;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES; // 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle2, &ADC_Config);

      // -------------------ADC33 Init 结构体 参数 初始化--------------------
      // ADC3
      ADC_Handle3.Instance = RHEOSTAT_ADC3;
      // 时钟为fpclk 4分频
      ADC_Handle3.Init.ClockPrescaler = ADC_CLOCKPRESCALER_PCLK_DIV4;
      // ADC 分辨率
      ADC_Handle3.Init.Resolution = ADC_RESOLUTION_12B;
      // 禁止扫描模式，多通道采集才需要
      ADC_Handle1.Init.ScanConvMode = DISABLE;
      // 连续转换
      ADC_Handle3.Init.ContinuousConvMode = ENABLE;
      // 非连续转换
      ADC_Handle3.Init.DiscontinuousConvMode = DISABLE;
      // 非连续转换个数
      ADC_Handle3.Init.NbrOfDiscConversion   = 0;
      //禁止外部边沿触发
      ADC_Handle3.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
      //使用软件触发，外部触发不用配置，注释掉即可
      //ADC_Handle.Init.ExternalTrigConv      = ADC_EXTERNALTRIGCONV_T1_CC1;
      //数据右对齐
      ADC_Handle3.Init.DataAlign = ADC_DATAALIGN_RIGHT;
      //转换通道个数
      ADC_Handle3.Init.NbrOfConversion = 1;
      //使能连续转换请求
      ADC_Handle3.Init.DMAContinuousRequests = ENABLE;
      //转换完成标志
      ADC_Handle3.Init.EOCSelection          = DISABLE;
      // 初始化ADC
      HAL_ADC_Init(&ADC_Handle3);
      // 配置 ADC3 通道13转换顺序为1，第一个转换，采样时间为3个时钟周期
      ADC_Config.Channel      = RHEOSTAT_ADC_CHANNEL;
      ADC_Config.Rank         = 1;
      ADC_Config.SamplingTime = ADC_SAMPLETIME_3CYCLES;// 采样时间间隔
      ADC_Config.Offset       = 0;
      HAL_ADC_ConfigChannel(&ADC_Handle3, &ADC_Config);

      /*配置三重AD采样*/
      mode.Mode = ADC_TRIPLEMODE_INTERL;
      mode.DMAAccessMode = ADC_DMAACCESSMODE_2;
      mode.TwoSamplingDelay = ADC_TWOSAMPLINGDELAY_5CYCLES;

      HAL_ADCEx_MultiModeConfigChannel(&ADC_Handle1, &mode);

      HAL_ADC_Start(&ADC_Handle2);
      HAL_ADC_Start(&ADC_Handle3);

      __HAL_LINKDMA(&ADC_Handle1, DMA_Handle, DMA_Init_Handle);
      __HAL_LINKDMA(&ADC_Handle2, DMA_Handle, DMA_Init_Handle);
      __HAL_LINKDMA(&ADC_Handle3, DMA_Handle, DMA_Init_Handle);
      HAL_ADCEx_MultiModeStart_DMA(&ADC_Handle1, (uint32_t *)ADC_ConvertedValue, 3);
   }

首先，我们使用了DMA_HandleTypeDef定义了一个DMA初始化类型变量，该结构体内容我们在DMA篇已经做了非常详细的讲解；
另外还使用ADC_HandleTypeDef和ADC_ChannelConfTypeDef结构体分别定义一个ADC初始化和ADC通道配置变量，这两个结构体我们之前已经有详细讲解。

调用RHEOSTAT_ADC_DMA_CLK_ENABLE()和RHEOSTAT_ADC_CLK_ENABLE()函数开启ADC时钟以及开启DMA时钟。

我们需要对DMA进行必要的配置。首先设置外设基地址就是ADC的通用规则数据寄存器地址；存储器的地址就是我们指定的数据存储区空间，ADC_ConvertedValue是我们定义的一个全局数组名，它是一个无符号32位有三个元素的整数数字；ADC规则转换对应只有一个数据寄存器所以地址不能递增，我们指定的存储区也需要递增地址。ADC的通用规则数据寄存器是32位有效，我们配置ADC为DMA模式2，设置数据大小为字大小。ADC配置为连续转换模式DMA也设置为循环传输模式。设置好DMA相关参数后就使能DMA的ADC通道。

接下来我们使用ADC_InitTypeDef结构体变量ADC_InitStructure来配置ADC1为12位分辨率、不使用扫描模式、启动连续转换、使用内部软件触发无需外部触发事件、使用右对齐数据格式、转换通道为1，并调用ADC_Init函数完成ADC1工作环境配置。ADC2和ADC3使用与ADC1相同配置即可。

ADC_ChannelConfTypeDef函数用来绑定ADC通道转换顺序和采样时间。绑定ADC通道引脚并设置相应的转换顺序。
接下来我们使用ADC_MultiModeTypeDef结构体变量mode来配置ADC为三重ADC交替模式、分频系数为4、需要设置DMA模式2、10个周期的采样延迟。

HAL_ADC_Start函数控制ADC转换启动。

HAL_ADCEx_MultiModeConfigChannel函数控制是否使能ADC的DMA请求，如果使能请求，并调用HAL_ADCEx_MultiModeStart_DMA函数使能DMA，则在ADC转换完成后就请求DMA实现数据传输。三重模式只需使能ADC1的DMA通道。

主函数
=============

.. code-block:: c
   :caption: 代码清单 29‑14 主函数
   :name: 代码清单29_14

   int main(void)
   {
      /* 配置系统时钟为168 MHz */
      SystemClock_Config();

      /* 初始化USART1 配置模式为 115200 8-N-1 */
      DEBUG_USART_Config();

      Rheostat_Init();
      while (1) {
            Delay(0xffffee);
            ADC_ConvertedValueLocal[0] =(float)((uint16_t)
                                       ADC_ConvertedValue[0]*3.3/4096);
            ADC_ConvertedValueLocal[1] =(float)((uint16_t)
                                       ADC_ConvertedValue[1]*3.3/4096);
            ADC_ConvertedValueLocal[2] =(float)((uint16_t)
                                       ADC_ConvertedValue[2]*3.3/4096);
            printf("\r\n The current AD value = 0x%08X \r\n",
                  ADC_ConvertedValue[0]);
            printf("\r\n The current AD value = 0x%08X \r\n",
                  ADC_ConvertedValue[1]);
            printf("\r\n The current AD value = 0x%08X \r\n",
                  ADC_ConvertedValue[2]);
            printf("\r\n The current ADC1 value = %f V \r\n",
                  ADC_ConvertedValueLocal[0]);
            printf("\r\n The current ADC2 value = %f V \r\n",
                  ADC_ConvertedValueLocal[1]);
            printf("\r\n The current ADC3 value = %f V \r\n",
                  ADC_ConvertedValueLocal[2]);
      }
   }

主函数先初始化系统时钟再调用USARTx_Config函数配置调试串口相关参数，函数定义在bsp_debug_usart.c文件中。

接下来调用Rheostat_Init函数进行ADC初始化配置并启动ADC。Rheostat_Init函数是定义在bsp_adc.c文件中，
它只是简单的分别调用Rheostat_ADC_GPIO_Config()和Rheostat_ADC_Mode_Config ()。

Delay函数只是一个简单的延时函数。

我们配置了DMA数据传输所以它会自动把ADC转换完成后数据保存到数组变量ADC_ConvertedValue内，根据DMA模式2的数据存放规则，ADC_ConvertedValue[0]的低16位存放ADC1数据、高16位存放ADC2数据，ADC_ConvertedValue[1]的低16位存放ADC3数据、高16位存放ADC1数据，ADC_ConvertedValue[2]的低16位存放ADC2数据、高16位存放ADC3数据，我们可以根据需要提取出对应ADC的转换结果数据。经过简单地计算就可以得到每个ADC对应的实际电压。

最后就是把相关数据打印至串口调试助手。

下载验证
=============

保证开发板相关硬件连接正确，用USB线连接开发板“USB
转串口”接口跟电脑，在电脑端打开串口调试助手，把编译好的程序下载到开发板。在串口调试助手可看到不断有数据从开发板传输过来，此时我们旋转电位器改变其电阻值，那么对应的数据也会有变化。
