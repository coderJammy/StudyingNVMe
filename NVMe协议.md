#NVM Express 协议

#文档说明

本文可以看做是对[NVME1.2版协议][1]的研读笔记，部分段落是协议原文的直接翻译。一些数据格式则是直接以C语言结构体的形式呈现。


#1 NVM Express协议是什么？
NVM(Non-Volatile Memory)即非易失存储器。NVM Express是一套主机与存储设备之间的访问接口（寄存器级的接口）。典型应用如挂在PCIE总线下的SSD设备。

NVM Express协议描述了如何通过寄存器接口访问NVM子系统，并定义了一组访问NVM子系统的标准命令集。

##1.1 

##1.2 (字母，词组，缩写等)定义

	*Admin Queue ： 管理队列。指ID为0的提交队列(Submision Queue)和完成队列(Completion Queue)。或称为Amdin Submision Queue和Amdin Completion Queue。主机通过此队列发送Admin命令和接收命令状态。

	*arbitration burst : 提交队列一次可提交的最大命令个数。


#2 控制器寄存器(Controller Registers)

Controller Registers是一段存储器空间，可以接次序访问或按变量位宽访问（要具有这两特点，有些计算机架构需定义此空间为非缓存的空间）。主机须用本机位宽或32BIT对齐访问，否则会引发不确定的行为。不支持一次访问两个或多个寄存器。未定义的寄存器或寄存器中未定义的位读为0。

##2.1 寄存器定义

	+--------+--------+-------------------------------+
	| Start  | End    | Description
	+--------+--------+-------------------------------
	| 0x0000 | 0x003F | nvme bar(see struct nvme_bar)
	+--------+--------+-------------------------------
	| 0x0040 | 0x0EFF | Reserved
	+--------+--------+-------------------------------
	| 0x0F00 | 0x0FFF | Command Set Specific
	+--------+--------+-------------------------------
	|0x1000+x|0x1003+x| SQn Tail Doorbell
	+--------+--------+-------------------------------
	|0x1000+y|0x1003+y| CQn Head Doorbell
	+--------+--------+-------------------------------

~~~{.c}
	struct nvme_bar {
		__u64			cap;	/* Controller Capabilities */
		__u32			vs;	/* Version */
		__u32			intms;	/* Interrupt Mask Set */
		__u32			intmc;	/* Interrupt Mask Clear */
		__u32			cc;	/* Controller Configuration */
		__u32			rsvd1;	/* Reserved */
		__u32			csts;	/* Controller Status */
		__u32			rsvd2;	/* Reserved */
		__u32			aqa;	/* Admin Queue Attributes */
		__u64			asq;	/* Admin SQ Base Address */
		__u64			acq;	/* Admin CQ Base Address */
	};
~~~