RCC—使用HSE/HSI配置时钟
-----------------------

本章参考资料：《STM32F4xx中文参考手册》RCC章节。

学习本章时，配合《STM32F4xx中文参考手册》RCC章节一起阅读，效果会更佳，特别是涉及到寄存器说明的部分。

RCC ：reset clock control  复位和时钟控制器。本章我们主要讲解时钟部分，特别是要着重理解时钟树，
理解了时钟树，F407的一切时钟的来龙去脉都会了如指掌。

RCC主要作用—时钟部分
~~~~~~~~~~~~~~~~~~~~

设置系统时钟SYSCLK、设置AHB分频因子（决定HCLK等于多少）、设置APB2分频因子（决定PCLK2等于多少）、设置APB1分频因子（
决定PCLK1等于多少）、设置各个外设的分频因子；控制AHB、APB2和APB1这三条总线时钟的开启、控制每个外设的时钟的开启。
对于SYSCLK、HCLK、PCLK2、PCLK1这四个时钟的配置一般是：HCLK = SYSCLK=PLLCLK = 168M，
PCLK1=HCLK/2 = 84M，
PCLK1=HCLK/4 = 42M。这个时钟配置也是库函数的标准配置，我们用的最多的就是这个。

RCC框图剖析—时钟部分
~~~~~~~~~~~~~~~~~~~~

时钟树单纯讲理论的话会比较枯燥，如果选取一条主线，并辅以代码，先主后次讲解的话会很容易，而且记忆还更深刻。我们这里选取库函数时钟系统时钟函数：SetSysClock(); 以这个函数的编写流程来讲解时钟树，这个函数也是我们用库的时候默认的系统时钟设置函数。该函数的功能是利用HSE把时钟设置为：
HCLK = SYSCLK=PLLCLK = 168M，PCLK1=HCLK/2 = 84M，PCLK1=HCLK/4 = 42M
下面我们就以这个代码的流程为主线，来分析时钟树，对应的是图中的黄色部分，代码流程在时钟树中以数字的大小顺序标识。

.. image:: media/image2.png
   :align: center
   :alt: 图 15‑1 STM32时钟树
   :name: 图15_1

系统时钟
^^^^^^^^

①HSE高速外部时钟信号
''''''''''''''''''''

HSE是高速的外部时钟信号，可以由有源晶振或者无源晶振提供，频率从4-26MHZ不等。当使用有源晶振时，时钟从OSC_IN引脚进入，OSC_OUT引脚悬空，当选用无源晶振时，时钟从OSC_IN和OSC_OUT进入，并且要配谐振电容。HSE我们使用25M的无源晶振。如果我们使用HSE或者HSE经过PLL倍频之后的时钟作为系统时钟SYSCLK，当HSE故障时候，不仅HSE会被关闭，PLL也会被关闭，此时高速的内部时钟时钟信号HSI会作为备用的系统时钟，直到HSE恢复正常，HSI=16M。

②锁相环PLL
''''''''''

PLL的主要作用是对时钟进行倍频，然后把时钟输出到各个功能部件。PLL有两个，一个是主PLL，另外一个是专用的PLLI2S，他们均由HSE或者HSI提供时钟输入信号。

主PLL有两路的时钟输出，第一个输出时钟PLLCLK用于系统时钟，F407里面最高是168M，
第二个输出用于USB OTG FS的时钟（48M）、RNG和SDIO时钟（<=48M）。专用的PLLI2S用于生成精确时钟，给I2S提供时钟。

HSE或者HSI经过PLL时钟输入分频因子M（2~63）分频后，成为VCO的时钟输入，VCO的时钟必须在1~2M之间，我们选择HSE=25M作为PLL的时钟输入，M设置为25，那么VCO输入时钟就等于1M。

VCO输入时钟经过VCO倍频因子N倍频之后，成为VCO时钟输出，VCO时钟必须在192~432M之间。我们配置N为336，
则VCO的输出时钟等于336M。如果要把系统时钟超频，就得在VCO倍频系数N这里做手脚。
PLLCLK_OUTMAX = VCOCLK_OUTMAX/P_MIN = 432/2=216M，即F407最高可超频到216M。

VCO输出时钟之后有三个分频因子：PLLCLK分频因子p，USB OTG FS/RNG/SDIO时钟分频因子Q，
分频因子R（F446才有，F407没有）。p可以取值2、4、6、8,我们配置为2，则得到PLLCLK=168M。
Q可以取值4~15，但是USB OTG FS必须使用48M，Q=VCO输出时钟336/48=7。
有关PLL的配置有一个专门的RCC PLL配置寄存器RCC_PLLCFGR，
具体描述看手册即可。

PLL的时钟配置经过，稍微整理下可由如下公式表达：

VCOCLK_IN = PLLCLK_IN / M  = HSE / 25 = 1M

VCOCLK_OUT = VCOCLK_IN * N = 1M * 336 = 336M

PLLCLK_OUT=VCOCLK_OUT/P=336/2=168M

USBCLK = VCOCLK_OUT/Q=336/7=48。。

③3.系统时钟SYSCLK
''''''''''''''''''''''''''''

