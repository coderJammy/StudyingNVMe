#NVM Express 协议

#文档说明

本文可以看做是对[NVME1.2版协议][1]的研读笔记，部分段落是协议原文的直接翻译。一些数据格式则是直接以C语言结构体的形式呈现。大部分示例代码从Linux和QEMU复制而来。


#1 NVM Express协议是什么？
NVM(Non-Volatile Memory)即非易失存储器。NVM Express是一套主机与存储设备之间的访问接口（寄存器级的接口）。典型应用如挂在PCIE总线下的SSD设备。

NVM Express协议描述了如何通过寄存器接口访问NVM子系统，并定义了一组访问NVM子系统的标准命令集。

##1.1 

##1.2 (字母，词组，缩写等)定义

	- Admin Queue ： 管理队列。指ID为0的提交队列(Submission Queue)和完成队列(Completion Queue)。或称为Amdin Submission Queue和Amdin Completion Queue。主机通过此队列发送Admin命令和接收命令状态。

	- arbitration burst : 提交队列一次可提交的最大命令个数。


#2 控制器寄存器(Controller Registers)

Controller Registers是一段存储器空间，可以接次序访问或按变量位宽访问（要具有这两特点，有些计算机架构需定义此空间为非缓存的空间）。主机须用本机位宽或32BIT对齐访问，否则会引发不确定的行为。不支持一次访问两个或多个寄存器。未定义的寄存器或寄存器中未定义的位读为0。

##2.1 寄存器定义

	0x0000 +--------------------------+
	       |    BAR                   |
	       |                          |
	0x0038 +--------------------------+
	       |    CMBLOC                |
	0x003C +--------------------------+
	       |    CMBSZ                 |
	0x0040 +--------------------------+
	       |    Reserved              |
	0x0F00 +--------------------------+
	       |    Command Set Specific  |
	0x1000 +--------------------------+  ---
	       |    SQ0 HDBL              |    \ Doorbell
		   |                          |    / Stride
	       +--------------------------+  ---
	       |    CQ0 TDBL              |
		   |                          |
	       +--------------------------+ 
		   ...               ...
	       +--------------------------+ 
	       |    SQn HDBL              |
		   |                          |
	       +--------------------------+
	       |    CQn TDBL              |
		   |                          |
	       +--------------------------+

其中BAR的定义如下：

~~~{.c}

	struct nvme_bar {
		__u64			cap;	/* Controller Capabilities */
		__u32			vs;		/* Version */
		__u32			intms;	/* Interrupt Mask Set */
		__u32			intmc;	/* Interrupt Mask Clear */
		__u32			cc;		/* Controller Configuration */
		__u32			rsvd1;	/* Reserved */
		__u32			csts;	/* Controller Status */
		__u32			rsvd2;	/* Reserved */
		__u32			aqa;	/* Admin Queue Attributes */
		__u64			asq;	/* Admin SQ Base Address */
		__u64			acq;	/* Admin CQ Base Address */
	};
~~~

###2.1.1 CAP(Contoller Capbilities)
CAP寄存器定义了控制器基本的功能。如：

	- MPSMAX : 所支持最大的Host Memory Page Size.(4K的倍数)
	- MPSMIN : 所支持最小的Host Memory Page Size.(4K的倍数)
	- CSS    : 所支持的命令集（目前只支持NVM Command Set）。
	- NSSRS  : 标志位。为1则支持NVM子系统复位。
	- DSTRD  : Doorbell Stride. Doorbell寄存器之间对齐位宽（4字节的倍数）。
	- TO     : 主机等待csts.rdy位跳变的最长时间。
	- AMS    : 仲裁机制。（Round Robin and Vendor Specific）
	- CQR    : 标志位。为1则表示IO SQ/CQ队列存储空间在物理上要连续。
	- MQES   : 队列最大长度。（0'based value；指Entries个数）

###2.1.2 VS (Version)
支持的协议版本。目前有效的有：1.0，1.1，1.2。对应的值分别为：

	- 0x00010000
	- 0x00010100
	- 0x00010200

###2.1.3 INTMS (Interrupt Mask Set)
中断屏蔽使能寄存器。


###2.1.4 INTCS (Interrupt Mask Clear)
中断屏蔽清除寄存器。

INTMS和INTCS在下列中断模式下用作中断屏蔽使能和清除(bit significant)

	- pin-base interrupt
	- single message MSI
	- multiple message MSI

在MSI-X中断模式下时则使用interrupt mask table.此时主机不应访问INTMS和INTCS寄存器。

###2.1.5 CC (Controller Configuration)
控制器配置寄存器。可用的配置项有：

	- IOCQES	 : IO CQ Entry Size.(作用值为2^IOCQES) 
	- IOSQES	 : IO SQ Entry Size.(作用值为2^IOSQES) 
	- SHN    : 关闭消息通知类型。
	- AMS    : 使用中的仲裁机制。
	- MPS    : Memory Page Size。（4K的倍数，最小4K最大128M）
	- CSS    : 使用中的IO命令集。目前只支持NVM Command Set。
	- EN     : 写1使能控制器。


