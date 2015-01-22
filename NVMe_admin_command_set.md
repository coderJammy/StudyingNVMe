# NVMe Admin Command Set

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

##1 Identify
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

###1.1 Controller Identify Struture

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

###1.2 Namespace Identify Struture

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

###1.3 Namespace List (spec 1.2)


###1.4 HOST驱动如何发送Identify

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


###1.5 NVM设备如何处理Identify命令
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


##2 创建I/O完成队列(Create I/O Completion Queue Command)

###**命令字抽象**

~~~{.c}

	struct nvme_create_cq {
		__u8			opcode;
		__u8			flags;
		__u16			command_id;
		__u32			rsvd1[5];
		__le64			prp1;
		__u64			rsvd8;
		__le16			cqid;
		__le16			qsize;
		__le16			cq_flags;
		__le16			irq_vector;
		__u32			rsvd12[4];
	};
~~~

###**Linux驱动发送创建CQ命令**
~~~{.c}

	static int adapter_alloc_cq(struct nvme_dev *dev, u16 qid,
							struct nvme_queue *nvmeq)
	{
		int status;
		struct nvme_command c;
		int flags = NVME_QUEUE_PHYS_CONTIG | NVME_CQ_IRQ_ENABLED;
	
		memset(&c, 0, sizeof(c));
		c.create_cq.opcode = nvme_admin_create_cq;
		c.create_cq.prp1 = cpu_to_le64(nvmeq->cq_dma_addr);
		c.create_cq.cqid = cpu_to_le16(qid);
		c.create_cq.qsize = cpu_to_le16(nvmeq->q_depth - 1);
		c.create_cq.cq_flags = cpu_to_le16(flags);
		c.create_cq.irq_vector = cpu_to_le16(nvmeq->cq_vector);
	
		status = nvme_submit_admin_cmd(dev, &c, NULL);
		if (status)
			return -EIO;
		return 0;
	}
~~~

分析以上代码：
	
	1. opcode  : 命令字。
	2. prp1    : 指向CQ的64bits内存地址或者PRP List指针。
	3. cqid    : CQ的唯一标识。
	4. qsize   : 指的是CQ entry的个数。
	5. cq_flag : 标志位。00b为1时prp1为一个64bits内存地址，否则为PRP表指针。01b为中断使能。
	6. irq_vector : 中断向量。

其中`flags = NVME_QUEUE_PHYS_CONTIG | NVME_CQ_IRQ_ENABLED;`
即flags=0x02，表示中断使能，而prp1值是一个PRP表指针，CQ不是物理连续的。

参数从何而来？
这些参数包括：`nvmeq->cq_dma_addr`，`qid`，`nvmeq->q_depth`，，`nvmeq->cq_vector`。
下面依次分析Linux驱动中的代码：

**`nvmeq->cq_dma_addr`和`nvmeq->q_depth`**

~~~{.c}

	#define NVME_Q_DEPTH		1024

	static int nvme_dev_map(struct nvme_dev *dev)
	{
		...
		dev->q_depth = min_t(int, NVME_CAP_MQES(cap) + 1, NVME_Q_DEPTH);
		...
	}
	static struct nvme_queue *nvme_alloc_queue(struct nvme_dev *dev, int qid,
								int depth, int vector)
	{
		...
		nvmeq->cqes = dma_alloc_coherent(dmadev, CQ_SIZE(depth),
						&nvmeq->cq_dma_addr, GFP_KERNEL);
		...
		nvmeq->cq_vector = vector;
		...
	}
~~~

由此可见，首先确定CQ的深度，读取寄存器`CAP`获得NVM控制器最多支持Queue Entries个数，`NVME_Q_DEPTH`为驱动定义最多支持的Queue Entries个数，取较小值。然后向Linux内核申请内存，大小为`CQ_SIZE(depth)`，返回的物理地址存于`nvmeq->cq_dma_addr`，而`nvmeq->cqes`是映射的虚拟内存地址。

**`qid`和`nvmeq->cq_vector`**

~~~{.c}

	static int nvme_create_queue(struct nvme_queue *nvmeq, int qid)
	{
		...
		result = adapter_alloc_cq(dev, qid, nvmeq);
		...
	}
	static void nvme_create_io_queues(struct nvme_dev *dev)
	{
		unsigned i, max;
	
		max = min(dev->max_qid, num_online_cpus());
		for (i = dev->queue_count; i <= max; i++)
			if (!nvme_alloc_queue(dev, i, dev->q_depth, i - 1))
				break;
	
		max = min(dev->queue_count - 1, num_online_cpus());
		for (i = dev->online_queues; i <= max; i++)
			if (nvme_create_queue(raw_nvmeq(dev, i), i))
				break;
	}
~~~

可知`qid`从`dev->online_queues`向上递增，`dev->online_queues`为已经创建的Queue个数。而`nvmeq->cq_vector`作为中断向量。



###**何时创建**

函数调用关系：

	nvme_cpu_workfn -> nvme_assign_io_queues -> 	nvme_create_io_queues ->	
	nvme_create_queue ->adapter_alloc_cq

初始化工作队列

	INIT_WORK(&dev->cpu_work, nvme_cpu_workfn);

nvme_cpu_workfn这个函数会在“一段时间”后被调用。这个“一段时间”则操作系统的调度有关。

**Controller创建CQ命令处理流程**
以QEMU为例
