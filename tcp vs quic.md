TCP recovery

# reno
在经典reno的算法中，tcp 使用3个dack来触发fast retransmit。但是在经典reno里存在一些问题。举个例子：一次发送了8个包(tcp 分段)0 1 2 3 4 5 6 7，此时由于一次网络抖动丢了0 5包。我们在收到1 2 3时发送3个dack触发fast retransmit重传0(由于我们收到了3个dack，代表网络上少了3个包，此时根据包数守恒原则我们可以重传1 2 3，这显然也是没有必要的)，然后降低ssthresh 启动fast recovery。之后 1 2 3的重传包会导致3个对5的dack。会触发5的 的fast retranmsmit，再次启动fast recovery。
这里存在一些问题：
 - 1、当第一个包 0 丢失，我们重传了0。由于我们收到了3个dack(1 2 3)，代表网络上少了3个包，此时根据包数守恒原则我们可以重传1 2 3，这显然也是没有必要的。
 - 2、重传的1 2 3 引起5 的3个dack是不合理的，根据重排特性，tcp给与了3个包的重排弹性(这也是为什么 需要3个dack 才会触发fast retransmit)所以，这里应该由6 7 8 触发的dack才能判断5丢失
 - 3、我们因为一次网络抖动启动了2次 fast recovery，并且这两次基本是连续的。这会导致cwnd 和ssthresh 降低到一个很小的值。恢复起来也会变得很慢。如果出现多个丢包，对tcp而言将会严重影响速率。
 - 4、如上例子，假如只有5丢失，我们发送6 7两个包，这显然不够3个包触发dack，那么这个包只能等到rto才会重传。



# new reno解决上述问题3，部分解决问题2
new reno  设置一个最大的recovery 点，一般为highest seq即为7，在7以内的丢包只会降低一次窗口，这样可以解决问题3。new reno还会检查previous_highest_ack 和当前highest_ack的差值是否大于4，如上述2例子，如果是，则表示是由重传包导致3个dack，这样部分解决2.同样时间戳也可以达到同样效果，但不过多说明。但new reno也存在些问题
 - 1、假如在一次窗口的恢复期中丢失了多个包，并且highest_ack - previous_highest_ack  < 4 MSS，如上例子假如0 2 丢失，在第一次重传时 我们会重传0 1 2 3，此时重传的2又丢了，0 1 3 会触发dack 触发2 的fast retransmit，如果重传的2重排的，接收端收到的顺序是重传的0 1 3 2，也会触发fast retransmit。同样按照重排弹性原则，应该由 3 以后的包触发的dack才能认为2 丢失。
 - 2、new reno 任然无法避免重传无用包。
 - 3、new reno 任然无法解决尾包问题
 - 4、reno 建议tcp实现者在收到乱续包时立即发送ack 加速恢复效率。但是如果tcp没有遵从这个建议，那么对于new reno的性能而言会产生重大影响。因为当我们delay ack的时候会导致ack number的一次跃迁，这样会很可能大于4个MSS，而停止fast retransmit。
 - 5、同4 ack的丢失也会影响性能

# early retransmit尝试去解决尾包问题：
ER给与了一个算法，通过计算网络中的包数来判断是否需要重传尾包，但ER算法任然有些问题
TCP Early Retransmit的缺点
 - RFC文档指出最坏的情况：如果传输2片，这样ER_threashold == 0，所以只要这两片重排，那么就会重传，缺乏对重排的弹性。
 - 还是3片1，2，3但是我们丢了后2片，确认了1, 法触发ER。

# sack 尝试解决不必要的重传。
通过告诉发送者我们收到的哪些包，来避免不必要的重传。sack block 记载了接收者接收到的数据段。但是sack依然有缺陷：
 - sack的block只有3个且遵循fifo原则。如果在高丢包率的网络3个明显不够
 - sack 永远保存最新信息，对于旧的丢包信息，sack 无法检测到。
 - sack 不可靠，如果我们卡在某个丢包上，那么即使这个包对应的dack声明了部分后续包接收到了，这后续宝也有可能被重传。这是应为sack block声明并不可靠，tcp可以根据当前中断的内存或者其他信息抛弃掉不需要的数据。
 - RTO之后sack清零，如上所述RTO 默认为receiver 违约。
 - sack并不是每个设备都支持

# fack 利用sack解决ack丢失而无法进入fast retransmit问题
fack记录sack 接收到的最大包fack，既fack - unacked > 3 mss || dack == 3 即可进入fast retransmit。避免因为dack丢失或者不足，而无法触发fast retransmit.并且纠正了网络上的包的数量的统计值(在fack之前，丢包只有重传包被正确响应时 才会减去对应的值)，避免窗口不足而出现的阻塞现象。

# TCP RACK(draft-ietf-tcpm-rack-01 暂未被归纳入rfc)
为了解决TCP ER的缺点，TCP实现了另一种机制RACK。RACK并不是一种特殊的ack 格式，而是发送端在收到ack时候的一种特殊处理逻辑，用于尾包的丢包检测。
 - RACK实现了了尾包的超时定时器(TPO)，而不像ER哪像需要等待一个固定dack threshold，而至使导致rto超时。
 - 当收到ack时tcp不再依赖dack的次数，而是依赖收到包的时间，当前时间和与最后一次测量的rtt，确定一个小于rto的恢复周期，在这个期限内收到乱序的包对应的ack即为有效。

# 综上所述：
目前tcp任然有以下几点问题：
 - tcp rack未被实现，目前只有dev linux 存在rack。所以尾包的丢失依然是个弱项。
 - sack 对旧包的兼容性和违约处理依然存在弱点。
