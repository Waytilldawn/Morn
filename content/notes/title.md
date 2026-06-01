---
title: 2 Commodity Hardware Today
date: '2026-05-30 17:05:05'
updated: '2026-06-01 11:17:19'
enableToc: true
enableBackLinks: true
---

# 2 Commodity Hardware Today

[cpumemory.pdf](assets/cpumemory-20260529105852-9gvth07.pdf)

### 2 Commodity Hardware Today

#### 2.0 Intro

​![image](assets/image-20260529093216-u9t8fuz.png "标准南桥-北桥架构")​

**特点**:

* 一个CPU与其它CPU的交互的信息都通过FSB(Front Side Bus)
* CPU与RAM的信息交互需要通过北桥
* Memory controller 被包含在 北桥 里，它的实现决定使用的RAM的类型

  *Remark: 内存控制器负责管理 CPU 与内存之间的数据读写时序、地址映射、刷新等操作，访问延迟对系统性能敏感，所以设计放在北桥(链接CPU与RAM)之间。*
* CPU和南桥连接着的设备交互需要北桥进行路由

  *Remark:南桥设备虽然与CPU多了一道北桥，但是由于连接着南桥的一般是低速设备，所以影响不大*

‍

**问题和瓶颈**:

* 早期，连接在北桥和南桥上的设备都需要通过CPU,造成不必要开销

  *Remark:后续一些设备支持DMA(Direct Memory Access),在北桥控制下可以不经过CPU, 但是这也增加了北桥的负荷，传送到北桥的信息必须经过处理后才能被路由到对应方向，这个过程占用性能。*

* 北桥到RAM的总线

  *Remark:早期多个RAM共用一条总线，后续总线数目增加，构成多通道，一条通道连着对应的一组RAM，同时使用交错的方式来进行读写，让相邻的逻辑地址落在不同的通道上，实现并行处理*

‍

​![image](assets/image-20260529103629-i66rpho.png "带有外部控制器的北桥架构")​

‍

**优势**

* 更多的Memory bus 带来更多的带宽，支持更多Memory
* 多个Memory Controller支持并发访问，有巨大并发潜力

‍

**问题与瓶颈**

* 北桥需要解析请求地址，决定内存通道，如果解析速度不够快，会变成串行(受限于北桥逻辑运行频率)
* 北桥与单个MC之间线路带宽瓶颈

‍

​![image](assets/image-20260529104855-32bz4lm.png)​

更进一步，将MC集成到CPU内部(Non-Uniform Memory Architecture)

**特点**

* 处理器数目与Memory bank数目一致

‍

**问题与瓶颈**

* CPU1到相邻的和要经过一个才能到达的CPU速率不同(NUMA factors)
* CPU(node)之间的连接花费高昂，且可能导致高NUMA factor

‍

#### 2.1 RAM Types

##### 2.1.1 Static RAM

​![image](assets/image-20260529111124-brskju9.png)​

**特点**

* 只要V_dd电源维持，则保持稳态
* WL高电平时准许读取，读到$BL,\overline{BL}$
* 写入先调$BL,\overline{BL}$，再拉WL

* 当WL拉高时几乎迅速读取

‍

##### 2.1.2 Dynamic RAM

​![image](assets/image-20260529111707-u3f912f.png)​

**特点**

* 电容C保存状态
* Transistor M控制读取权限
* 读/写都需要AL接入，同时调整DL让C充电/放电，这需要**一定时间**
* 单元尺寸比SRAM小
* 无需单独供电

‍

**问题与瓶颈**

* 电容要足够小

  *Remark:因为这一点，所以就算是数值上非常微小的电荷流失，也会让其无法识别，因此需要刷新(论文那时频率为 every 64ms)*
* 刷新时候读取不可用

  *Remark:刷新时候是将原有数据重新写回，视作一次完整读操作,挤占大量工作时间*
* 电容电量如此微小以至于需要放大器
* 每次读取都会让电容电量少一些

  > This means every read operation must be followed by an operation to recharge the capacitor.
  >

  *Remark:这里使用刚才的放大器的输出来进行补充，但是不可避免消耗更多能量和时间*

* 充放电不是即时的

  ​![image](assets/image-20260529113838-gh7qaqh.png)​

  *Remark1:放大器受到的信号并不是矩形波，因此得考虑什么时候这个单元的输出可用*

  *Remark2:读取必须充放电充分这一特点严重影响DRAM的速度*

‍

##### 2.1.3 DRAM Access

‍

**内存映射的过程**

程序通过虚拟地址选择内存地址->

处理器转换为物理地址->

Memeory controller选择指向的RAM芯片

‍

**地址的编码**

当地址线增加的时候，Demultiplexer 的尺寸会大幅度增加，同时控制更多脉冲同时到达并不容易。

‍

看一个具体的例子

‍

​![image](assets/image-20260529120151-qexaiuk.png)​

*Remark:* 这里我们管理16个单元，如果只使用1个4-16译码器，那么需要的门数量要比2个2-4要多，因此拆分出来更好。

**问题**

* 对于读来说，需要等待放电、放大的延迟后才可以稳定出现在data bus上。
* 对于写(对指定cell的充电)来说，也同样需要一定时间延迟才可以完成写入

* 一个RAM芯片可能需要30根线，这些线要接到Memory controller上，但是可能有多个RAM芯片

