DMA—直接存储区访问
------------------

本章参考资料：《STM32F4xx中文参考手册》DMA控制器章节。

学习本章时，配合《STM32F4xx中文参考手册》DMA控制器章节一起阅读，效果会更佳，特别是涉及到寄存器说明的部分。本章内容专业名称较多，内容丰富也较难理解，但非常有必要细读研究。

特别说明，本章内容是以STM32F4xx系列资源讲解。

DMA简介
~~~~~~~

DMA(Direct Memory Access,直接存储区访问)为实现数据高速在外设寄存器与存储器之间或者存储器与存储器之间传输提供了高效的方法。
之所以称之为高效，是因为DMA传输实现高速数据移动过程无需任何CPU操作控制。从硬件层次上来说，
DMA控制器是独立于Cortex-M4内核的，有点类似GPIO、USART外设一般，只是DMA的功能是可以快速移动内存数据。

STM32F4xx系列的DMA功能齐全，工作模式众多，适合不同编程环境要求。STM32F4xx系列的DMA支持外设到存储器传输、存储器到外设传输和存储器到存储器传输三种传输模式。这里的外设一般指外设的数据寄存器，比如ADC、SPI、I2C、DCMI等等外设的数据寄存器，存储器一般是指片内SRAM、外部存储器、片内Flash等等。

外设到存储器传输就是把外设数据寄存器内容转移到指定的内存空间。比如进行ADC采集时我们可以利用DMA传输把AD转换数据转移到我们定义的存储区中，这样对于多通道采集、采样频率高、连续输出数据的AD采集是非常高效的处理方法。

存储区到外设传输就是把特定存储区内容转移至外设的数据寄存器中，这种多用于外设的发送通信。

存储器到存储器传输就是把一个指定的存储区内容拷贝到另一个存储区空间。功能类似于C语言内存拷贝函数memcpy，利用DMA传输可以达到更高的传输效率，特别是DMA传输是不占用CPU的，可以节省很多CPU资源。

DMA功能框图
~~~~~~~~~~~

STM32F4xx系列的DMA可以实现外设寄存器与存储器之间或者存储器与存储器之间传输三种模式，这要得益于DMA控制器是采样AHB主总线的，
可以控制AHB总线矩阵来启动AHB事务。具体见 图21_1_：DMA控制器的框图。

.. image:: media/image2.png
   :align: center
   :alt: 图 21‑1 DMA框图
   :name: 图21_1

①外设通道选择
''''''''''''''''

STM32F4xx系列资源丰富，具有两个DMA控制器，同时外设繁多，为实现正常传输，DMA需要通道选择控制。每个DMA控制器具有8个数据流，
每个数据流对应8个外设请求。在实现DMA传输之前，DMA控制器会通过DMA数据流x配置寄存器DMA_SxCR(x为0~7，对应8个DMA数据流)
的CHSEL[2:0]位选择对应的通道作为该数据流的目标外设。

外设通道选择要解决的主要问题是决定哪一个外设作为该数据流的源地址或者目标地址。

DMA请求映射情况参考表 21-1和表 21-2。

.. image:: media/image3.png
   :align: center
   :alt: 图 21‑2 DMA1各个通道的请求映像
   :name: 图21_2

图 21‑2 DMA1各个通道的请求映像

.. image:: media/image4.png
   :align: center
   :alt: 图 21‑3 DMA2各个通道的请求映像
   :name: 图21_3

图 21‑3 DMA2各个通道的请求映像

每个外设请求都占用一个数据流通道，相同外设请求可以占用不同数据流通道。比如SPI3_RX请求，即SPI3数据发送请求，
占用DMA1的数据流0的通道0，因此当我们使用该请求时，我们需要在把DMA_S0CR寄存器的CHSEL[2:0]设置为“000”，
此时相同数据流的其他通道不被选择，处于不可用状态，比如此时不能使用数据流0的通道1即I2C1_RX请求等等。

查阅表 21-1可以发现SPI3_RX请求不仅仅在数据流0的通道0，同时数据流2的通道0也是SPI3_RX请求，
实际上其他外设基本上都有两个对应数据流通道，这两个数据流通道都是可选的，这样设计是尽可能提供多个数据流同时使用情况选择。

②仲裁器
''''''''''

一个DMA控制器对应8个数据流，数据流包含要传输数据的源地址、目标地址、数据等等信息。如果我们需要同时使用同一个DMA控制器(DMA1或DMA2)多个外设请求时，那必然需要同时使用多个数据流，那究竟哪一个数据流具有优先传输的权利呢？这就需要仲裁器来管理判断了。

仲裁器管理数据流方法分为两个阶段。第一阶段属于软件阶段，我们在配置数据流时可以通过寄存器设定它的优先级别，具体配置DMA_SxCR寄存器PL[1:0]位，可以设置为非常高、高、中和低四个级别。第二阶段属于硬件阶段，如果两个或以上数据流软件设置优先级一样，则他们优先级取决于数据流编号，编号越低越具有优先权，比如数据流2优先级高于数据流3。

③FIFO
''''''''''''''

每个数据流都独立拥有四级32位FIFO(先进先出存储器缓冲区)。DMA传输具有FIFO模式和直接模式。

直接模式在每个外设请求都立即启动对存储器传输。在直接模式下，如果DMA配置为存储器到外设传输那DMA会见一个数据存放在FIFO内，如果外设启动DMA传输请求就可以马上将数据传输过去。