系统时钟来源可以是：HSI、PLLCLK、HSE，具体的由时钟配置寄存器RCC_CFGR的SW位配置。
我们这里设置系统时钟：SYSCLK = PLLCLK = 168M。如果系统时钟是由HSE经过PLL倍频之后的PLLCLK得到，
当HSE出现故障的时候，系统时钟会切换为HSI=16M，直到HSE恢复正常为止。

④AHB总线时钟HCLK
''''''''''''''''''''''''''''''

系统时钟SYSCLK经过AHB预分频器分频之后得到时钟叫AHB总线时钟，即HCLK，分频因子可以是:[1,2,4，8，16，64，128，256，512]，
具体的由时钟配置寄存器RCC_CFGR的HPRE位设置。片上大部分外设的时钟都是经过HCLK分频得到，
至于AHB总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，我们这里只需粗线条的设置好APB的时钟即可。我们这里设置为1分频，即HCLK=SYSCLK=168M。

⑤APB2总线时钟HCLK2
''''''''''''''''''''''''''''''''

APB2总线时钟PCLK2由HCLK经过高速APB2预分频器得到，分频因子可以是:[1,2,4，8，16]，
具体由时钟配置寄存器RCC_CFGR的PPRE2位设置。HCLK2属于高速的总线时钟，片上高速的外设就挂载到这条总线上，
比如全部的GPIO、USART1、SPI1等。至于APB2总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，
我们这里只需粗线条的设置好APB2的时钟即可。我们这里设置为2分频，即PCLK2 = HCLK /2= 84M。

⑥APB1总线时钟HCLK1
''''''''''''''''''

APB1总线时钟PCLK1由HCLK经过低速APB预分频器得到，分频因子可以是:[1,2,4，8，16]，
具体由时钟配置寄存器RCC_CFGR的PPRE1位设置。
HCLK1属于低速的总线时钟，最高为42M，片上低速的外设就挂载到这条总线上，比如USART2/3/4/5、SPI2/3，I2C1/2等。
至于APB1总线上的外设的时钟设置为多少，得等到我们使用该外设的时候才设置，
我们这里只需粗线条的设置好APB1的时钟即可。我们这里设置为4分频，即PCLK1 = HCLK/4 = 42M。

设置系统时钟库函数
''''''''''''''''''

上面的6个步骤对应的设置系统时钟库函数具体见代码清单 151，为了方便阅读，已经把跟407不相关的代码删掉，
把英文注释翻译成了中文，并把代码标上了序号，总共6个步骤。该函数是直接操作寄存器的，有关寄存器部分请参考数据手册的RCC的寄存器描述部分。

