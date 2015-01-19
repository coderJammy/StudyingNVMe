#NVM Express 协议

#文档说明

本文可以看做是对[NVME1.2版协议][1]的研读笔记，部分段落是协议原文的直接翻译。一些数据格式则是直接以C语言结构体的形式呈现。大部分示例代码从Linux和QEMU复制而来。


#1 NVM Express协议是什么？
NVM(Non-Volatile Memory)即非易失存储器。NVM Express是一套主机与存储设备之间的访问接口（寄存器级的接口）。典型应用如挂在PCIE总线下的SSD设备。

NVM Express协议描述了如何通过寄存器接口访问NVM子系统，并定义了一组访问NVM子系统的标准命令集。

##1.1 

##1.2 (字母，词组，缩写等)定义

	- Admin Queue ： 管理队列。指ID为0的提交队列(Submision Queue)和完成队列(Completion Queue)。或称为Amdin Submision Queue和Amdin Completion Queue。主机通过此队列发送Admin命令和接收命令状态。

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
	       |    Reseved               |
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

INTMS和INTCS在下列中断模式下用作中断屏蔽使能和清除（bit significant）。

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


#4 Admin Command

~~~{.c}

	/* Admin commands */
	enum nvme_admin_opcode {
		nvme_admin_delete_sq		= 0x00,
		nvme_admin_create_sq		= 0x01,
		nvme_admin_get_log_page		= 0x02,
		nvme_admin_delete_cq		= 0x04,
		nvme_admin_create_cq		= 0x05,
		nvme_admin_identify		= 0x06,
		nvme_admin_abort_cmd		= 0x08,
		nvme_admin_set_features		= 0x09,
		nvme_admin_get_features		= 0x0a,
		nvme_admin_async_event		= 0x0c,
		nvme_admin_activate_fw		= 0x10,
		nvme_admin_download_fw		= 0x11,
		nvme_admin_format_nvm		= 0x80,
		nvme_admin_security_send	= 0x81,
		nvme_admin_security_recv	= 0x82,
	};
~~~

##4.1 Identify
主机使用设备前，需首先获得设备的信息。Identify命令会从设备中读出4KB关于NVM子系统的数据。
Identify命令格式如下：

~~~{.c}

	struct nvme_identify {
		__u8			opcode;
		__u8			flags;
		__u16			command_id;
		__le32			nsid;
		__u64			rsvd2[2];
		__le64			prp1;
		__le64			prp2;
		__le32			cns;
		__u32			rsvd11[5];
	};
~~~

Identify命令可读取三类数据，由cns(Controller or Namespace Structure)值决定：

	- cns=0x00 : Controller Identify Data
	- cns=0x01 : Namespace Identify Data
	- cns=0x02 : Namespace List

###4.1.1 Controller Identify Struture

~~~{.c}
	
	struct nvme_id_ctrl {
		__le16			vid;		/* Vendor ID assigned by the PCI SIG */
		__le16			ssvid;		/* Vendor ID for subsystem */
		char			sn[20];		/* Serial Number as an ASCII string */
		char			mn[40];		/* Model Number */
		char			fr[8];		/* (Currently acttive) Firmware Revision*/
		__u8			rab;		/* Recommended Arbitration Burst */
		__u8			ieee[3];	/* OUI for the controller verdor */
		__u8			mic;		/* Multi-Path I/O and Namespace Sharing Capabilites */
		__u8			mdts;		/* Maxmum Data Transfer Size (2^mdts)*/
		__u16			cntlid;		/* Unique Controller ID*/
		__u32			ver;		/* (NVMe Spec) Version */
		__u8			rsvd84[172];
		__le16			oacs;		/* Optioan Asynchronous Command Supported */
		__u8			acl;		/* Abort Command Limit */
		__u8			aerl;		/* Asynchronous Event Request Limit */
		__u8			frmw;		/* Firmware Updates*/
		__u8			lpa;		/* Log Page Attribute */
		__u8			elpe;		/* Error Log Page Entries */
		__u8			npss;		/* Number of Power States Support */
		__u8			avscc;		/* Admin Vendor Specific Command Configuration */
		__u8			apsta;		/* Autonomous Power State Transition Attributes*/
		__le16			wctemp;		/* Warning Composite Temperature Threshold */
		__le16			cctemp;		/* Critical Composite Temperature Threshold*/
		__u8			rsvd270[242];
		__u8			sqes;		/* SQ Entry Size*/
		__u8			cqes;		/* CQ Entry Size*/
		__u8			rsvd514[2];
		__le32			nn;			/* Number of Namespace*/
		__le16			oncs;		/* Optional NVM Command Support */
		__le16			fuses;		/* Fused Operation Support*/
		__u8			fna;		/* Format NVM Attributes*/
		__u8			vwc;		/* Volatile Write Cache */
		__le16			awun;		/* Atomic Write Unit Normal*/
		__le16			awupf;		/* Atomic Write Unit Power Fail */
		__u8			nvscc;		/* NVM Vendor Specific Command Configuration */
		__u8			rsvd531;
		__le16			acwu;		/* Atomic Compare & Write Unit*/
		__u8			rsvd534[2];
		__le32			sgls;		/* SGL Support*/ 
		__u8			rsvd540[1508];
		struct nvme_id_power_state	psd[32];/* Power State Descriptors*/
		__u8			vs[1024];	/* Vendor Specific */
	};