FIFO用于在源数据传输到目标地址之前临时存放这些数据。可以通过DMA数据流xFIFO控制寄存器DMA_SxFCR的FTH[1:0]位来控制FIFO的阈值，分别为1/4、1/2、3/4和满。如果数据存储量达到阈值级别时，FIFO内容将传输到目标中。

FIFO对于要求源地址和目标地址数据宽度不同时非常有用，比如源数据是源源不断的字节数据，而目标地址要求输出字宽度的数据，即在实现数据传输时同时把原来4个8位字节的数据拼凑成一个32位字数据。此时使用FIFO功能先把数据缓存起来，分别根据需要输出数据。

FIFO另外一个作用使用于突发(burst)传输。

④存储器端口、⑤外设端口
''''''''''''''''''''''''''''
DMA控制器实现双AHB主接口，更好利用总线矩阵和并行传输。DMA控制器通过存储器端口和外设端口与存储器和外设进行数据传输，
关系见 图21_4_。DMA控制器的功能是快速转移内存数据，需要一个连接至源数据地址的端口和一个连接至目标地址的端口。

DMA2(DMA控制器2)的存储器端口和外设端口都是连接到AHB总线矩阵，可以使用AHB总线矩阵功能。DMA2存储器和外设端口可以访问相关的内存地址，包括有内部Flash、内部SRAM、AHB1外设、AHB2外设、APB2外设和外部存储器空间。

DMA1的存储区端口相比DMA2的要减少AHB2外设的访问权，同时DMA1外设端口是没有连接至总线矩阵的，只有连接到APB1外设，所以DMA1不能实现存储器到存储器传输。

.. image:: media/image5.png
   :align: center
   :alt: 图21_4
   :name: 图21_4

DMA数据配置
~~~~~~~~~~~

DMA工作模式多样，具有多种可能工作模式，具体可能配置见 图21_5_。

.. image:: media/image6.png
   :align: center
   :alt: 图21_5
   :name: 图21_5

DMA传输模式
''''''''''''''''

DMA2支持全部三种传输模式，而DMA1只有外设到存储器和存储器到外设两种模式。模式选择可以通过DMA_SxCR寄存器的DIR[1:0]位控制，进而将DMA_SxCR寄存器的EN位置1就可以使能DMA传输。

在DMA_SxCR寄存器的PSIZE[1:0]和MSIZE[1:0]位分别指定外设和存储器数据宽度大小，可以指定为字节(8位)、半字(16位)和字(32位)，我们可以根据实际情况设置。直接模式要求外设和存储器数据宽度大小一样，实际上在这种模式下DMA数据流直接使用PSIZE，MSIZE不被使用。

源地址和目标地址
''''''''''''''''

DMA数据流x外设地址DMA_SxPAR(x为0~7)寄存器用来指定外设地址，它是一个32位数据有效寄存器。
DMA数据流x存储器0地址DMA_SxM0AR(x为0~7) 寄存器和DMA数据流x存储器1地址DMA_SxM1AR(x为0~7)
寄存器用来存放存储器地址，其中DMA_SxM1AR只用于双缓冲模式，DMA_SxM0AR和DMA_SxM1AR都是32位数据有效的。

当选择外设到存储器模式时，即设置DMA_SxCR寄存器的DIR[1:0] 位为“00”，DMA_SxPAR寄存器为外设地址，
也是传输的源地址，DMA_SxM0AR寄存器为存储器地址，也是传输的目标地址。对于存储器到存储器传输模式，
即设置DIR[1:0] 位为“10”时，采用与外设到存储器模式相同配置。而对于存储器到外设，即设置DIR[1:0]位为“01”时，
DMA_SxM0AR寄存器作为为源地址，DMA_SxPAR寄存器作为目标地址。

流控制器
''''''''''''''''
流控制器主要涉及到一个控制DMA传输停止问题。DMA传输在DMA_SxCR寄存器的EN位被置1后就进入准备传输状态，
如果有外设请求DMA传输就可以进行数据传输。很多情况下，我们明确知道传输数据的数目，比如要传1000个或者2000个数据，
这样我们就可以在传输之前设置DMA_SxNDTR寄存器为要传输数目值，DMA控制器在传输完这么多数目数据后就可以控制DMA停止传输。

DMA数据流x数据项数DMA_SxNDTR(x为0~7)寄存器用来记录当前仍需要传输数目，它是一个16位数据有效寄存器，即最大值为65535，
这个值在程序设计是非常有用也是需要注意的地方。我们在编程时一般都会明确指定一个传输数量，
在完成一次数目传输后DMA_SxNDTR计数值就会自减，当达到零时就说明传输完成。

如果某些情况下在传输之前我们无法确定数据的数目，那DMA就无法自动控制传输停止了，
此时需要外设通过硬件通信向DMA控制器发送停止传输信号。这里有一个大前提就是外设必须是可以发出这个停止传输信号，
只有SDIO才有这个功能，其他外设不具备此功能。