AMS，MPS，CSS域应在使能EN位之前配置。

###2.1.6 CSTS (Controller Status)
控制器状态寄存器。状态位有：

	- PP     : 标志位。为1时表示控制器暂停命令处理转而处理其它事件。
	- NSSRO  : 标示位。为1时表示有NVM子系统复位事件。
	- SHST   : 关闭状态。（00b=未关闭，01b=关闭处理中 10b=关闭处理完成）
	- CFS    : 标志位。为1时表示有致命错误发生。
	- RDY    : 标志位。为1时表示准备好接收并处理命令。

CC.EN清零时CSTS.RDY也应清零。

###2.1.7 NSR (NVM Subsystem Reset)
向此寄存器写0x4E564D65("NVMe")会引发一个NVM子系统复位。写其它值时无效果，读为0。

###2.1.8 AQA (Admin Queue Attribute)
定义Admin SQ和Admin CQ的属性。属性列表如下：

	- ACQS  : Admin CQ Size;即命令队列的深度(Entry)。最小为1最大为4095(0's based)。
	- ASQS  : Admin SQ Size;见ACQS

###2.1.9 ASQ (Admin SQ Base Address)
Admin SQ的64位物理地址。这个地址应Memory Page对齐（见2.1.5 CC.MPS）。

###2.1.10 ACQ (Admin CQ Base Address)
Admin CQ的64位物理地址。这个地址应Memory Page对齐（见2.1.5 CC.MPS）。

###2.1.11 CMBLOC (Controller Memory Buffer Location)
定义了控制器Memory Buffer的地址。为0时则无效。

###2.1.12 CMBSZ (Controller Memory Buffer Size)
定义了控制器Memory Buffer的大小。如不支持则此寄存器值为0。
关于Controller Memory Buffer，请参考另外的一文()。

###2.1.13 SQnTDBL (SQ n Tail Doorbell)
###2.1.14 CQnHDBL (CQ n Head Doorbell)
SQnTDBL和CQnHDBL都是低16位有效的Entry Pointer。

#3 内存结构 Memory Structures
内存结构位于主机内存当中，若控制器支持Controller Memory Buffer，内存结构也可能位于控制器内存当中。

##3.1 队列
下面的图展示了一个空的队列和一个满的队列。队列的大小(深度)指队列的Entry个数。对于I/O CQ和I/O SQ，队列深度最大为65536，对Admin CQ和Admin SQ队列深度最大为4096。

	                +---+<--Address Queue Base     +---+<--Address Queue Base     
	                | E |                          | O |
	                +---+                          +---+
	                +---+                          +---+
	   Head-------->| E |<--Tail(Producer)	       | E |<--Tail(Producer)
	  (Consumer)    +---+                          +---+
	                +---+                          +---+
	                | E |             Head-------->| O |
	                +---+            (Consumer)    +---+
	                 ...                            ...
	                +---+                          +---+
	                | E |                          | O |
	                +---+                          +---+
	                +---+                          +---+
	                | E |                          | O |
	                +---+                          +---+
				An Empty Queue                  A Full Queue

	       +---+                         +---+
	       | E |   An Empty Entry;       | O |   An Occupied Entry;
	       +---+                         +---+
										
##3.2 SQ Entry-Command Format
每一条SQ Entry即为一条命令，由64 bytes组成。格式如下：

~~~{.c}

	struct nvme_common_command {
		__u8			opcode;
		__u8			flags;
		__u16			command_id;
		__le32			nsid;
		__le32			cdw2[2];
		__le64			metadata;
		__le64			prp1;
		__le64			prp2;
		__le32			cdw10[6];
	};
~~~

其中：
	
	- opcode    : 操作命令字。见NVM命令集定义。
	- flags     : 标志位，PRP或SGL使用，FUSE指示等。
	- command_id: 命令ID，主机会分配一个唯一的ID，命令完成后在CQ中会对应ID的Entry。
	- nsid      : 命令要访问的Namespace ID.
	- cdw2[2]   : reserved
	- metadata  : metadata 指针
	- prp1      : data 指针。格式为PRP或者SGL。
	- prp2      : data 指针。格式为PRP或者SGL。
	- cdw10[6]  : 命令相关。不同的命令具有不同的意义。

PRP和SGL用于指定数据，数据源或者数据缓冲存储区。

##3.3 数据指针PRP (Physical Region Page)
PRP Entry是指向物理内存页(Physical Memory Page)的指针,关使用称为分散聚合(scatter/gather)的机制来组织数据。这些数据物理上可以不连续(*分散*)，使用分散聚合机制使其在逻辑上看起来是一块连续的内存空间(*聚合*)。
PRP Entry 格式如下：

	|63                               n+1|n                    0|
	+------------------------------------+------------------+-+-+
	|         Page Base Address          |      Offset      |0|0|
	+------------------------------------+------------------+-+-+

PRP Entry长度为8bytes，由页基址(Page Base Address)和页内偏移量(Offset)组成，最低两位固定为0。n的大小取决于页的大小，Page Size为4K时，n=11，8k则为12。下列两种情况下，Offset必须为0：

	1 不是第一个PRP Entry
	2 指向的是一个PRP List

PRP表(PRP List)是指一组PRP Entries，这些PRP Entries位于同一个页内连续空间内。一条命令中，一个PRP Entry无法描述所有的数据时，PRP Entry便指向一个PRP List，PRP List内有更多有PRP Entries来描述数据。所一个PRP List也无法描述所有数据，则PRP list在页内的最后一PRP Entry指向另外一个PRP List，以此类推。

PRP表中的PRP Entry中，Offset应为0。到底会使用多少PRP Entries，是由命令的参数和页的大小共同决定。


##3.4 数据指针SGL



##3.5 Metadata Region
Metadata可理解为关于数据的数据。Metadata即要作为逻辑数据扩展，也可单独传输。

##3.6 CQ Entry
CQ Entry 长度为16bytes，格式如下:

	    |31                    |15                  0|
	    +--------------------------------------------+
	DW0 |        Command Specific                    |
	    +--------------------------------------------+
	DW1 |        Reserved                            |
	    +----------------------+---------------------+
	DW2 |    SQ Identifier     |  SQ Head Pointer    |
	    +--------------------+-+---------------------+
	DW3 |    Status Field    |P|  Command Identifier |
	    +--------------------+-+---------------------+

C语言抽象如下(Copied from Linux Kernel source code)：

~~~{.c}

	struct nvme_completion {
		__le32	result;		/* Used by admin commands to return data */
		__u32	rsvd;
		__le16	sq_head;	/* how much of this queue may be reclaimed */
		__le16	sq_id;		/* submission queue that generated this entry */
		__u16	command_id;	/* of the command which completed */
		__le16	status;		/* did the command fail, and if so, why? */
};
~~~

#5 I/O Command


#6 NVM控制器架构
*本章多由翻译标准协议得来。详见 Page 161(7 Controller Architecture) of NVM_Express_1_2_Gold_20141103.pdf *

##6.1 概述
主机通过事先创建好的SQ(提交队列)向控制器发送命令，命令提交后SQ Tail Doorbell全向前推，此时controller会收到有新命令事件通知。controller的固件会保存一个Doorbell的值，与新Doorbell值比较可计算得到新命令的个数。

控制器读取SQ中新命令并处理。除FUSED操作外，所有新命令的处理顺序没有限制，而数据的传输顺序也可能跟命令顺序不一样。

命令本身没有优先级，SQ有优先级。所以命令的优先级取决于提交到哪一个SQ。

命令处理完成后，controller把CQ Entry写入对应的CQ，并中断通知主机有命令完成(中断机制详见中断)。
CQ Entry中有SQ ID和command ID用于唯一标识已经完成的命令。

主机发送IO命令前，首先会创建SQ和CQ。CQ要先于对应的SQ创建。

##6.2 命令的提交和完成机制
###6.2.1 命令处理流程
处理流程如下：

	1. 主机在SQ中合适的内存地址填写命令字。
	2. 更新SQ的Tail Doorbell（用于通知controller新命令已发出）。
	3. Controller从SQ内存中读取一条或多条命令（会涉及SQ优先级，读取顺序等）。
	4. Controller继续执行未完成的命令，各条命令的完成时间是不确定的，可乱序。
	5. 一条命令完成后，一条CQ Entry会写入对应的CQ。
	6. 若中断可用，Controller会向主机发送一次中断，通知一条命令完成。
	7. 主机读取并处理CQ中的CQ Entry。
	8. 处理完CQ Entry后，主机将CQ Head Doorbell向前推进，表示处理已经完成。一次可能处理多个CQ Entry。


###6.2.2 主机如何创建一条命令
主机创建命令前，首先检查对应的SQ是否为FULL，当有可用的SQ Entry可用时，继续以下步骤:

	a 


##6.3 复位


##6.4 队列管理
**创建和初始化**

I/O SQ 和 I/O CQ要经过创建和初始化后方可使用，步骤如下：

	1. 初始化Admin SQ和Admin CQ(设置寄存器AQA，ASQ，和ACQ).
	2. 通过Admin SQ发送Set Features(with NVME_FEAT_NUM_QUEUES)命令，得到设备支持的QUEUE个数。
	3. 读寄存器获得队列大小（CAP.MQES）和队列内存是否需要物理连续(CAP.CQR)等属性。
	4. 根据NVME_FEAT_NUM_QUEUES，CAP.MQES，CAP.CQR等参数或限制创建CQ并向设备发送创建CQ命令。
	5. 类似4创建SQ。


此时I/O SQ 和 I/O CQ创建和初始化完毕，主机可通过其发送命令。

**队列协作**

假设有一对Admin Queue和多对I/O Queue，Admin SQ和Admin CQ用于管理整个控制器，某一对IO CQ和IO SQ则用于读写操作。IO Queue的个数可能会根据CPU核心数或线程数来分配。

**队列终止**

要放弃执行一大批命令，协议会建议删除对应的IO SQ并重新建立。

##6.5 中断

##6.6 控制器初始化和关闭
**初始化**

上电后Controller需经过初始化才能发送第一条命令，初始化流程参考如下：

	1. 以PCI或PCIE总线为例，首先配置总线寄存器。如电源管理选项，中断类型配置等。
	2. 主机等待设备复位完成。(寄存器CSTS.RDY变为0)
	3. 配置Admin Queue
	4. 配置设备寄存器。（仲裁机制，页大小，命令集等配置）
	5. 使能控制器（set CC.EN=1）。
	6. 等待控制器Ready.(Ready to process commands)
	7. 发Identify命令获得设备信息。
	8. 主机计算IO QUEUE个数，配置中断（MSI，MSI-X）。
	9. 创建IO QUEUE。先CQ后SQ。
	10. 开启异步事件通知功能（若需要）。

**Linux NVMe驱动如何初始化**

~~~{.c}

	static int nvme_dev_start(struct nvme_dev *dev)
	{
		int result;
		...
		result = nvme_dev_map(dev);
		...
		result = nvme_configure_admin_queue(dev);
		...
		result = nvme_setup_io_queues(dev);
		...
		return result;
	}
	static int nvme_dev_map(struct nvme_dev *dev)
	{
		...
		struct pci_dev *pdev = dev->pci_dev;
	
		if (pci_enable_device_mem(pdev))
			return result;
		...
		dev->bar = ioremap(pci_resource_start(pdev, 0), 8192);
		...
		cap = readq(&dev->bar->cap);
		dev->q_depth = min_t(int, NVME_CAP_MQES(cap) + 1, NVME_Q_DEPTH);
		dev->db_stride = 1 << NVME_CAP_STRIDE(cap);
		dev->dbs = ((void __iomem *)dev->bar) + 4096;
		...
	}
	static void nvme_cpu_workfn(struct work_struct *work)
	{
		struct nvme_dev *dev = container_of(work, struct nvme_dev, cpu_work);
		if (dev->initialized)
			nvme_assign_io_queues(dev);
	}
	static int nvme_probe(struct pci_dev *pdev, const struct pci_device_id *id)
	{
		int result = -ENOMEM;
		struct nvme_dev *dev;
	
		dev = kzalloc(sizeof(*dev), GFP_KERNEL);
		...
		dev->entry = kcalloc(num_possible_cpus(), sizeof(*dev->entry),
									GFP_KERNEL);
		...
		dev->queues = kcalloc(num_possible_cpus() + 1, sizeof(void *),
									GFP_KERNEL);
		...
		dev->io_queue = alloc_percpu(unsigned short);
		...
		INIT_WORK(&dev->cpu_work, nvme_cpu_workfn);
		...
		result = nvme_dev_start(dev);
		...
		result = nvme_dev_add(dev);
		...
		return result;
	}
~~~

nvme_probe在驱动加载成功时会调用，分析以上代码可知

	1. 驱动会为每一个CPU分配一对IO Queue。每对IO Queue有一个中断回调函数。
	2. 获得设备的BAR空间，并将地址映射到IO空间上来。
	3. 创建Admin Queue。检查CPU个数与设备最多支持的IO Queues，释放可能多分配的IO Queue。
	4. IO Queue在每个CPU工作的时候各自创建，初始化时只设定工作函数。
	5. 添加设备（Identify并添加盘符或设备等），见4.1.4。

**关闭**
断电或关闭事件即将发生等正常关闭时，主机应遵循一个关闭流程：

	1. 不再提交新命令，允许未完成的命令执行完
	2. 主机删除所有I/O SQ
	3. 删除所有IO CQ
	4. 设置CC.SHN为01b,通知设备关闭操作。设备完成关闭相关的操作后设置寄存器CSTS.SHST为10b

对于异常关闭：

	1. 	不再提交新命令
	2. 	设置CC.SHN为01b,通知设备关闭操作。设备完成关闭相关的操作后设置寄存器CSTS.SHST为10b

关闭后的控制器要重新接收命令，首先要复位控制器，并且重走一遍初始化流程。