~~~

###4.1.2 Namespace Identify Struture

~~~{.c}

	struct nvme_id_ns {
		__le64			nsze;		/* Namespace Size (in logical blocks)*/
		__le64			ncap;		/* Namespace Capacity */
		__le64			nuse;		/* Namespace Utilization*/
		__u8			nsfeat;		/* Namespace Features*/
		__u8			nlbaf;		/* Number of LBA Formats*/
		__u8			flbas;		/* Formatted LBA Size*/
		__u8			mc;			/* Metadata Capabilities*/
		__u8			dpc;		/* End-to-end Data Protection Capabilities*/
		__u8			dps;		/* End-to-end Data Protection Type Setting */
		__u8			nmic;		/* Multi-path I/O and Namespace Sharing Capabilities*/
		__u8			rescap;		/* Reservation Capabilities*/
		__u8			fpi;		/* Format Progress Indicator*/
		__u8			rsvd33;
		__le16			nawun;		/* Namespace Atomic Write Unit Normal*/
		__le16			nawupf;		/* Namespace Atomic Write Unit Power Fail*/
		__le16			nacwu;		/* Namespace Atomic Compare & Write Unit*/
		__u8			rsvd40[80];
		__u8			eui64[8];	/* IEEE Extended Unique Identifier*/
		struct nvme_lbaf	lbaf[16];/* list of LBA Format Support*/
		__u8			rsvd192[192];
		__u8			vs[3712];	/* Vendor Specific*/
	};
~~~

###4.1.3 Namespace List (spec 1.2)


###4.1.4 HOST驱动如何发送Identify

~~~{.c}

	int nvme_identify(struct nvme_dev *dev, unsigned nsid, unsigned cns,
								dma_addr_t dma_addr)
	{
		struct nvme_command c;
	
		memset(&c, 0, sizeof(c));
		c.identify.opcode = nvme_admin_identify;
		c.identify.nsid = cpu_to_le32(nsid);
		c.identify.prp1 = cpu_to_le64(dma_addr);
		c.identify.cns = cpu_to_le32(cns);
	
		return nvme_submit_admin_cmd(dev, &c, NULL);
	}

	static int nvme_dev_add(struct nvme_dev *dev)
	{
		...
		struct nvme_id_ctrl *ctrl;
		...
	
		res = nvme_identify(dev, 0, 1, dma_addr);
	
		...
	
		nn = le32_to_cpup(&ctrl->nn);
		...
		for (i = 1; i <= nn; i++) {
			res = nvme_identify(dev, i, 0, dma_addr);
	
			...
	
		}
	
		...
	
		list_for_each_entry(ns, &dev->namespaces, list)
			add_disk(ns->disk);
	
		...
	}
~~~

可见，HOST会首先读取Controller Identify信息，从而获得NN(Number of Namespaces)信息，然后依次读取Namespace Identify信息。


###4.1.5 NVM设备如何处理Identify命令
以QEMU为例:

~~~{.c}

	static uint16_t nvme_identify(NvmeCtrl *n, NvmeCmd *cmd)
	{
	    NvmeNamespace *ns;
	    NvmeIdentify *c = (NvmeIdentify *)cmd;
	    uint32_t cns  = le32_to_cpu(c->cns);
	    uint32_t nsid = le32_to_cpu(c->nsid);
	    uint64_t prp1 = le64_to_cpu(c->prp1);
	    uint64_t prp2 = le64_to_cpu(c->prp2);
	
	    if (cns) {
	        return nvme_dma_read_prp(n, (uint8_t *)&n->id_ctrl, sizeof(n->id_ctrl),
	            prp1, prp2);
	    }
	    if (nsid == 0 || nsid > n->num_namespaces) {
	        return NVME_INVALID_NSID | NVME_DNR;
	    }
	
	    ns = &n->namespaces[nsid - 1];
	    return nvme_dma_read_prp(n, (uint8_t *)&ns->id_ns, sizeof(ns->id_ns),
	        prp1, prp2);
	}
~~~

上列代码可知，NVM设备处理Identify流程如下：

	1. 收到命令，提取命令信息
	2. 如果是读Controller Identify Data，则读取数据并返回，否则继续3
	3. 检查命令是否有效，有效则返回Namespace Identify Data

关于Controller Identify Data和Namespace Identify Data则存于设备内存中，在设备启动或复位时进行初始化，外更详细代码参见QEMU源码。


##4.2 创建I/O完成队列(Create I/O Completion Queue Command)


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

	1 主机在SQ中合适的内存地址填写命令字。
	2 更新SQ的Tail Doorbell（用于通知controller新命令已发出）。
	3 Controller从SQ内存中读取一条或多条命令（会涉及SQ优先级，读取顺序等）。
	4 Controller继续执行未完成的命令，各条命令的完成时间是不确定的，可乱序。
	5 一条命令完成后，一条CQ Entry会写入对应的CQ。
	6 若中断可用，Controller会向主机发送一次中断，通知一条命令完成。
	7 主机读取并处理CQ中的CQ Entry。
	8 处理完CQ Entry后，主机将CQ Head Doorbell向前推进，表示处理已经完成。一次可能处理多个CQ Entry。


###6.2.2 主机如何创建一条命令
主机创建命令前，首先检查对应的SQ是否为FULL，当有可用的SQ Entry可用时，继续以下步骤:

	a 


##6.3 复位


##6.4 队列管理

