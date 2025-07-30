#   DDR学习笔记

## 电源管理

### 整体架构

https://www.wowotech.net/pm_subsystem/pm_architecture.html

http://events17.linuxfoundation.org/sites/events/files/slides/Intro_Kernel_PM.pdf

在保证系统运转的基础上，尽量节省对能量的消耗。

电源管理（Power Management）在Linux Kernel中，是一个比较庞大的子系统，涉及到供电（Power Supply）、充电（Charger）、时钟（Clock）、频率（Frequency）、电压（Voltage）、睡眠/唤醒（Suspend/Resume）等方方面面

![image-20250730105111474](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730105111474.png)

这些组件（framework）是一个中间层的软件，提供软件开发的框架。

- 一是屏蔽具体的实现细节，固定对上的接口，这样可以方便上层软件的开发和维护；
- 二是尽可能抽象公共逻辑，并在Framework内实现，以提高重用性、减少开发量；
- 三是向下层提供一系列的回调函数（callback function），下层软件可能面对差别较大的现实，但只要填充这些回调函数，即可完成所有逻辑，减小了开发的难度。

> 组件介绍

- Power Supply，是一个供用户空间程序监控系统的供电状态（电池供电、USB供电、AC供电等等）的class。通俗的讲，它是一个Battery&Charger驱动的Framework
- **Clock Framework**，Clock驱动的Framework，用于统一管理系统的时钟资源
- Regulator Framework，Voltage/Current Regulator驱动的Framework。该驱动用于调节CPU等模块的电压和电流值
- Dynamic Tick/Clock Event，在传统的Linux Kernel中，系统Tick是**固定周期**（如10ms）的，因此**每隔一个Tick**，就会**产生一个Timer中断**。这会唤醒处于Idle或者Sleep状态的CPU，而**很多时候这种唤醒是没有意义**的。因此新的Kernel就提出了**Dynamic Tick**的概念，Tick不再是周期性的，而是根据系统中定时器的情况，不规律的产生，这样可以减少很多无用的Timer中断
- CPU Idle（空闲），用于控制CPU Idle状态的Framework
- **Generic PM**，传统意义上的Power Management，如Power Off、Suspend to RAM、Suspend to Disk、Hibernate等
- Runtime PM and Wakelock，运行时的Power Management，不再需要用户程序的干涉，由Kernel统一调度，实时的关闭或打开设备，以便在使用性能和省电性能之间找到最佳的平衡
  注3：Runtime PM是Linux Kernel亲生的运行时电源管理机制，Wakelock是由Android提出的机制。这两种机制的目的是一样的，因此只需要支持一种即可。另外，由于Wakelock机制路子太野了，饱受Linux社区的鄙视，因此我们不会对该机制进行太多的描述。
- **CPU Freq/Device Freq**，用于实现CPU以及Device频率调整的Framework
- **OPP**（Operating Performance Point，操作性能点），是指可以使SOCs或者Devices正常工作的电压和频率组合。内核提供这一个Layer，是为了在众多的电压和频率组合中，筛选出一些相对固定的组合，从而使事情变得更为简单一些
- **PM QOS**，所谓的PM QOS，是指系**统在指定的运行状态下（不同电压、频率，不同模式之间切换，等等）的工作质量**，包括latency（延迟）、timeout、throughput（吞吐量）三个参数，单位分别为us、us和kb/s。通过QOS参数，可以分析、改善系统的性能

### Generic PM

https://www.wowotech.net/pm_subsystem/generic_pm_architecture.html

指Linux系统中那些常规的电源管理手段，包括关机（Power off）、待机（Standby or Hibernate）、重启（Reboot）等。

这些手段是在嵌入式Linux普及之前的PC或者服务器时代使用的。在那个计算机科学的蛮荒时代，人类在摩尔定律的刺激下，孜孜追求的是计算机的计算能力、处理性能，因此并不特别关心Power消耗。

在这种背景下发展出来的Linux电源管理机制，都是粗放的、静态的、被动的。