循环模式
''''''''''''''''

循环模式相对应于一次模式。一次模式就是传输一次就停止传输，下一次传输需要手动控制，而循环模式在传输一次后会自动按照相同配置重新传输，周而复始直至被控制停止或传输发生错误。

通过DMA_SxCR寄存器的CIRC位可以使能循环模式。

传输类型
''''''''''''''''
DMA传输类型有单次(Single)传输和突发(Burst)传输。突发传输就是用非常短时间结合非常高数据信号率传输数据，相对正常传输速度，突发传输就是在传输阶段把速度瞬间提高，实现高速传输，在数据传输完成后恢复正常速度，有点类似达到数据块“秒传”效果。为达到这个效果突发传输过程要占用AHB总线，保证要求每个数据项在传输过程不被分割，这样一次性把数据全部传输完才释放AHB总线；而单次传输时必须通过AHB的总线仲裁多次控制才传输完成。

单次和突发传输数据使用具体情况参考表 214。其中PBURST[1:0]和MBURST[1:0]位是位于DMA_SxCR寄存器中的，
用于分别设置外设和存储器不同节拍数的突发传输，对应为单次传输、4个节拍增量传输、8个节拍增量传输和16个节拍增量传输。
PINC位和MINC位是寄存器DMA_SxCR寄存器的第9和第10位，如果位被置1则在每次数据传输后数据地址指针自动递增，
其增量由PSIZE和MSIZE值决定，比如，设置PSIZE为半字大小，那么下一次传输地址将是前一次地址递增2。

.. image:: media/image7.png
   :align: center
   :alt: 图21_6
   :name: 图21_6

突发传输与FIFO密切相关，突发传输需要结合FIFO使用，具体要求FIFO阈值一定要是内存突发传输数据量的整数倍。
FIFO阈值选择和存储器突发大小必须配合使用，具体参考上表 图21_6_。

.. image:: media/image8.png
   :align: center
   :alt: 图21_7
   :name: 图21_7

直接模式
''''''''''''''''
默认情况下，DMA工作在直接模式，不使能FIFO阈值级别。

直接模式在每个外设请求都立即启动对存储器传输的单次传输。直接模式要求源地址和目标地址的数据宽度必须一致，所以只有PSIZE控制，而MSIZE值被忽略。突发传输是基于FIFO的所以直接模式不被支持。另外直接模式不能用于存储器到存储器传输。

在直接模式下，如果DMA配置为存储器到外设传输那DMA会见一个数据存放在FIFO内，如果外设启动DMA传输请求就可以马上将数据传输过去。

双缓冲模式
''''''''''''''''
设置DMA_SxCR寄存器的DBM位为1可启动双缓冲传输模式，并自动激活循环模式。双缓冲不应用与存储器到存储器的传输。双缓冲模式下，两个存储器地址指针都有效，即DMA_SxM1AR寄存器将被激活使用。开始传输使用DMA_SxM0AR寄存器的地址指针所对应的存储区，当这个存储区数据传输完DMA控制器会自动切换至DMA_SxM1AR寄存器的地址指针所对应的另一块存储区，如果这一块也传输完成就再切换至DMA_SxM0AR寄存器的地址指针所对应的存储区，这样循环调用。

当其中一个存储区传输完成时都会把传输完成中断标志TCIF位置1，如果我们使能了DMA_SxCR寄存器的传输完成中断，则可以产生中断信号，这个对我们编程非常有用。另外一个非常有用的信息是DMA_SxCR寄存器的CT位，当DMA控制器是在访问使用DMA_SxM0AR时CT=0，此时CPU不能访问DMA_SxM0AR，但可以向DMA_SxM1AR填充或者读取数据；当DMA控制器是在访问使用DMA_SxM1AR时CT=1，此时CPU不能访问DMA_SxM1AR，但可以向DMA_SxM0AR填充或者读取数据。另外在未使能DMA数据流传输时，可以直接写CT位，改变开始传输的目标存储区。

双缓冲模式应用在需要解码程序的地方是非常有效的。比如MP3格式音频解码播放，MP3是被压缩的文件格式，我们需要特定的解码库程序来解码文件才能得到可以播放的PCM信号，解码需要一定的实际，按照常规方法是读取一段原始数据到缓冲区，然后对缓冲区内容进行解码，解码后才输出到音频播放电路，这种流程对CPU运算速度要求高，很容易出现播放不流畅现象。如果我们使用DMA双缓冲模式传输数据就可以非常好的解决这个问题，达到解码和输出音频数据到音频电路同步进行的效果。