.. code-block:: c
   :caption: 代码 15‑1 设置系统时钟库函数
   :name: 代码清单15_1

    /**
    * @brief  System Clock Configuration
    *         The system Clock is configured as follow :
    *            System Clock source            = PLL (HSE)
    *            SYSCLK(Hz)                     = 168000000
    *            HCLK(Hz)                       = 168000000
    *            AHB Prescaler                  = 1
    *            APB1 Prescaler                 = 4
    *            APB2 Prescaler                 = 2
    *            HSE Frequency(Hz)              = 8000000
    *            PLL_M                          = 25
    *            PLL_N                          = 336
    *            PLL_P                          = 2
    *            PLL_Q                          = 7
    *            VDD(V)                         = 3.3
    *            Main regulator output voltage  = Scale1 mode
    *            Flash Latency(WS)              = 5
    * @param  None
    * @retval None
    */
    static void SystemClock_Config(void)
    {
        RCC_ClkInitTypeDef RCC_ClkInitStruct;
        RCC_OscInitTypeDef RCC_OscInitStruct;

        /* Enable Power Control clock */
        __HAL_RCC_PWR_CLK_ENABLE();

        /* The voltage scaling allows optimizing the power consumptio
        when the device is clocked below the maximum system frequency,
        to update the voltage scaling value
            regarding system frequency refer to product datasheet.  */
        __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

        /* Enable HSE Oscillator and activate PLL with HSE as source */
        RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
        RCC_OscInitStruct.HSEState = RCC_HSE_ON;
        RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
        RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
        RCC_OscInitStruct.PLL.PLLM = 25;
        RCC_OscInitStruct.PLL.PLLN = 336;
        RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
        RCC_OscInitStruct.PLL.PLLQ = 7;
        if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) {
            while (1) {};
        }

        /* Select PLL as system clock source and configure the HCLK,
        PCLK1 and PCLK2 clocks dividers */
        RCC_ClkInitStruct.ClockType = (RCC_CLOCKTYPE_SYSCLK |
        RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_PCLK1| RCC_CLOCKTYPE_PCLK2);
        RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
        RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
        RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
        RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
        if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct,FLASH_LATENCY_5) != HAL_OK) {
            while (1) {};
        }

        /* STM32F405x/407x/415x/417x Revision Z devices:prefetch is supported  */
        if (HAL_GetREVID() == 0x1001) {
            /* Enable the Flash prefetch */
            __HAL_FLASH_PREFETCH_BUFFER_ENABLE();
        }
    }


其他时钟
^^^^^^^^

通过对系统时钟设置的讲解，整个时钟树我们已经把握的有六七成，剩下的时钟部分我们讲解几个重要的。

A、RTC时钟
''''''''''

RTCCLK 时钟源可以是 HSE 1 MHz（ HSE 由一个可编程的预分频器分频）、LSE 或者LSI时钟。
选择方式是编程 RCC 备份域控制寄存器 (RCC_BDCR)
中的 RTCSEL[1:0] 位和 RCC时钟配置寄存器 (RCC_CFGR) 中的 RTCPRE[4:0] 位。所做的选择只能通过复位备份域的方式修改。
我们通常的做法是由LSE给RTC提供时钟，大小为32.768KHZ。LSE由外接的晶体谐振器产生，所配的谐振电容精度要求高，不然很容易不起震。

B、独立看门狗时钟
'''''''''''''''''

独立看门狗时钟由内部的低速时钟LSI提供，大小为32KHZ。

C、I2S时钟
''''''''''

I2S时钟可由外部的时钟引脚I2S_CKIN输入，也可由专用的PLLI2SCLK提供，具体的由RCC 时钟配置寄存器 (
RCC_CFGR)的I2SSCR位配置。我们在使用I2S外设驱动W8978的时候，使用的时钟是PLLI2SCLK，这样就可以省掉一个有源晶振。

D、PHY以太网时钟
''''''''''''''''''''''''''

F407要想实现以太网功能，除了有本身内置的MAC之外，还需要外接一个PHY芯片，常见的PHY芯片有DP83848和LAN8720，
其中DP83848支持MII和RMII接口，LAN8720只支持RMII接口。野火F407开发板用的是RMII接口，选择的PHY芯片是LAB8720。
使用RMII接口的好处是使用的IO减少了一半，速度还是跟MII接口一样。当使用RMII接口时，PHY芯片只需输出一路时钟给MCU即可，
如果是MII接口，PHY芯片则需要提供两路时钟给MCU。

E、USB PHY 时钟
''''''''''''''''''''''''''''