Generic PM在传统的计算机操作系统中被广泛使用，因为那个时候对计算机的使用大多是**主动方式**。而对当前的移动互联来说，就非常不合时宜了，因为人们需要移动设备实时在线、**实时接收被动事件**（如来电），也就不可能主观地暂停使用（哪怕短短的一段时间）。这也就推动了Runtime PM的出现，

#### Generic PM的软件架构

![image-20250730144833320](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730144833320.png)

根据上面的描述可知，Generic PM主要处理关机、重启、冬眠（Hibernate）、睡眠（Sleep，在Kernel中也称作Suspend）。在内核中，大致可以分为三个软件层次：

- API Layer，用于向用户空间提供接口，其中关机和重启的接口形式是系统调用（在新的内核中，关机接口还有一种新方式，具体讲到的时候再说），Hibernate和Suspend的接口形式是sysfs。
- PM Core，位于kernel/power/目录下，主要处理和硬件无关的核心逻辑。
- PM Driver，分为两个部分，一是**体系结构无关的Driver**，提供Driver框架（Framework）。另一部分是具体的**体系结构相关的Driver**，这也是电源管理驱动开发需要涉及到的内容（图中红色边框的模块）。

#### reboot过程



![image-20250730150318850](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730150318850.png)

### PM接口

主要功能是：对下，定义Device PM相关的回调函数，让各个Driver实现；对上，实现统一的PM操作函数，供PM核心逻辑调用。

**对上统一调用，对下统一注册**

> device PM callbacks

通过 **统一的回调接口（PM callbacks）**，在 **恰当的时机**（如空闲、暂停、休眠）将设备 **同步切换至合理状态**（如关闭、低功耗、睡眠），以最小化系统功耗。

1. **旧版问题**：PM callbacks 直接散落在 `bus_type`、`device_driver` 等大型结构中，扩展性差，改动波及面广。
2. **新版方案**：通过 **`struct dev_pm_ops`** 统一封装所有回调，实现 **解耦**：
   - **抽象**：电源行为与设备模型分离，仅需包含此结构。
   - **扩展**：新增回调只需修改 `dev_pm_ops`，无需触碰上层结构。

**`从散落耦合到dev_pm_ops统一封装，内核电源管理实现优雅解耦`**

```c
struct dev_pm_ops {
	int (*prepare)(struct device *dev);
	void (*complete)(struct device *dev);
	int (*suspend)(struct device *dev);
	int (*resume)(struct device *dev);
	int (*freeze)(struct device *dev);
	int (*thaw)(struct device *dev);
	int (*poweroff)(struct device *dev);
	int (*restore)(struct device *dev);
	int (*suspend_late)(struct device *dev);
	int (*resume_early)(struct device *dev);
	int (*freeze_late)(struct device *dev);
	int (*thaw_early)(struct device *dev);
	int (*poweroff_late)(struct device *dev);
	int (*restore_early)(struct device *dev);
	int (*suspend_noirq)(struct device *dev);
	int (*resume_noirq)(struct device *dev);
	int (*freeze_noirq)(struct device *dev);
	int (*thaw_noirq)(struct device *dev);
	int (*poweroff_noirq)(struct device *dev);
	int (*restore_noirq)(struct device *dev);
	int (*runtime_suspend)(struct device *dev);
	int (*runtime_resume)(struct device *dev);
	int (*runtime_idle)(struct device *dev);
};
```

**PM Core**：在电源管理流程（如 suspend/resume）中，按 **明确的阶段顺序** 调用预定义的 callbacks（如 `prepare → suspend → suspend_late → suspend_noirq`）。

**`PM Core管流程，Driver管实现；回调是工具，场景才是老师`**

> device PM callbacks在设备模型中的体现

Linux设备模型中的很多数据结构，都会包含struct dev_pm_ops变量，比如bus_type、device_driver、class、device_type等结构中的pm指针。

> device PM callbacks的操作函数

内核在定义device PM callbacks数据结构的同时，为了方便使用该数据结构，也定义了大量的操作API，这些API分为两类。

