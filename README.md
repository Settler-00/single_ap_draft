# single_ap_draft




链路阻塞标准：
高可靠传输
针对高优先级业务，当两个频段上的拥塞时延和拥塞长度均保持较低值时，表示当前链路通畅。在这种情况下，不使用上述的比例调度发送，而是将业务报文在两个频段上冗余传输，从而进一步提高高优先级业务报文的传输可靠性。
设定链路通畅的满足条件为：
(拥塞时延 <= 1.1*平均往返时延) && (拥塞长度 <= 8)

双频阻塞条件：
针对两个频段的链路都具有较高拥塞时延和拥塞长度的场景，尝试使用更高一级优先级的硬件队列进行报文发送。若更高一级优先级的硬件队列链路满足通畅条件，则暂时借用该硬件队列发送报文。
设定链路阻塞的满足条件为：
(拥塞时延 >= 5*平均往返时延) || (拥塞长度 > 64)
考虑到流量控制线程会配合进行链路拥塞检测，一旦出现双频拥塞的场景，流量控制线程会降低业务带宽，从而降低链路传输压力，因此双频拥塞的持续时间不会很长。



双频多通道调度的具体实现
（1）报文获取
①流量控制线程将不同优先级的报文放在对应优先级的Ring Buffer中，并触发信号量，通知发送线程取出待发送的业务报文。
②发送线程被唤醒后，从各个Ring Buffer中取出对应优先级的报文，送入调度函数处理。
（2）发送调度
①根据当前报文优先级对应两个频段的硬件队列的拥塞时延和拥塞长度，计算两个频段的发送比例。
②根据报文发送比例，选择其中一个频段用于发送。
设2.4GHz的发送比例为A，5GHz的发送比例为1-A。
在0到1之间生成一个随机数rand。
若随机数rand在0到A之间，则选择2.4GHz发送；否则选择5GHz发送。
③针对高优先级报文，若两个频段的硬件队列的拥塞时延和拥塞长度均保持较低水平，即链路通畅，则将业务报文在两个频段上冗余传输，以保证传输可靠性。
④两个频段的硬件队列均阻塞时：
最高优先级业务：由于性能最好的AC_VO硬件队列都无法抢占到信道进行发送，说明此时的信道质量极差，使用默认的比例发送机制。
其他优先级业务：判断本优先级向上一个优先级的硬件队列拥塞情况，若该硬件队列的某个频段下链路通畅，则暂时借用该硬件队列发送报文；否则继续等待，尝试发送。
⑤将报文发送使用的频段、硬件队列等信息保存进结构体，送给封装函数用于封装、缓存和发送。



技术文档相关：
120Mbps  1Mbps  RTT双向<50ms@3个9  RTT双向<15ms@5个9  总121Mbps
240Mbps  1Mbps  RTT双向<50ms@3个9  RTT双向<15ms@5个9  241Mbps
设计的整体架构分上行和下行分别描述，在每种链路设计中又分为发送方和接收方。发送方的设计主要在于发送流程的修改，通过相关字段的添加和修改，以及对数据包的频段和优先级的分配来实现对数据包不同的发送方式，然后调用原流程发送。总体设计概要如下表所示。
下行：
发送方：双频多队列冗余发送；速率降级处理
接收方：冗余去重
上行：
发送方：ack确认单频高优先级发送；TCP业务报文5G多队列平均发送
接收方：ack确认和数据报文分开整序

其中，下行链路的设计在ONT处对数据包做冗余处理，对于同一个数据包，分别使用两个频段，在VI、BE、BK三个硬件队列发送，一共发六份。同时，在发送端做速率降级处理，用比较低的MCS等级发送，使用更低码率的调制编码方式，从而提高可靠性。

上行链路的设计依据报文类型分为两种情况，当数据包为ack确认报文时，正常使用单频，并将其映射到优先级最高的队列VO发送；当数据包为上行的业务报文时，则依据包的序号将其等分为三份，分别从剩余三个队列的5G频段发送。在接收端ONT处依据报文的类型，将ack确认和上行数据报文分开设置原子变量计数，然后分别映射到两个接收窗口作整序。

下行冗余发送
由于下行流量不大，为减少丢包可以发多份数据包，其对时延的影响有限，因此可以做冗余发送处理，使用多个AC队列同时发包，确保发出的数据包最大限度地能被接收方接收，减少丢帧，同时走不同的硬件队列也大大降低汇聚的概率，从而提升传输的可靠性。
此设计中，我们选择三个硬件队列同时发送，剩下的一个作备用。因此，对于同一个数据包，分别用两个频段、三个硬件队列做冗余发送，一共发六份。

上行确认优先发送设计
由于要让TCP确认优先发，故选择优先级最高的VO队列，而其可容忍一部分的丢包，因此无需再进行多队列冗余发送。


上行TCP业务多队列发送设计
由于上行的业务流量较大，采用一个硬件队列发送时，可能会导致发送窗口大小不够，即使在空口有空闲的情况下，也可能因为发送窗口的限制而制约了发包的效率。而通过将报文分配到多个不同的硬件队列发送，可有效地分担单个队列上的负载，减小窗口阻塞的概率，从而降低时延。
此设计中，对于上行的业务报文，我们充分利用VI、BE、BK三个硬件队列进行发送，发送的队列数量变多，同时每个队列所需承担的业务量仅为原来的三分之一。