F407的USB没有集成PHY，要想实现USB高速传输的话，必须外置USB PHY芯片，常用的芯片是USB3300。
当外接USB PHY芯片时，PHY芯片需要给MCU提供一个时钟。

外扩USB3300会占用非常多的IO，跟SDRAM和RGB888的IO会复用的很厉害，鉴于USB高速传输用的比较少，野火F407骄阳就没有外扩这个芯片。

F、MCO时钟输出
''''''''''''''''''''''''''''

MCO是microcontroller clock output的缩写，是微控制器时钟输出引脚，主要作用是可以对外提供时钟，相当于一个有源晶振。
F407中有两个MCO，由PA8/PC9复用所得。MCO1所需的时钟源通过 RCC 时钟配置寄存器 (RCC_CFGR)
中的 MCO1PRE[2:0] 和 MCO1[1:0]
位选择。MCO2所需的时钟源通过 RCC 时钟配置寄存器 (RCC_CFGR) 中的 MCO2PRE[2:0] 和 MCO2位选择。
有关MCO的IO、时钟选择和输出速率的具体信息如下表所示：

========  ===  ==============================  ============
时钟输出  IO              时钟来源             最大输出速率
========  ===  ==============================  ============
MCO1      PA8  HSI、LSE、HSE、PLLCLK           100M
MCO2      PC9  HSE、PLLCLK、SYSCLK、PLLI2SCLK  100M
========  ===  ==============================  ============

配置系统时钟实验
~~~~~~~~~~~~~~~~

使用HSE
^^^^^^^

一般情况下，我们都是使用HSE，然后HSE经过PLL倍频之后作为系统时钟。F407系统时钟最高为168M，
这个是官方推荐的最高的稳定时钟，如果你想铤而走险，也可以超频，超频最高能到216M。 

如果我们使用库函数编程，当程序来到main函数之前，启动文件：startup_stm32f40xxx.s已经调用SystemInit()函数把系统时钟初始化成168MHZ，SystemInit()在库文件：system_stm32f4xx.c中定义。如果我们想把系统时钟设置低一点或者超频的话，可以修改底层的库文件，但是为了维持库的完整性，我们可以根据时钟树的流程自行写一个。

使用HSI
^^^^^^^

当HSE直接或者间接（HSE经过PLL倍频）的作为系统时钟的时候，如果HSE发生故障，不仅HSE会被关闭，连PLL也会被关闭，这个时候系统会自动切换HSI作为系统时钟，此时SYSCLK=HSI=16M，如果没有开启CSS和CSS中断的话，那么整个系统就只能在低速率运行，这是系统跟瘫痪没什么两样。

如果开启了CSS功能的话，那么可以当HSE故障时，在CSS中断里面采取补救措施，使用HSI，重新设置系统频率为168M，让系统恢复正常使用。但这只是权宜之计，并非万全之策，最好的方法还是要采取相应的补救措施并报警，然后修复HSE。临时使用HSI只是为了把损失降低到最小，毕竟HSI较于HSE精度还是要低点。

F103系列中，使用HSI最大只能把系统设置为64M，并不能跟使用HSE一样把系统时钟设置为72M，究其原因是HSI在进入PLL倍频的时候必须2分频，导致PLL倍频因子调到最大也只能到64M，而HSE进入PLL倍频的时候则不用2分频。

在F407中，无论是使用HSI还是HSE都可以把系统时钟设置为168M，因为HSE或者HSI在进入PLL倍频的时候都会被分频为1M之后再倍频。

还有一种情况是，有些用户不想用HSE，想用HSI，但是又不知道怎么用HSI来设置系统时钟，因为调用库函数都是使用HSE，下面我们给出个使用HSI配置系统时钟例子，起个抛砖引玉的作用。

硬件设计
^^^^^^^^

1. RCC

2. LED一个

RCC是单片机内部资源，不需要外部电路。通过LED闪烁的频率来直观的判断不同系统时钟频率对软件延时的效果。

软件设计
^^^^^^^^

我们编写两个RCC驱动文件，bsp_clkconfig.h和bsp_clkconfig.c，用来存放RCC系统时钟配置函数。

编程要点
''''''''

编程要点对应着时钟树图中的序号。

1、开启HSE/HSI ，并等待 HSE/HSI 稳定

2、设置 AHB、APB2、APB1的预分频因子

3、设置PLL的时钟来源，设置VCO输入时钟 分频因子PLL_M，设置VCO输出时钟倍频因子PLL_N，设置PLLCLK时钟分频因子PLL_P，
设置OTG FS,SDIO,RNG 时钟分频因子 PLL_Q。

4、开启PLL，并等待PLL稳定

5、把PLLCK切换为系统时钟SYSCLK

6、读取时钟切换状态位，确保PLLCLK被选为系统时钟

代码分析
''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。

使用HSE配置系统时钟
======================

.. code-block:: c
   :caption: 代码 15‑2 HSE作为系统时钟来源
   :name: 代码清单15_2

    /*
    * 使用HSE时，设置系统时钟的步骤
    * 1、开启HSE ，并等待 HSE 稳定
    * 2、设置 AHB、APB2、APB1的预分频因子
    * 3、设置PLL的时钟来源
    *    设置VCO输入时钟 分频因子        m
    *    设置VCO输出时钟 倍频因子        n
    *    设置PLLCLK时钟分频因子          p
    *    设置OTG FS,SDIO,RNG时钟分频因子 q
    * 4、开启PLL，并等待PLL稳定
    * 5、把PLLCK切换为系统时钟SYSCLK
    * 6、读取时钟切换状态位，确保PLLCLK被选为系统时钟
    */
    /*
    * m: VCO输入时钟 分频因子，取值2~63
    * n: VCO输出时钟 倍频因子，取值50~432
    * p: PLLCLK时钟分频因子  ，取值2，4，6，8
    * q: OTG FS,SDIO,RNG时钟分频因子，取值4~15
    * 函数调用举例，使用HSE设置时钟
    * SYSCLK=HCLK=168MHz,PCLK2=HCLK/2=84MHz,PCLK1=HCLK/4=42MHz
    * HSE_SetSysClock(25, 336, 2, 7);
    * 
    HSE作为时钟来源，经过PLL倍频作为系统时钟，这是通常的做法

    */
    void HSE_SetSysClock(uint32_t m, uint32_t n, uint32_t p, uint32_t q)
    {
        RCC_ClkInitTypeDef RCC_ClkInitStruct;
        RCC_OscInitTypeDef RCC_OscInitStruct;

        /* Enable Power Control clock */
        __HAL_RCC_PWR_CLK_ENABLE();

        /* The voltage scaling allows optimizing the power consumption when
        the device is clocked below the maximum system frequency, to update
        the voltage scaling value regarding system frequency refer to product
        datasheet.  */
        __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

        /* Enable HSE Oscillator and activate PLL with HSE as source */
        RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
        RCC_OscInitStruct.HSEState = RCC_HSE_ON;
        RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
        RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
        RCC_OscInitStruct.PLL.PLLM = 25;
        RCC_OscInitStruct.PLL.PLLN = 336;
        RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
        RCC_OscInitStruct.PLL.PLLQ = 7;
        if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) {
            while (1) {};
        }

        /* Select PLL as system clock source and configure the HCLK,
        PCLK1 and PCLK2 clocks dividers */
        RCC_ClkInitStruct.ClockType = (RCC_CLOCKTYPE_SYSCLK |
                                        RCC_CLOCKTYPE_HCLK |
                                        RCC_CLOCKTYPE_PCLK1 |
                                        RCC_CLOCKTYPE_PCLK2);
        RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
        RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
        RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
        RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
        if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) !=
                                HAL_OK) {
            while (1) {};
        }

        /* STM32F405x/407x/415x/417x Revision Z devices: prefetch is supported  */
        if (HAL_GetREVID() == 0x1001) {
            /* Enable the Flash prefetch */
            __HAL_FLASH_PREFETCH_BUFFER_ENABLE();
        }
    }

这个函数采用库函数编写， 代码理解参考注释即可。函数有4个形参m、n、p、q，具体说明见表 15-1 PLL 配置因子。

表 15-1 PLL 配置因子

====  ==============================  ========
形参             形参说明             取值范围
====  ==============================  ========
m     VCO输入时钟分频因子             2~63
n     VCO输出时钟倍频因子             192~432
p     PLLCLK时钟分频因子              2/4/6/8
q     OTGFS,SDIO,RNG时钟分频因子	4~15
====  ==============================  ========

HSE我们使用25M，参数m我们一般也设置为25，所以我们需要修改系统时钟的时候只需要修改参数n和p即可，SYSCLK=PLLCLK=HSE/m*n/p。

函数调用举例：HSE_SetSysClock(25, 336, 2, 7) 把系统时钟设置为168M，这个跟库里面的系统时钟配置是一样的。
HSE_SetSysClock(25, 432, 2, 9)把系统
时钟设置为216M，这个是超频，要慎用。

使用HSI配置系统时钟
====================

.. code-block:: c

    /*
    * 使用HSI时，设置系统时钟的步骤
    * 1、开启HSI ，并等待 HSI 稳定
    * 2、设置 AHB、APB2、APB1的预分频因子
    * 3、设置PLL的时钟来源
    *    设置VCO输入时钟 分频因子        m
    *    设置VCO输出时钟 倍频因子        n
    *    设置SYSCLK时钟分频因子          p
    *    设置OTG FS,SDIO,RNG时钟分频因子 q
    * 4、开启PLL，并等待PLL稳定
    * 5、把PLLCK切换为系统时钟SYSCLK
    * 6、读取时钟切换状态位，确保PLLCLK被选为系统时钟
    */

    /*
    * m: VCO输入时钟 分频因子，取值2~63
    * n: VCO输出时钟 倍频因子，取值192~432
    * p: PLLCLK时钟分频因子  ，取值2，4，6，8
    * q: OTG FS,SDIO,RNG时钟分频因子，取值4~15
    * 函数调用举例，使用HSI设置时钟
    * SYSCLK=HCLK=168M,PCLK2=HCLK/2=84M,PCLK1=HCLK/4=42M
    * HSI_SetSysClock(16, 336, 2, 7);
    * HSE作为时钟来源，经过PLL倍频作为系统时钟，这是通常的做法
    void HSI_SetSysClock(uint32_t m, uint32_t n, uint32_t p, uint32_t q)
    {
        RCC_ClkInitTypeDef RCC_ClkInitStruct;
        RCC_OscInitTypeDef RCC_OscInitStruct;

        /* Enable Power Control clock */
        __HAL_RCC_PWR_CLK_ENABLE();

        /* The voltage scaling allows optimizing the power consumption
        when the device is clocked below the maximum system frequency,
        to update the voltage scaling value  regarding system
        frequency refer toproduct datasheet.  */
        __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

        /* Enable HSE Oscillator and activate PLL with HSE as source */
        RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
        RCC_OscInitStruct.HSEState = RCC_HSE_ON;
        RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
        RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
        RCC_OscInitStruct.PLL.PLLM = 25;
        RCC_OscInitStruct.PLL.PLLN = 336;
        RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
        RCC_OscInitStruct.PLL.PLLQ = 7;
        if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) {
            while (1) {};
        }

        /* Select PLL as system clock source and configure the HCLK,
        PCLK1 and PCLK2 clocks dividers */
        RCC_ClkInitStruct.ClockType = (RCC_CLOCKTYPE_SYSCLK |
                                    RCC_CLOCKTYPE_HCLK |
                                    RCC_CLOCKTYPE_PCLK1 |
                                    RCC_CLOCKTYPE_PCLK2);
        RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
        RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
        RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
        RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
        if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) !=
            HAL_OK) {
            while (1) {};
        }

        /* STM32F405x/407x/415x/417x Revision Z devices: prefetch is supported  */
        if (HAL_GetREVID() == 0x1001) {
            /* Enable the Flash prefetch */
            __HAL_FLASH_PREFETCH_BUFFER_ENABLE();
        }
    }


这个函数采用库函数编写， 代码理解参考注释即可。函数有4个形参m、n、p、q，具体说明见表 15-2 PLL 配置因子。

表 15-2 PLL 配置因子

====  ==============================  ========
形参             形参说明             取值范围
====  ==============================  ========
m     VCO输入时钟分频因子             2~63
n     VCO输出时钟倍频因子             192~432
p     PLLCLK时钟分频因子              2/4/6/8
q     OTGFS,SDIO,RNG时钟分频因子	4~15
====  ==============================  ========

HSI为16M，参数m我们一般也设置为16，所以我们需要修改系统时钟的时候只需要修改参数n和p即可，SYSCLK=PLLCLK=HSI/m*n/p。

函数调用举例：HSI_SetSysClock(16, 336, 2, 7) 把系统时钟设置为168M，这个跟库里面的系统时钟配置是一样的。HSI_SetSysClock(16, 432, 2, 9)把系统时钟设置为216M，这个是超频，要慎用。

软件延时
====================

.. code-block:: c

    void Delay(__IO uint32_t nCount)

    {

        for (; nCount != 0; nCount--);

    }

软件延时函数，使用不同的系统时钟，延时时间不一样，可以通过LED闪烁的频率来判断。

MCO输出
====================

在F407中，PA8/PC9可以复用为MCO1/2引脚，对外提供时钟输出，我们也可以用示波器监控该引脚的输出来判断我们的系统时钟是否设置正确野火F407开发板中PA8与 PC9都引出来了，在摄像头接口位置，都可以使用示波器来监控波形。

主函数
====================

.. code-block:: c

     // 使用HSE或者HSI 配置系统时钟
    #include "stm32f4xx.h"
    #include "./rcc/bsp_clkconfig.h"
    #include "./led/bsp_led.h"

     void Delay(__IO u32 nCount);

    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {
        // 程序来到main函数之前，启动文件：statup_stm32f4xx.s已经调用
        // SystemInit()函数把系统时钟初始化成168MHZ
        // SystemInit()在system_stm32f4xx.c中定义
        // 如果用户想修改系统时钟，可自行编写程序修改
        // 重新设置系统时钟，这时候可以选择使用HSE还是HSI

        // 使用HSE，配置系统时钟为168M
        HSE_SetSysClock(25, 336, 2, 7);

        //系统时钟超频到216M爽一下，最高是216M，别往死里整
        //HSE_SetSysClock(25, 432, 2, 9);

        // 使用HSI，配置系统时钟为168M
    //  HSI_SetSysClock(16, 336, 2, 7);

        // LED 端口初始化
        LED_GPIO_Config();

        // MCO GPIO  初始化
        MCO1_GPIO_Config();
        MCO2_GPIO_Config();

        // MCO1 输出PLLCLK
        HAL_RCC_MCOConfig(RCC_MCO1,RCC_MCO1SOURCE_PLLCLK, RCC_MCODIV_2);

        // MCO2 输出SYSCLK
        HAL_RCC_MCOConfig(RCC_MCO2,RCC_MCO2SOURCE_SYSCLK, RCC_MCODIV_2);

        while (1) {
            LED1( ON );      // 亮
            Delay(0x0FFFFF);
            LED1( OFF );      // 灭
            Delay(0x0FFFFF);
        }
    }

    void Delay(__IO uint32_t nCount)   //简单的延时函数
    {
        for (; nCount != 0; nCount--);
    }

在主函数中，可以调用HSE_SetSysClock()或者HSI_SetSysClock()这两个函数把系统时钟设置成各种常用的时钟，然后通过MCO引脚监控，或者通过LED闪烁的快慢体验不同的系统时钟对同一个软件延时函数的影响。

下载验证
^^^^^^^^

把编译好的程序下载到开发板，可以看到设置不同的系统时钟时，LED闪烁的快慢不一样。更精确的数据我们可以用示波器监控MCO引脚看到。