- 通用的辅助性质的API，直接调用指定设备所绑定的driver的、pm指针的、相应的callback，如下

  ```c
     1: extern int pm_generic_prepare(struct device *dev);
     2: extern int pm_generic_suspend_late(struct device *dev);
     3: extern int pm_generic_suspend_noirq(struct device *dev);
     4: extern int pm_generic_suspend(struct device *dev);
     5: extern int pm_generic_resume_early(struct device *dev);
     6: extern int pm_generic_resume_noirq(struct device *dev);
     7: extern int pm_generic_resume(struct device *dev); 
     8: extern int pm_generic_freeze_noirq(struct device *dev);
     9: extern int pm_generic_freeze_late(struct device *dev);
    10: extern int pm_generic_freeze(struct device *dev);
    11: extern int pm_generic_thaw_noirq(struct device *dev);
    12: extern int pm_generic_thaw_early(struct device *dev);
    13: extern int pm_generic_thaw(struct device *dev);
    14: extern int pm_generic_restore_noirq(struct device *dev);
    15: extern int pm_generic_restore_early(struct device *dev);
    16: extern int pm_generic_restore(struct device *dev);
    17: extern int pm_generic_poweroff_noirq(struct device *dev);
    18: extern int pm_generic_poweroff_late(struct device *dev);
    19: extern int pm_generic_poweroff(struct device *dev); 
    20: extern void pm_generic_complete(struct device *dev);
  ```

  以pm_generic_prepare为例，就是查看dev->driver->pm->prepare接口是否存在，如果存在，直接调用并返回结果。

- 和整体电源管理行为相关的API，目的是将各个独立的电源管理行为组合起来，组成一个较为简单的功能，如下

  ```c
     1: #ifdef CONFIG_PM_SLEEP
     2: extern void device_pm_lock(void);
     3: extern void dpm_resume_start(pm_message_t state);
     4: extern void dpm_resume_end(pm_message_t state);
     5: extern void dpm_resume(pm_message_t state);
     6: extern void dpm_complete(pm_message_t state);
     7:  
     8: extern void device_pm_unlock(void);
     9: extern int dpm_suspend_end(pm_message_t state);
    10: extern int dpm_suspend_start(pm_message_t state);
    11: extern int dpm_suspend(pm_message_t state);
    12: extern int dpm_prepare(pm_message_t state);
    13:  
    14: extern void __suspend_report_result(const char *function, void *fn, int ret);
    15:  
    16: #define suspend_report_result(fn, ret)                                  \
    17:         do {                                                            \
    18:                 __suspend_report_result(__func__, fn, ret);             \
    19:         } while (0)
    20:  
    21: extern int device_pm_wait_for_dev(struct device *sub, struct device *dev);
    22: extern void dpm_for_each_dev(void *data, void (*fn)(struct device *, void *));
  ```

### Hibernate和Sleep功能

Hibernate和Sleep两个功能是Linux Generic PM的核心功能，它们的目的是类似的：暂停使用——>保存上下文——>关闭系统以节电········>恢复系统——>恢复上下文——>继续使用。

- **Hibernate**（冬眠）和**Sleep**（睡眠）

  是Linux电源管理在用户角度的抽象，是用户可以看到的实实在在的东西。它们的共同点，是保存系统运行的上下文后挂起（suspend）系统，并在系统恢复后接着运行，就像什么事情都没有发生一样。它们的不同点，是上下文保存的位置、系统恢复的触发方式以及具体的实现机制。

- **Suspend**

  有两个层次的含义。一是Hibernate和Sleep功能在底层实现上的统称，都是指挂起（Suspend）系统，根据上下文的保存位置，可以分为Suspend to Disk（STD，即Hibernate，上下文保存在硬盘/磁盘中）和Suspend to RAM（STR，为Sleep的一种，上下文保存在RAM中）；二是Sleep功能在代码级的实现，表现为“kernel/power/suspend.c”文件。

- **Standby**，是Sleep功能的一个特例，可以翻译为“**打盹**”。

  正常的Sleep（STR），会在处理完上下文后，由arch-dependent代码将CPU置为低功耗状态（通常为Sleep）。而现实中，根据对功耗和睡眠唤醒时间的不同需求，CPU可能会提供多种低功耗状态，如除Sleep之外，会提供Standby状态，该状态下，CPU处于浅睡眠模式，有任何的风吹草动，就会立即醒来。