DMA中断
''''''''''''''''
每个DMA数据流可以在发送以下事件时产生中断：

1)达到半传输：DMA数据传输达到一半时HTIF标志位被置1，如果使能HTIE中断控制位将产生达到半传输中断；

2)传输完成：DMA数据传输完成时TCIF标志位被置1，如果使能TCIE中断控制位将产生传输完成中断；

3)传输错误：DMA访问总线发生错误或者在双缓冲模式下试图访问“受限”存储器地址寄存器时TEIF标志位被置1，如果使能TEIE中断控制位将产生传输错误中断；

4)FIFO错误：发生FIFO下溢或者上溢时FEIF标志位被置1，如果使能FEIE中断控制位将产生FIFO错误中断；

5)直接模式错误：在外设到存储器的直接模式下，因为存储器总线没得到授权，使得先前数据没有完成被传输到存储器空间上，此时DMEIF标志位被置1，如果使能DMEIE中断控制位将产生直接模式错误中断。

DMA初始化结构体详解
~~~~~~~~~~~~~~~~~~~

HAL函数对每个外设都建立了一个初始化结构体xxx_InitTypeDef(xxx为外设名称)，结构体成员用于设置外设工作参数，并由HAL库函数xxx_Init()调用这些设定参数进入设置外设相应的寄存器，达到配置外设工作环境的目的。

结构体xxx_InitTypeDef和库函数xxx_Init配合使用是HAL库精髓所在，理解了结构体xxx_InitTypeDef每个成员意义基本上就可以对该外设运用自如了。结构体xxx_InitTypeDef定义在stm32f4xx_hal_xxx.h(后面xxx为外设名称)文件中，库函数xxx_Init定义在stm32f4xx_hal_xxx.c文件中，编程时我们可以结合这两个文件内注释使用。

DMA_InitTypeDef初始化结构体
''''''''''''''''''''''''''''''''

.. code-block:: c

    typedef struct {
        uint32_t Channel;              //通道选择
        uint32_t Direction;            //传输方向
        uint32_t PeriphInc;            //外设递增
        uint32_t MemInc;               //存储器递增
        uint32_t PeriphDataAlignment;  //外设数据宽度
        uint32_t MemDataAlignment;     //存储器数据宽度
        uint32_t Mode;                 //模式选择
        uint32_t Priority;             //优先级
        uint32_t FIFOMode;             //FIFO模式
        uint32_t FIFOThreshold;        //FIFO阈值
        uint32_t MemBurst;             //存储器突发传输
        uint32_t PeriphBurst;          //外设突发传输
     } DMA_InitTypeDef;

1)Channel：DMA请求通道选择，可选通道0至通道7，每个外设对应固定的通道，具体设置值需要查表 图21_2_ 和表 图21_3_；
它设定DMA_SxCR寄存器的CHSEL[2:0]位的值。例如，我们使用模拟数字转换器ADC3规则采集4个输入通道的电压数据，
查表 图21_3_ 可知使用通道2。

2)Direction：传输方向选择，可选外设到存储器、存储器到外设以及存储器到存储器。它设定DMA_SxCR寄存器的DIR[1:0]位的值。ADC采集显然使用外设到存储器模式。

3)PeriphInc：如果配置为PeriphInc_Enable，使能外设地址自动递增功能，它设定DMA_SxCR寄存器的PINC位的值；一般外设都是只有一个数据寄存器，所以一般不会使能该位。ADC3的数据寄存器地址是固定并且只有一个所以不使能外设地址递增。

4)MemInc：如果配置为MemInc_Enable，使能存储器地址自动递增功能，它设定DMA_SxCR寄存器的MINC位的值；我们自定义的存储区一般都是存放多个数据的，所以使能存储器地址自动递增功能。我们之前已经定义了一个包含4个元素的数字用来存放数据，使能存储区地址递增功能，自动把每个通道数据存放到对应数组元素内。

5)PeriphDataAlignment：外设数据宽度，可选字节(8位)、半字(16位)和字(32位)，它设定DMA_SxCR寄存器的PSIZE[1:0]位的值。ADC数据寄存器只有低16位数据有效，使用半字数据宽度。

6)MemDataAlignment：存储器数据宽度，可选字节(8位)、半字(16位)和字(32位)，它设定DMA_SxCR寄存器的MSIZE[1:0]位的值。保存ADC转换数据也要使用半字数据宽度，这跟我们定义的数组是相对应的。

7)Mode：DMA传输模式选择，可选一次传输或者循环传输，它设定DMA_SxCR寄存器的CIRC位的值。我们希望ADC采集是持续循环进行的，所以使用循环传输模式。

8)Priority：软件设置数据流的优先级，有4个可选优先级分别为非常高、高、中和低，它设定DMA_SxCR寄存器的PL[1:0]位的值。DMA优先级只有在多个DMA数据流同时使用时才有意义，这里我们设置为非常高优先级就可以了。

9)FIFOMode：FIFO模式使能，如果设置为DMA_FIFOMode_Enable表示使能FIFO模式功能；它设定DMA_SxFCR寄存器的DMDIS位。ADC采集传输使用直接传输模式即可，不需要使用FIFO模式。

10)FIFOThreshold：FIFO阈值选择，可选4种状态分别为FIFO容量的1/4、1/2、3/4和满；
它设定DMA_SxFCR寄存器的FTH[1:0]位； DMA_FIFOMode设置为DMA_FIFOMode_Disable，
那DMA_FIFOThreshold值无效。ADC采集传输不使用FIFO模式，设置改值无效。

11)MemBurst：存储器突发模式选择，可选单次模式、4节拍的增量突发模式、8节拍的增量突发模式或16节拍的增量突发模式，它设定DMA_SxCR寄存器的MBURST[1:0]位的值。ADC采集传输是直接模式，要求使用单次模式。

12)PeriphBurst：外设突发模式选择，可选单次模式、4节拍的增量突发模式、8节拍的增量突发模式或16节拍的增量突发模式，它设定DMA_SxCR寄存器的PBURST[1:0]位的值。ADC采集传输是直接模式，要求使用单次模式。

DMA_HandleTypeDef初始化结构体
''''''''''''''''''''''''''''''''

.. code-block:: c

    typedef struct __DMA_HandleTypeDef {
        DMA_Stream_TypeDef         *Instance;   //注册基地址
        DMA_InitTypeDef            Init;        //DMA通信参数
        HAL_LockTypeDef            Lock;        //DMA锁定对象
        __IO HAL_DMA_StateTypeDef  State;       //DMA传输状态
        void                       *Parent;     //父类指针
        void (* XferCpltCallback)( struct __DMA_HandleTypeDef * hdma);
        //DMA传输完成回调函数
        void (* XferHalfCpltCallback)( struct __DMA_HandleTypeDef * hdma);
        //DMA传输完成一半回调函数
        void (* XferM1CpltCallback)( struct __DMA_HandleTypeDef * hdma);
        //Memory1 DMA传输完成回调函数
        void (* XferM1HalfCpltCallback)( struct __DMA_HandleTypeDef * hdma);
        //Memory1 DMA传输完成一半回调函数
        void (* XferErrorCallback)( struct __DMA_HandleTypeDef * hdma);
        //DMA传输错误回调函数
        void (* XferAbortCallback)( struct __DMA_HandleTypeDef * hdma);
        //DMA传输中止回调函数
        __IO uint32_t               ErrorCode;             //DMA错误码
        uint32_t                    StreamBaseAddress;     //DMA数据流基地址
        uint32_t                    StreamIndex;		//DMA数据流索引
    } DMA_HandleTypeDef;

1)*Instance： 指向DMA数据流基地址的指针，即指定使用哪个DMA数据流。可选数据流0至数据流7。
例如，我们使用模拟数字转换器ADC3规则采集4个输入通道的电压数据，
查表 图21_3_ 可知可以使用数据流0或者数据流1，这里支持两个数据流是为了避免多个通道使用时发生冲突，提供备选数据流可选。

2)Init：这里包含上面介绍DMA_InitTypeDef结构体的所有参数的初始化。

3)Lock：DMA锁定对象。DMA进程锁，通常都在DMA传输设置开始前锁上进程锁，设置完毕后释放进程锁。

4)State：DMA传输状态。它包含六种状态，1、复位状态，尚未初始化或者禁能。2、就绪状态，已经完成初始化，随时可以传输数据。3、传输忙，DMA传输进程正在进行。4、传输超时状态。5、传输错误状态。6、传输中止状态。

5)*Parent：父类指针。只要将该指针指向一些ADC、UART等外设的handle类，就等于完成了继承。

6)DMA传输过程中的回调函数。包括传输完成，传输完成一半，传输错误，传输中止回调函数。这些回调函数中可以加入用户的处理代码。

7)ErrorCode：DMA错误码，包含无错误：HAL_DMA_ERROR_NONE，传输错误HAL_DMA_ERROR_TE，FIFO错误HAL_DMA_ERROR_FE，直接模式错误：HAL_DMA_ERROR_DME，超时错误：HAL_DMA_ERROR_TIMEOUT，参数错误：HAL_DMA_ERROR_PARAM，没有回调函数正在执行退出请求错误：HAL_DMA_ERROR_NO_XFER，不支持模式错误：HAL_DMA_ERROR_NOT_SUPPORTED。

8)StreamBaseAddress：DMA数据流基地址，用来根据定义句柄计算数据流的基地址。

9)StreamIndex：DMA数据流索引，根据数据流的序号来确定数据流的偏移地址。

DMA存储器到存储器模式实验
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

DMA工作模式多样，具体如何使用需要配合实际传输条件具体分析。接下来我们通过两个实验详细讲解DMA不同模式下的使用配置，加深我们对DMA功能的理解。

DMA运行高效，使用方便，在很多测试实验都会用到，这里先详解存储器到存储器和存储器到外设这两种模式，其他功能模式在其他章节会有很多使用到的情况，也会有相关的分析。

存储器到存储器模式可以实现数据在两个内存的快速拷贝。我们先定义一个静态的源数据，然后使用DMA传输把源数据拷贝到目标地址上，最后对比源数据和目标地址的数据，看看是否传输准确。

硬件设计
''''''''''''''''''''''''''''''''

DMA存储器到存储器实验不需要其他硬件要求，只用到RGB彩色灯用于指示程序状态，关于RGB彩色灯电路可以参考GPIO章节。

软件设计
''''''''''''''''''''''''''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。这个实验代码比较简单，主要程序代码都在main.c文件中。

编程要点
=============

1) 使能DMA数据流时钟并复位初始化DMA数据流；

2) 配置DMA数据参数；

3) 使能DMA数据流，进行传输；

4) 等待传输完成，并对源数据和目标地址数据进行比较。

代码分析
========================

DMA宏定义及相关变量定义
+++++++++++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑1 DMA数据流和相关变量定义
   :name: 代码清单21_1

    /* 相关宏定义，使用存储器到存储器传输必须使用DMA2 */
    DMA_HandleTypeDef DMA_Handle;

    #define DMA_STREAM               DMA2_Stream0
    #define DMA_CHANNEL              DMA_CHANNEL_0
    #define DMA_STREAM_CLOCK()       __DMA2_CLK_ENABLE()

    #define BUFFER_SIZE              32

    /* 定义aSRC_Const_Buffer数组作为DMA传输数据源
    const关键字将aSRC_Const_Buffer数组变量定义为常量类型 */
    const uint32_t aSRC_Const_Buffer[BUFFER_SIZE]= {
        0x01020304,0x05060708,0x090A0B0C,0x0D0E0F10,
        0x11121314,0x15161718,0x191A1B1C,0x1D1E1F20,
        0x21222324,0x25262728,0x292A2B2C,0x2D2E2F30,
        0x31323334,0x35363738,0x393A3B3C,0x3D3E3F40,
        0x41424344,0x44564748,0x494A4B4C,0x4D4E4F50,
        0x51525345,0x55565758,0x595A5B5C,0x5D5E5F60,
        0x61626364,0x65666768,0x696A6B6C,0x6D6E6F70,
        0x71727374,0x75767778,0x797A7B7C,0x7D7E7F80
    };
    /* 定义DMA传输目标存储器 */
    uint32_t aDST_Buffer[BUFFER_SIZE];

使用宏定义设置外设配置方便程序修改和升级。

存储器到存储器传输必须使用DMA2，但对数据流编号以及通道选择就没有硬性要求，可以自由选择。

aSRC_Const_Buffer[BUFFER_SIZE]是定义用来存放源数据的，并且使用了const关键字修饰，即常量类型，使得变量是存储在内部flash空间上。

DMA数据配置
+++++++++++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑2 DMA传输参数配置
   :name: 代码清单21_2

    static void DMA_Config(void)
    {
        HAL_StatusTypeDef DMA_status = HAL_ERROR;
        /* 使能DMA时钟 */
        DMA_STREAM_CLOCK();

        DMA_Handle.Instance = DMA_STREAM;
        /* DMA数据流通道选择 */
        DMA_Handle.Init .Channel = DMA_CHANNEL;
        /* 存储器到存储器模式 */
        DMA_Handle.Init.Direction = DMA_MEMORY_TO_MEMORY;
        /* 使能自动递增功能 */
        DMA_Handle.Init.PeriphInc = DMA_PINC_ENABLE;
        /* 使能自动递增功能 */
        DMA_Handle.Init.MemInc = DMA_MINC_ENABLE;
        /* 源数据是字大小(32位) */
        DMA_Handle.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
        /* 目标数据也是字大小(32位) */
        DMA_Handle.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
        /* 一次传输模式，存储器到存储器模式不能使用循环传输 */
        DMA_Handle.Init.Mode = DMA_NORMAL;
        /* DMA数据流优先级为高 */
        DMA_Handle.Init.Priority = DMA_PRIORITY_HIGH;
        /* 禁用FIFO模式 */
        DMA_Handle.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        DMA_Handle.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
        /* 单次模式 */
        DMA_Handle.Init.MemBurst = DMA_MBURST_SINGLE;
        /* 单次模式 */
        DMA_Handle.Init.PeriphBurst = DMA_PBURST_SINGLE;
        /* 完成DMA数据流参数配置 */
        HAL_DMA_Init(&DMA_Handle);

        DMA_status = HAL_DMA_Start(&DMA_Handle,(uint32_t)aSRC_Const_Buffer,
                (uint32_t)aDST_Buffer,BUFFER_SIZE);
        /* 判断DMA状态 */
        if (DMA_status != HAL_OK) {
            /* DMA出错就让程序运行下面循环：RGB彩色灯闪烁 */
            while (1) {
                LED_RED;
                Delay(0xFFFFFF);
                LED_RGBOFF;
                Delay(0xFFFFFF);
            }
        }
    }

使用DMA_DMA_HandleTypeDef结构体定义一个DMA数据流初始化变量，这个结构体内容我们之前已经有详细讲解。

调用DMA_STREAM_CLOCK函数开启DMA数据流时钟，使用DMA控制器之前必须开启对应的时钟。

存储器到存储器模式通道选择没有具体规定，只能使用一次传输模式不能循环传输，最后我调用HAL_DMA_Init函数完成DMA数据流的初始化配置。

HAL_DMA_Start函数用于启动DMA数据流传输，源地址和目标地址使用之前定义的数组首地址，返回DMA传输状态。

如果DMA传输没有就绪就会闪烁RGB彩灯提示。

存储器数据对比
+++++++++++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑3 源数据与目标地址数据对比
   :name: 代码清单21_3

    uint8_t Buffercmp(const uint32_t* pBuffer,
                    uint32_t* pBuffer1, uint16_t BufferLength)
    {
        /* 数据长度递减 */
        while (BufferLength--) {
            /* 判断两个数据源是否对应相等 */
            if (*pBuffer != *pBuffer1) {
                /* 对应数据源不相等马上退出函数，并返回0 */
                return 0;
            }
            /* 递增两个数据源的地址指针 */
            pBuffer++;
            pBuffer1++;
        }
        /* 完成判断并且对应数据相对 */
        return 1;
    }

判断指定长度的两个数据源是否完全相等，如果完全相等返回1；只要其中一对数据不相等返回0。它需要三个形参，前两个是两个数据源的地址，第三个是要比较数据长度。

主函数
+++++++++++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑4 存储器到存储器模式主函数
   :name: 代码清单21_4

    int main(void)
    {
        /* 定义存放比较结果变量 */
        uint8_t TransferStatus;
        /* 系统时钟初始化成168 MHz */
        SystemClock_Config();
        /* LED 端口初始化 */
        LED_GPIO_Config();
        /* 设置RGB彩色灯为紫色 */
        LED_PURPLE;

        /* 简单延时函数 */
        Delay(0xFFFFFF);

        /* DMA传输配置 */
        DMA_Config();

        /* 等待DMA传输完成 */
        while (__HAL_DMA_GET_FLAG(&DMA_Handle,DMA_FLAG_TCIF0_4)==DISABLE) {

        }

        /* 比较源数据与传输后数据 */
        TransferStatus=Buffercmp(aSRC_Const_Buffer, aDST_Buffer, BUFFER_SIZE);

        /* 判断源数据与传输后数据比较结果*/
        if (TransferStatus==0) {
            /* 源数据与传输后数据不相等时RGB彩色灯显示红色 */
            LED_RED;
        } else {
            /* 源数据与传输后数据相等时RGB彩色灯显示蓝色 */
            LED_BLUE;
        }

        while (1) {
        }
    }

首先定义一个变量用来保存存储器数据比较结果。

SystemClock_Config函数初始化系统时钟。

RGB彩色灯用来指示程序进程，使用之前需要初始化它，LED_GPIO_Config定义在bsp_led.c文件中。开始设置RGB彩色灯为紫色，LED_PURPLE是定义在bsp_led.h文件的一个宏定义。

Delay函数只是一个简单的延时函数。

调用DMA_Config函数完成DMA数据流配置并启动DMA数据传输。

__HAL_DMA_GET_FLAG函数获取DMA数据流事件标志位的当前状态，这里获取DMA数据传输完成这个标志位，使用循环持续等待直到该标志位被置位，即DMA传输完成这个事件发生，然后退出循环，运行之后程序。

确定DMA传输完成之后就可以调用Buffercmp函数比较源数据与DMA传输后目标地址的数据是否一一对应。TransferStatus保存比较结果，如果为1表示两个数据源一一对应相等说明DMA传输成功；相反，如果为0表示两个数据源数据存在不等情况，说明DMA传输出错。

如果DMA传输成功设置RGB彩色灯为蓝色，如果DMA传输出错设置RGB彩色灯为红色。

下载验证
============

确保开发板供电正常，编译程序并下载。观察RGB彩色灯变化情况。正常情况下RGB彩色灯先为紫色，然后变成蓝色。如果DMA传输出错才会为红色。

DMA存储器到外设模式实验
~~~~~~~~~~~~~~~~~~~~~~~

DMA存储器到外设传输模式非常方便把存储器数据传输外设数据寄存器中，这在STM32芯片向其他目标主机，比如电脑、另外一块开发板或者功能芯片，发送数据是非常有用的。RS-232串口通信是我们常用开发板与PC端通信的方法。我们可以使用DMA传输把指定的存储器数据转移到USART数据寄存器内，并发送至PC端，在串口调试助手显示。

硬件设计
''''''''''''''

存储器到外设模式使用到USART1功能，具体电路设置参考USART章节，无需其他硬件设计。

软件设计
'''''''''''''

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。我们编写两个串口驱动文件bsp_usart_dma.c和bsp_usart_dma.h，有关串口和DMA的宏定义以及驱动函数都在里边。

.. _编程要点-1:

编程要点
===============

1) 配置USART通信功能；

2) 设置DMA为存储器到外设模式，设置数据流通道，指定USART数据寄存器为目标地址，循环发送模式；

3) 使能DMA数据流；

4) 使能USART的DMA发送请求。

5) DMA传输同时CPU可以运行其他任务。

代码分析
===============

USART和DMA宏定义
+++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑5 USART和DMA相关宏定义
   :name: 代码清单21_5

     //串口波特率

     #define DEBUG_USART_BAUDRATE 115200

     //引脚定义

     /*******************************************************/

     #define DEBUG_USART USART1

     #define DEBUG_USART_CLK_ENABLE() __USART1_CLK_ENABLE();

     #define DEBUG_USART_RX_GPIO_PORT GPIOA

     #define DEBUG_USART_RX_GPIO_CLK_ENABLE() __GPIOA_CLK_ENABLE()

     #define DEBUG_USART_RX_PIN GPIO_PIN_10

     #define DEBUG_USART_RX_AF GPIO_AF7_USART1

     #define DEBUG_USART_TX_GPIO_PORT GPIOA

     #define DEBUG_USART_TX_GPIO_CLK_ENABLE() __GPIOA_CLK_ENABLE()

     #define DEBUG_USART_TX_PIN GPIO_PIN_9

     #define DEBUG_USART_TX_AF GPIO_AF7_USART1

     #define DEBUG_USART_IRQHandler USART1_IRQHandler

     #define DEBUG_USART_IRQ USART1_IRQn

     /************************************************************/

     //DMA

     #define SENDBUFF_SIZE 10//发送的数据量

     #define DEBUG_USART_DMA_CLK_ENABLE() __DMA2_CLK_ENABLE()

     #define DEBUG_USART_DMA_CHANNEL DMA_CHANNEL_4

     #define DEBUG_USART_DMA_STREAM DMA2_Stream7

使用宏定义设置外设配置方便程序修改和升级。

USART部分设置与USART章节内容相同，可以参考USART章节内容理解。

查阅 图21_3_ 可知USART1对应DMA2的数据流7通道4。

串口DMA传输配置
+++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑6 USART1 发送请求DMA设置
   :name: 代码清单21_6

   void USART_DMA_Config(void)
   {
     /*开启DMA时钟*/
     DEBUG_USART_DMA_CLK_ENABLE();

     DMA_Handle.Instance = DEBUG_USART_DMA_STREAM;
     /*usart1 tx对应dma2，通道4，数据流7*/
     DMA_Handle.Init.Channel = DEBUG_USART_DMA_CHANNEL;
     /*方向：从内存到外设*/
     DMA_Handle.Init.Direction= DMA_MEMORY_TO_PERIPH;
     /*外设地址不增*/
     DMA_Handle.Init.PeriphInc = DMA_PINC_DISABLE;
     /*内存地址自增*/
     DMA_Handle.Init.MemInc = DMA_MINC_ENABLE;
     /*外设数据单位*/
     DMA_Handle.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
     /*内存数据单位 8bit*/
     DMA_Handle.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
     /*DMA模式：不断循环*/
     DMA_Handle.Init.Mode = DMA_CIRCULAR;
     /*优先级：中*/
     DMA_Handle.Init.Priority = DMA_PRIORITY_MEDIUM;
     /*禁用FIFO*/
     DMA_Handle.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
     DMA_Handle.Init.FIFOThreshold = DMA_FIFO_THRESHOLD_FULL;
     /*存储器突发传输 16个节拍*/
     DMA_Handle.Init.MemBurst = DMA_MBURST_SINGLE;
     /*外设突发传输 1个节拍*/
     DMA_Handle.Init.PeriphBurst = DMA_PBURST_SINGLE;
     /*配置DMA2的数据流7*/
     /* Deinitialize the stream for new transfer */
     HAL_DMA_DeInit(&DMA_Handle);
     /* Configure the DMA stream */
     HAL_DMA_Init(&DMA_Handle);

     /* Associate the DMA handle */
     __HAL_LINKDMA(&UartHandle, hdmatx, DMA_Handle);
    }


使用DMA_HandleTypeDef结构体定义一个DMA数据流初始化变量，这个结构体内容我们之前已经有详细讲解。

调用DEBUG_USART_DMA_CLK_ENABLE宏开启DMA数据流时钟，使用DMA控制器之前必须开启对应的时钟。

USART有固定的DMA通道，USART数据寄存器地址也是固定的，外设地址不可以使用自动递增，源数据使用我们自定义的数组空间，
存储器地址使用自动递增，采用循环发送模式，最后我调用HAL_DMA_DeInit函数复位到缺省配置状态，DMA_Init函数完成DMA数据流的初始化配置。

__HAL_LINKDMA函数用于链接DMA数据流及通道到串口外设通道上。

主函数
+++++++++++++++++++++

.. code-block:: c
   :caption: 代码清单 21‑7 存储器到外设模式主函数
   :name: 代码清单21_7

    int main(void)
    {
        uint16_t i;

        /* 系统时钟初始化成168 MHz */
        SystemClock_Config();
        /* 初始化USART */
        Debug_USART_Config();

        /* 配置使用DMA模式 */
        USART_DMA_Config();

        /* 配置RGB彩色灯 */
        LED_GPIO_Config();

        printf("\r\n USART1 DMA TX 测试 \r\n");

        /*填充将要发送的数据*/
        for (i=0; i<SENDBUFF_SIZE; i++) {
            SendBuff[i]  = 'A';

        }

        /*为演示DMA持续运行而CPU还能处理其它事情，持续使用DMA发送数据，量非常大，
        *长时间运行可能会导致电脑端串口调试助手会卡死，鼠标乱飞的情况，
        *或把DMA配置中的循环模式改为单次模式*/

        HAL_UART_Transmit_DMA (&UartHandle,(uint8_t *)SendBuff,SENDBUFF_SIZE);
        /* 此时CPU是空闲的，可以干其他的事情 */
        //例如同时控制LED
        while (1) {
            LED1_TOGGLE
            Delay(0xFFFFFF);
        }
    }

SystemClock_Config函数初始化系统时钟。

Debug_USART_Config函数定义在bsp_usart_dma.c中，它完成USART初始化配置，包括GPIO初始化，USART通信参数设置等等，具体可参考USART章节讲解。

USART_DMA_Config函数也是定义在bsp_usart_dma.c中，之前我们已经详细分析了。

LED_GPIO_Config函数定义在bsp_led.c中，它完成RGB彩色灯初始化配置，具体可参考GPIO章节讲解。

使用for循环填充源数据，SendBuff[SENDBUFF_SIZE]是一个全局无符号8位整数数组，是DMA传输的源数据。

HAL_UART_Transmit_DMA函数用于启动USART的DMA传输。只需要指定源数据地址及长度，运行该函数后USART的DMA发送传输就开始了，根据配置它会通过USART循环发送数据。

DMA传输过程是不占用CPU资源的，可以一边传输一次运行其他任务。

下载验证
===========

保证开发板相关硬件连接正确，用USB线连接开发板“USB TO UART”接口跟电脑，在电脑端打开串口调试助手，
把编译好的程序下载到开发板。程序运行后在串口调试助手可接收到大量的数据，同时开发板上RGB彩色灯不断闪烁。

这里要注意为演示DMA持续运行并且CPU还能处理其它事情，持续使用DMA发送数据，量非常大，长时间运行可能会导致电脑端串口调试助手会卡死，鼠标乱飞的情况，所以在测试时最好把串口调试助手的自动清除接收区数据功能勾选上或把DMA配置中的循环模式改为单次模式。
