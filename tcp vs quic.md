SACK 对比 QUIC ack frame

先看TCP recovery有哪些问题
1、TCP sack 具有以下缺点：
 - receiver 可能违约，乱序的包即使被sacked，接收者也可以根据内存使用的实际的情况将sacked seq抛弃。
 - sack block 有限(3个)，这就意味着sack的确认范围很窄
 - sack 只适用于FR和ER，当RTO 超时触发，sack机制将被关闭直到上次记录的RecoveryPoint被确认，并且清空之前的sack消息(RTO超时在tcp传输中会严重影响性能，在RTO 期间 receiver很可能会根据需要丢弃sacked的包，sender无法预知哪些包将会被丢弃(ps: rfc上没有指明，但如果能有协议保证丢包顺序，也许可以大大提升传输速率)，所以我们将抛弃所有sack信息)。在slow start 期间重新收集sack信息。
 - sack的HighRxt游标是一个单向游标，既只会从低seq number 到高seq number。这意味着如果重传后如果数据丢失，那么对tcp的性能是致命影响。因为只有下一轮重传才会重传丢失的帧,在此期间内tcp 后续重传的数据全部不可信，直到smallest unsacked帧被确认
 - sack的触发需要第DupThresh个dack，但是dack 也是可以丢的。所以在第DupThresh个dack到第DupThresh-1个dack 之前可能丢了很多个dack，多个rtt。这部分quic也没有办法只有靠ER的TLP。

1、TCP Early Retransmit的缺点
 - ER_threashold = outgoing_segments - 1, 既 dacks >= ER_threashold则会触发重传，举个例子：1,2,3个segment被发送，但是2丢失。如果实现delay ack并且不支持sack，当1的ack被delay，3的到达立即触发ack(乱序seg)，ER_threashold = 1，但是再也没有下一片触发ack，也就没有dack。这样我们必须等到RTO才能重传。只有支持sack，他可以告诉我们3已经被收到了，这样会立即触发ER。
 - RFC文档指出最坏的情况：如果传输2片，这样ER_threashold == 0，所以只要这两片重排，那么就会重传，缺乏对重排的弹性。
 - 还是3片1，2，3但是我们丢了后2片，确认了1, 那么即使开启sack，也没法触发ER。

再看看QUIC 如何解决这些问题
 - QUIC ack 包含足够的段保证
 - QUIC 和TCP 一样对包的重排有一定容忍度，QUIC的ack frame 针对packet number，QUIC packet number连续且单调递增，永不重复。这样被确认到的pkt num必然有效，而不会和tcp一样卡在unacked的第一个segment，导致后续包呈现不确定状态。
 - QUIC 类似于sack，但是不再需要多个dack(QUIC 没有dack的概念)，当一个包在QUIC ack block的范围以外，有满足两个条件的任意一个就会认为这个包丢失：
   - 最大的确认包pkt num - 该包的pkt num > reordering_threshold，既超出容忍阈值。对应FR。
   - 该包丢失后持续一定时间(具体时间算法有两种，但不会超过2个rtt)，对应ER。相比于TCP的ER，超时明显靠谱的多。
 - 由于QUIC pkt number 单调递增，当重传时我们必须重新打包frame，生成一个新的包，这个包的pkt number 必然大于以前的包，并且这个包应该(之所以说应该是因为，spec并没有强制规定。)包含新的数据。这样可以使得QUIC不像TCP Sack的HighRxt，卡在某一个关键数据上。