- **Wakeup**

  睡眠时，为了缩短Wakeup时间，并不会关闭所有的供电，另外，为了较好的用户体验，通常会保留某些重要设备的供电（如键盘），那样**这些设备就可以唤醒系统**。这些刻意保留下来的、可以唤醒系统的设备，统称为**唤醒源**（Wakeup source）。而Wakeup source的选择，则是PM设计工作（特别是Sleep、Standby等功能）的重点。

> 软件架构

![image-20250730160149566](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730160149566.png)

> 用户空间接口

- **/sys/power/state**

  state是sysfs中一个文件，为Generic PM的核心接口，在“kernel/power/main.c”中实现，用于将系统置于指定的Power State（供电模式，如Hibernate、Sleep、Standby等）。不同的电源管理功能，在底层的实现，就是在不同Power State之间切换。

  在内核中，有两种类型的Power State，一种是Hibernate相关的，名称为“disk”，除“disk”之外，内核在"kernel/power/suspend.c"中通过数组的形式定义了另外3个state，如下：

  ```c
  const char *const pm_states[PM_SUSPEND_MAX] = {
      [PM_SUSPEND_FREEZE]  = "freeze",
      [PM_SUSPEND_STANDBY] = "standby", 	// s1
      [PM_SUSPEND_MEM]     = "mem",		// s3
  };
  ```

  写入特定的Power State字符串，将会把系统置为该模式。

- **/sys/power/pm_trace**

  PM Trace用于提供电源管理过程中的Trace记录，由“CONFIG_PM_TRACE”宏定义（kernel/power/Kconfig）控制是否编译进内核，并由“/sys/power/pm_trace”文件在运行时控制是否使能该功能。

- **/sys/power/pm_test**

  PM test用于对电源管理功能的测试，由“CONFIG_PM_DEBUG”宏定义（kernel/power/Kconfig）控制是否编译进内核。其核心思想是：

  - 将电源管理过程按照先后顺序，划分为多个步骤，如core、platform、devices等。这些步骤称作PM Test Level。
  - 系统通过一个全局变量（pm_test_level），保存系统当前的PM Test Level。该变量的值可以通过”/sys/power/pm_test“文件获取及修改。
  - 在每一个电源管理步骤结束后，插入PM test代码，该代码以当前执行步骤为参数，会判断当前的PM Test Level和执行步骤是否一致，如果一致，则说明该步骤执行成功。出于Test考量，执行成功后，系统会打印Test信息，并在等待一段时间后，退出PM过程。
  - 开发人员可以通过修改全局的Test Level，有目的测试所关心的步骤是否执行成功。

- **/sys/power/wakeup_count**

  该接口只和Sleep功能有关，因此由“CONFIG_PM_SLEEP”宏定义（kernel/power/Kconfig）控制。它的存在，是为了解决Sleep和Wakeup之间的同步问题。唤醒系统就是唤醒CPU，而唤醒CPU的唯一途径，就是Wakeup source产生中断（内核称作Wakeup event）。

  - 系统处于sleep状态时，产生了Wakeup event。此时应该直接唤醒系统。这一点没有问题。
  - 系统在进入sleep的过程中，产生了Wakeup event。此时应该放弃进入sleep。

- **/sys/power/disk**

  该接口是STD特有的。用于设置或获取STD的类型。当前内核支持的STD类型包括：

  ```c
  static const char * const hibernation_modes[] = {
      [HIBERNATION_PLATFORM]  = "platform",
      [HIBERNATION_SHUTDOWN]  = "shutdown",
      [HIBERNATION_REBOOT]    = "reboot",
  #ifdef CONFIG_SUSPEND
      [HIBERNATION_SUSPEND]   = "suspend",
  #endif
  };
  ```

  - platform，表示使用平台特有的机制，处理STD操作，如使用hibernation_ops等。
  - shutdown，通过关闭系统实现STD，内核会调用kernel_power_off接口。
  - reboot，通过重启系统实现STD，内核会调用kernel_restart接口。
  - suspend，利用STR功能，实现STD。该类型下，STD和STR底层的处理逻辑类似。

