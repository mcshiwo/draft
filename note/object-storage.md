
###CAP
C: Consistency(一致性):任何一个读操作总是能读取到之前完成的写操作结果，也就是在分布式环境中，多点的数据是一致的；
A: Availability(可用性):每一个操作总是能够在确定的时间内返回，也就是系统随时都是可用的；
P: Tolerance of network Partition(分区容忍性):在出现网络分区（比如断网）的情况下，分离的系统也能正常运行；

###一致性哈希指标

平衡性 (Balance)：哈希算法要均匀分布，不能有明显的映射规律
单调性 (Monotonicity)：有新节点加入时，已经存在的映射关系不能发生变化
分散性 (Spread) ：避免不同的内容映射到相同的位置和相同的内容映射到不同的位置


####虚拟节点--平衡性
为解决平衡性问题引入,虚拟节点”的 hash 计算可以采用对应节点的 IP 地址加数字后缀的方式

	虚拟节点个数在集群的整个生命周期中是不会变化
	虚拟节点与对象映射关系不会发生变化
	改变的仅是虚拟节点与节点的映射关系
	合理设置虚拟节点数目,一般是是节点个数的100倍

####Replica-冗余性
NWR是一种在分布式存储系统中用于控制一致性级别的一种策略

	N：同一份数据的Replica的份数，=3
	W：是更新一个数据对象的时候需要确保成功更新的份数，=2
	R：读取一个数据需要读取的Replica的份数，=1 or 2

####Zone-分区容忍性
分区容忍性：除了全部网络节点发生故障以外，所有子节点集合的故障都不允许导致整个系统的响应故障

	N：同一份数据的Replica的份数，=3
	W：是更新一个数据对象的时候需要确保成功更新的份数，=2
	R：读取一个数据需要读取的Replica的份数合理设置虚拟节点数目，=1 or 2


###特性

####超强的扩展性

扁平化的数据结构允许对象存储容量从TB级扩展到EB级,管理数十个到百亿个存储对
象,支持从数字节(Byte)到数万亿字节(TB)范围内的任意大小对象,解决了文件系统复杂
的iNode的机制带来的扩展性瓶颈,并使得对象存储无需像SAN存储那样管理数量庞大的逻
辑单元号(LUN)。对象存储系统通常在一个横向扩展(或网格硬件)架构上构建一个全
局的命名空间,
这使得对象存储非常适用在云计算环境中使用。
某些对象存储系统还可支持
升级、扩容过程中业务零中断。

####基于策略的自动化管理
由于云环境中的数据往往是动态、快速增长的,所以基于策略的自动化将变得非常重要。
对象存储支持从应用角度基于业务需求设置对象/容器的属性(元数据)策略,如数据保护
级别,保留期限,合规状况,远程复制的份数等。
这使得对象存储具备云的自服务特征同时, 有效的降低运维管理的成本,使得客户在存储
容量从TB增长到ZB时,运维管理成本不会随之飙升。

####多租户
多租户特性可以使用同一种架构,同一套系统为不同用户和应用提供存储服务,并分别为
这些用户和应用设置数据保护、数据存储策略,并确保这些数据之间相互隔离。

####数据完整性和安全性
对象存储系统一般通过连续后台数据扫描、数据完整性校验、自动化对象修复等技术,
新型的技术应用大大提高数据的完整性和安全性。
某些对象存储产品还引入了一些先进的算法(如:擦除码Erasure Code )和技术将数据切分
为多个分片,然后将这些分片存储到不同的设备/站点,在确保数据的完整性的同时获取最高
的存储利用率


###应用
####存储资源池(空间租赁)
使用对象存储构建类似Amazon S3的存储空间租赁服务,向个人、企业或应用提供按需
扩展的弹性存储服务。用户向资源池运营商按需购买存储资源后,通过基于web协议访问和
使用存储资源, 而无需采购和运维存储设备。多租户模型将不同的用户的数据隔离开来,确
保用户的数据安全。
####网盘应用
在海量存储资源池基础上,使用图形用户界面(GUI)实现对象存储资源的封装,向用
户提供类似Drop Box的网盘业务。用户可通过PC客户端、手机客户端、Web页面完成数据
的上传、下载、管理与分享。在网盘帮助下个人和家庭用户能够实现数据安全、持久的保存
和不同终端之间的数据同步;
企业客户通过网盘应用可实现更高效的信息分享、协同办公和非结构化数据管理,同时企业网
盘还可用于实现低成本的Windows远程备份,确保企业数据安全。

####集中备份
在大型企业或科研机构中,
对象存储通过与Comvault Simpana、
Symantec NBU等主流备
份软件结合,可向用户提供更具成本效益、更低TCO的集中备份方案。相对原有的磁带库或
虚拟磁带库等备份方案:重复数据删除特性能够帮助用户减少低设备采购,智能管理特性使得
备份系统无需即时维护,从而降低CAPEX和OPEX;分布式并行读写带来的巨大吞吐量和
在线/近线的存储模式有效降低RTO和RPO。

####归档和分级存储
对象存储通过与归档软件、分级存储软件结合,将在线系统中的数据无缝归档/分级存
储到对象存储,释放在线系统存储资源。对象存储提供几乎可无限扩展的容量,智能管理能
力,帮助用户降低海量数据归档的TCO;对象归档采用主动归档模式使得归档数据能够被按
需访问,而无需长时间的等待和延迟。

**附注**
Amazon的主营业务是B2C电子商务,其用户流量分布不均匀,某些特定的时段(比如圣诞节),流量会急剧攀升,亚
马逊在IT资源的投资非常尴尬--花大价钱购置的服务器、存储、带宽只是为了应对突发的高峰流量,而在其他大部分时间里,这
些资源利用率可能都不到一半。在这种情况下,Amazon 通过AWS服务Amazon把平时闲置的IT资源出租给其他用户使用。


###参考文献
探秘对象存储