‍

‍

##### 2.1.4 Conclusions

* SRAM需要的元件更多，带来扩展困难

* 内存单元需要单独选取

  *Remark:对于一个阵列译码器每次只能选取一个*

* 地址线越多，Memory controller就要越复杂
* 内存阵列的读/写有时间延迟

‍

#### 2.2 DRAM Access Technical Details

‍

DDR:Double Data Rate DRAM

##### 2.2.0 引言

SDRAM(Synchronous DRAM),需要依据时间工作，MC提供时钟，MC的频率决定Front Side Bus("the memory controller interface usded by the DRAM chips")的速度

*Remark:内存控制器规定时钟频率，但实际运行仍然受DRAM和主板物理层支持限制.*

‍

##### 2.2.1 Read Access Protocol

​![image](assets/image-20260530115221-38evu9f.png "一次读取过程
")​

一次读取过程经历如下步骤:

1. MC放低$\overline{RAS}$
2. 经过一段延迟$t_{RCD}$后,指定的单元行可以在上升沿被读取，但是还需要$CL$时间($\overline{CAS}$拉低到信号出现在总线上\)准备好(这部分时间取决于MC质量，主板，DRAM模块)

‍

实际上我们往往每次读多个单元，可能是2, 4, 8 words.而为了做到这一点，我们并不需要每次都改变$\overline{RAS}$, 只需要重新选取$\overline{CAS}$, 于是读取连续的内存地址。效率得到提升。发送新的$\overline{CAS}$信号的速率只受到Command Rate of the RAM影响。

*Remark:这里Command Rate of the RAM指DRAM 模块接受新指令的时间周期。*

‍

##### 2.2.2 Precharge and Activation

在新的$\overline{RAS}$信号被发送之前，当前的行需要被停止，新的行需要被提前充电.

*Remark:原先的行信号会将某内容读到位线上，此时电压被放大后锁存，要访问新的一行时候需要让位线恢复到中间水平。*

‍

​![image](assets/image-20260530134042-ksca4w9.png)​

在这里，选取新的一行需要$t_{RP}$时间来预充电，这一阶段与数据传输部分重叠(蓝色区域,2个时钟周期), Precharge 还多一个周期。

图中自橙色区域开始有7个周期，而总线只负载其中2个周期。所以对于理论上6.4GB/s,800MHz的总线，实际消息传递效率是2/7,约1.8GB/s.

此外，值得一提的是，在花费$t_{RP}$时间进行预充电之前，还需要进行准备，这里假设$t_{RAS}$(Active to Precharge delay)用时8个时钟周期的话，在这样情况下，也就是切换行之前只在该行只读一个数据的话会出现必须等待的一个时钟周期($2(CL)+2(t_{RCD})+3(t_{RP}) = 7 < 8$)，导致利用效率的下降

*Remark1:这里可以把*​*$t_{RAS}$*理解为一个独立的计时器，一个下限约束，只有经过了这段时间后，才可以发出预充电命令。

*Remark2:对于Figure 2.9,在右侧还有未画出的部分，所以在理解*​*$t_{RAS}$*​*时候，只根据图可能带来误解。在图向右继续延伸时候，在RowAddr指令后的至少8个时钟周期才可以进入下一个*​*$t_{RP}$*阶段.

‍

##### 2.2.3 Recharging

每个DRAM单元每64ms刷新一次，每一行的刷新任务由Memory controller控制，被安排在任务队列当中。

DRAM模块会追踪最后刷新的行并且对于每个新请求会增加地址计数器。

于是，如果恰好要取出即将刷新的行中的一个字，很有可能不得不等待刷新。

‍

##### 2.2.4 Memory Types

SDR: Single Data Rate

DDR: Double Data Rate

​![image](assets/image-20260530162547-0azhnnj.png)​

DRAM 单元阵列的运行频率可以与其上的主线的数据频率一致。比如100MHz运行的单元，在主线上的传输速率就是100Mb/s.更高的频率同时意味着更高的能量消耗，同时维持系统稳定所需的电压也要更高。

​![image](assets/image-20260530162923-by55n7q.png)​

在这里，上升沿和下降沿都可以传递数据(also "double-pumped" bus).为了做到这一点，我们引入缓冲区。为了做到这一点，我们每次从DRAM Cell Array中读取两个位置的数据。并且对应到两个沿。

​![image](assets/image-20260530163757-xhgouee.png)​

这里我们让I/O Buffer以更高的速率运行。

*Remark:这里可以看出，DDR的改进思路之一是增加每次读取的信息数，并且让传输速度能够匹配上。*

​![image](assets/image-20260530164510-fh1f417.png)​

DDR3又翻了一番。当然在构建双通道上，DDR3有一些劣势，DDR3因为频率高，对信号完整性要求苛刻，无法做到多个模块同时挂靠在一条总线上。而在北桥设计下，一般只有2个内存通道。

‍

##### 2.2.5 Conclusions

处理器处理信息的速度实际上比总线传输数据的速度要快不少，一个例子是对于一个2.933GHZ的Inter 2核CPU,总线空白1个周期，CPU空白11个周期。

另外，DDR的快速读取，是建立在读取的数据在阵列中连续的前提下的，如果要跨行读取，那么需要发送新的$\overline{RAS}$信号，花费额外时间。