- **/sys/power/image_size**

- **/sys/power/reserverd_size**

- **/sys/power/resume**

- **debugfs/suspend_status**

- **/dev/snapshot**



### PM QOS framework

http://www.wowotech.net/pm_subsystem/pm_qos_overview.html

https://zhuanlan.zhihu.com/p/561000691

PM QoS：Power Management Quality of Service，电源管理服务质量。在电源管理（PM）的**省电收益**与**性能损耗**之间建立动态平衡机制，通过**量化**各模块的**QoS需求**（延迟容忍度、吞吐量要求等），确保PM策略不会过度损害关键操作的性能。

1. **本质矛盾**
   - **省电** ↔ **性能** 是天然对立目标（如CPU降频降低功耗但增加计算延迟）。
   - QoS框架充当“调解员”，避免PM优化时“误伤”高优先级任务。
2. **框架角色**
   - **不测量QoS**，而是**响应需求**：
     驱动、进程等可声明自己的最低性能要求（如“USB传输期间CPU不得低于1GHz”），框架据此约束PM行为。
3. **设计哲学**
   - **需求驱动**：由**各模块主动提出**QoS约束，而非PM被动猜测。
   - **动态协商**：不同场景下自动调整PM策略（如视频播放时禁用睡眠，空闲时激进省电）。

> **`PM QoS是以需求驱动的性能-功耗平衡器，让省电不“越界”`**

#### 工作原理

![image-20250730172839918](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730172839918.png)

- requestors提出对QoS的constraint。常见的requestor包括应用进程、GPU device、net device、flash device等等，它们基于自身的实际特性，需要系统的QoS满足一定的条件，才能正常工作。

- pm qos core负责汇整、整理这些constraint，并根据实际情况，计算出它们的极值（最大或者最小）。
- requestee在需要的时候，从pm qos core处获取constraint的极值，并确保自身的行为，可以满足这些constraints。一般情况下，requestee都是电源管理有关的service，包括[cpuidle](http://www.wowotech.net/tag/cpuidle)、[runtime pm](http://www.wowotech.net/tag/rpm)、[pm domain](http://www.wowotech.net/pm_subsystem/pm_domain_overview.html)等等。

实际上，Linux kernel使用“QoS dependencies”的概念，分别用“Dependents on a QoS value”和“Watchers of QoS value”表述这两个实体（可以理解为**QoS 值的依赖方**和**QoS 值的监视方**），具体可参考kernel/power/qos.c和drivers/base/power/qos.c的头文件。

> 约束分类

- **系统级约束**

  | **约束类型**  | **内核宏定义**              | **应用场景**             |
  | :------------ | :-------------------------- | :----------------------- |
  | CPU & DMA延迟 | `PM_QOS_CPU_DMA_LATENCY`    | 音视频同步、实时任务调度 |
  | 网络延迟      | `PM_QOS_NETWORK_LATENCY`    | 在线游戏、VoIP通话       |
  | 网络吞吐量    | `PM_QOS_NETWORK_THROUGHPUT` | 大文件下载、视频流传输   |
  | 内存带宽      | `PM_QOS_MEMORY_BANDWIDTH`   | GPU渲染、高性能计算      |

- **设备级约束**

  | **约束类型**   | **内核宏定义**             | **应用场景**                   |
  | :------------- | :------------------------- | :----------------------------- |
  | Resume延迟     | `PM_QOS_RESUME_LATENCY`    | 快速唤醒的SSD设备              |
  | Active状态延迟 | `PM_QOS_ACTIVE_LATENCY`    | 触摸屏响应延迟优化             |
  | QoS标志位      | `PM_QOS_FLAG_NO_POWER_OFF` | 关键外设（如安全芯片）禁止断电 |

![image-20250730190524946](C:\Users\w50050217\AppData\Roaming\Typora\typora-user-images\image-20250730190524946.png)

- 需求方：各service、各driver。他们根据自己的功能需求，提出系统或某些功能的QOS约束，比如cpu&dma latency。

- 框架层：PM QOS framework，包含PM QOS classes、per device PM QOS。

  - 向需求方提供request的add、modify、remove等接口，用于管理QoS requests。对需求方的约束进行分类，计算出极值，比如cpu&dma latency不小于某个值。

  - 向执行方提供request value的查询接口。
  - PM QoS classes framework位于kernel/power/qos.c中，负责系统级别的PM QoS管理，通过misc设备（/dev/cpu_dma_latency），向用户空间程序提供PM QoS的request、modify、remove功能，以便满足各service对PM QoS的需求。
  - per-device PM QoS framework位于drivers/base/power/qos.c中，负责per-device的PM QoS管理。

- 执行方：power management的机制，比如cpuidle、cpu dvfs等。需要满足由框架层根据需求方提供的约束计算的极值，才能执行相应的低功耗机制。

![img](https://picx.zhimg.com/v2-a9f21d142afbbd1204691685cea66833_1440w.jpg)

#### 工作流程

> 初始化流程

![img](https://pic2.zhimg.com/v2-c813709f89d686da5b4791e331ef4eaf_r.jpg)

> **PM QoS class framework的函数接口**

PM QoS class framework提供的API有两类：

- 以函数调用的形式，为kernel space的driver、service等提供的。

- 以misc设备的形式，为用户空间进程提供的。

  > **misc 设备（Miscellaneous Device）** 是一类特殊的字符设备，用于简化**非标准硬件**或**功能单一的外设**的驱动开发。
  >
  > 设计初衷是为那些**不足以单独分类**（如不属于块设备、网络设备等明确类别）、但又需要用户空间接口的设备提供统一的注册和管理机制
  >
  > 这种设计模式**将内核模块的功能抽象为设备节点**，用户进程通过文件I/O（如`open()`、`ioctl()`）与之交互，是一种典型的 **“内核-用户空间通信”** 方案。

1. **向kernel其它driver提供的，用于提出PM QoS需求的API**

   ```c
   void pm_qos_add_request(struct pm_qos_request *req, int pm_qos_class, s32 value);
   void pm_qos_update_request(structpm_qos_request *req, s32 new_value);
   void pm_qos_update_request_timeout(struct pm_qos_request *req, unsigned long timeout_us);
   void pm_qos_remove_request(struct pm_qos_request *req);
   int pm_qos_request_active(struct pm_qos_request *req);
   ```

   ![img](https://pic3.zhimg.com/v2-f0362cfee1ae0c68e671bbc4e0129fa8_1440w.jpg)

2. **向kernel PM有关的模块提供的，用于获取、跟踪指定PM QoS需求的API**

   每当有新的QoS请求时，**framework会根据该QoS class的含义，计算出满足所有请求的一个极值**（如最大值、最小值等等）。该值可以通过pm_qos_request接口获得。例如cpuidle framework在选择C state时，会通过该接口获得系统对CPU&DMA latency的需求，并保证从C state返回时的延迟小于该value。

   另外，如果**某个实体在意某一个class的QoS value变化**，可以**通过pm_qos_add_notifier接口添加一个notifier**，这样当value变化时，framework便会通过notifier的回调函数，通知该实体。

   ![img](https://pic4.zhimg.com/v2-a181d826125108e50c98ba400f0a59f9_1440w.jpg)

3. **向per-device PM QoS framework提供，底层的PM QoS操作API**

   QoS class和per-device PM QoS都是基于底层的pm qos constraint封装而来的。对QoS class的使用者而言，可以不用关心这些底层细节。对per-device PM QoS framework而言，则需要利用它们实现自身的功能。

   ```c
   int pm_qos_update_target(struct pm_qos_constraints *c, struct plist_node *node, qos_req_action action, int value);
   
   bool pm_qos_update_flags(struct pm_qos_flags *pqf, qos_flags_request *req, qos_req_action action, s32 val);
   
   s32 pm_qos_read_value(struct pm_qos_constraints *c);
   ```

4. **向用户空间service提供的，用于提出QoS需求的API**

   /dev/cpu_dma_latency

   打开文件，将会使用默认值，向PM QoS framework添加一个QoS请求。

   关闭文件，会移除相应的请求。

   写入value，更改请求的值。

   读取文件，将会获取QoS的极值。

> **per-device PM QoS framework的函数接口**

针对指定设备的QoS framework

1. **向kernel其它driver提供的，用于提出per-device PM QoS需求的API**

   ```c
   int dev_pm_qos_add_request(struct device *dev, struct dev_pm_qos_request *req, enum dev_pm_qos_req_type type, s32 value);
   
   int dev_pm_qos_update_request(struct dev_pm_qos_request *req, s32 new_value);
   
   int dev_pm_qos_remove_request(struct dev_pm_qos_request *req);
   ```

   ![img](https://pic3.zhimg.com/v2-e454cd551b9baa53b99bfc3c46f25e6e_r.jpg)

2. **向kernel PM有关的模块（例如power domain）提供的，用于获取、跟踪指定PM QoS需求的API**

   ```c
   enum pm_qos_flags_status dev_pm_qos_flags(struct device *dev, s32 mask);
   
   s32 dev_pm_qos_read_value(struct device *dev);
   
   int dev_pm_qos_add_notifier(struct device *dev, struct notifier_block *notifier);
   
   int dev_pm_qos_remove_notifier(struct device *dev,struct notifier_block *notifier);
   ```

   ![img](https://pic2.zhimg.com/v2-f37a4767a4cd86430bf8e1f27d22c467_1440w.jpg)

3. **向用户空间service提供的，用于提出per-device PM QoS需求的API**

   通过sysfs文件，kernel允许用户空间程序对某个设备提出QoS需求，这些sysfs文件位于各个设备的sysf目录下，默认情况下，PM QoS framework不会创建这些文件，需要具体设备驱动调用dev_pm_qos_expose_*系列接口，主动创建。

### 补充点

#### ACPI

ACPI（Advanced Configuration and Power Interface，高级配置与电源接口）是一套由 Intel、Microsoft 和 Toshiba 共同制定的**硬件电源管理与配置规范**，现已成为现代计算机（x86/ARM 等架构）电源管理的核心标准。

> **ACPI 的核心目标**

- **统一硬件抽象**：提供操作系统与硬件固件（BIOS/UEFI）之间的标准化接口，取代传统的 BIOS 电源管理方式。
- **动态电源管理**：支持系统状态的智能切换（如睡眠、休眠、性能调节）。
- **设备配置**：通过结构化表（ACPI Tables）描述硬件拓扑，替代传统的 PnP（即插即用）机制。

> **ACPI 的关键组件**

**(1) ACPI 表（ACPI Tables）**

- **DSDT（Differentiated System Description Table）**：核心硬件描述表，包含设备信息和电源控制方法（AML 代码）。
- **FADT（Fixed ACPI Description Table）**：定义固定寄存器（如电源按钮、睡眠控制）。
- **SSDT（Secondary System Description Table）**：补充设备描述，支持模块化扩展。
- **MADT（Multiple APIC Description Table）**：多处理器配置信息（如 CPU 核心、中断控制器）。

**(2) ACPI 状态（Power States）**

- **全局状态（Gx）**：
  - **G0（Working）**：正常运行。
  - **G1（Sleeping）**：细分睡眠子状态（S0ix, S1-S4）。
  - **G2（Soft Off）**：完全关机（保留少量供电，如 USB 唤醒）。
  - **G3（Mechanical Off）**：彻底断电。
- **设备状态（Dx）**：
  - **D0（Full On）** 到 **D3（Off）**，表示设备功耗级别。

**(3) ACPI 方法（Control Methods）**

- 用 **AML（ACPI Machine Language）** 编写的逻辑，由操作系统在运行时解释执行。例如：
  - `_PS0`：打开设备电源。
  - `_PRW`：定义设备唤醒能力。
