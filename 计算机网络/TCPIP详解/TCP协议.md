# TCP协议
#计算机网络/TCPIP

<a href='TCPIP%E9%83%A8%E5%88%86-%E5%8D%8F%E8%AE%AE%E4%BB%8B%E7%BB%8D.pdf'>TCPIP部分-协议介绍.pdf</a>

![](TCP%E5%8D%8F%E8%AE%AE/66E75EB9-309A-45AA-A71C-099C0C5C1A29.png)


# 通过序列号和ACK确认应答提高可靠性
![](TCP%E5%8D%8F%E8%AE%AE/E28860F6-A0DB-4B20-BF8D-CF060937B656.png)

**ACK确认机制的理解**
![](TCP%E5%8D%8F%E8%AE%AE/BFC91124-4F78-4D40-8E11-24A491AB3D9C.png)
ACK确认里，对于发送方来说无论是发出去的包丢了还是接收方其实收到包了只是回来的ACK包丢了，它都认为数据没有发送成功，经过一段时间的等待会重发。
所以这时候接受方会收到重复的数据，接受方TCP层通过序列号来过滤重复数据交给应用层。

**序列号的理解**
![](TCP%E5%8D%8F%E8%AE%AE/FFC772F9-0ADF-4FCF-B208-23AD15AC559B.png)
![](TCP%E5%8D%8F%E8%AE%AE/422B14AF-B019-4070-AA0B-9A8E8CE59E4A.png)
电脑一开机操作系统会通过算法维护序列号并且不断变化，发包的初始序列号值是在2方SYN时拿到并设置的，未来每一方发送的数据包序列号都是在这个值基础上往上涨。
![](TCP%E5%8D%8F%E8%AE%AE/CC27D515-8520-4670-9B56-4CB24ABFB53F.png)
![](TCP%E5%8D%8F%E8%AE%AE/6A1B618C-442A-43B6-A6DC-E89F6FAC88A3.png)
如果能破解序列号增长规律，可以做黑客攻击。

# TCP以段为单位进行数据发送
![](TCP%E5%8D%8F%E8%AE%AE/064882B0-328B-4DAB-9D70-D9E5C2C271AA.png)
滑动窗口大小以MSS为单位。

# 滑动窗口技术
滑动窗口技术的功能特点总结：
1.提高发送速度。
2.也是一种流控机制。
3.有效率比较高的快速重传机制。

![](TCP%E5%8D%8F%E8%AE%AE/EF6B2904-22BB-48C6-99ED-E612B88B2E65.png)
![](TCP%E5%8D%8F%E8%AE%AE/BCCF03BE-729D-4CA4-8E74-BE4A2B32624F.png)
![](TCP%E5%8D%8F%E8%AE%AE/89C7FA55-6CFC-4867-BCA2-ACA3363F4716.png)
不只提高速度，窗口大小也就代表了接受方的缓存大小，所以滑动窗口也是一种接收方的流控机制，当然发送方也有流控机制就是慢启动和用塞控制。

虽然窗口大小为4个段（4个MSS大小），但是也不是4个段的数据一起发，虽然可以这样做，接受方也有这个能力可以收到，但是因为网络拥堵问题牵扯到另一个特性，就是慢启动，有个用塞窗口控制着发送方发送数据的大小，最终会选择2者中较小的作为一次发送数据段的最大个数，实际发送数量还和操作系统的资源调度有关，所以就算滑动窗口大小为4个段，TCP发数据也不一定发4个段。

接受方会在Ack包中通告窗口大小，大小和接受方发出ack时缓存空间的大小有关。

**滑动窗口里的快速重传**
![](TCP%E5%8D%8F%E8%AE%AE/A038B94F-3362-4907-9E06-6BDD1A954D48.png)
![](TCP%E5%8D%8F%E8%AE%AE/00B6C35F-BE55-47BD-8C5C-CCE74842AA75.png)这里的重传比超时重传效率更高，收到3个重复确认应答就重传，而且只需要传丢失的那个包数据。现在的TCP支持选择确认功能。

# TCP中的流量控制
## 接收方-窗口大小协商
![](TCP%E5%8D%8F%E8%AE%AE/C2DA7922-D5B2-441A-B922-708762ED0065.png)
![](TCP%E5%8D%8F%E8%AE%AE/7DD69D4C-714C-48BC-B521-EDE74A6DB9A5.png)
接受方缓存大小是不断变化的，当接受方缓存满了的时候，通告给发送方，滑动窗口完全可能会为0，发送方会停止发送。某一时刻接收方缓存空闲出来的时候，会发送一个窗口更新的通知包给发送方，去更新窗口大小，但是这个包也可能会丢，所以还需要当窗口大小为0时发送方间隔一段时间就发送一个窗口探测的数据包过去，询问接收方的窗口大小。

## 发送方-拥塞控制
![](TCP%E5%8D%8F%E8%AE%AE/02D8710C-D1F4-4958-A779-88AB55D1A234.png)
![](TCP%E5%8D%8F%E8%AE%AE/E1321971-4503-4BA5-AEB3-76FB4CE9460D.png)
要理解，实际中并不是发送一个报文段就一定会收到一个ack，很有可能接收方收到连续2个报文段只回复序列号较高的最后一个。

当用塞窗口增大到一定大小的时候，发送的数据迟迟收不到ack，就会触发超时重传机制，这是一种比收到重复ack快速重传严重的重传机制，因为它表示很有可能数据丢包了，网络拥堵了，这时候会把用塞窗口直接降为1，重新慢启动发送。

![](TCP%E5%8D%8F%E8%AE%AE/1BCBCD3B-3501-4787-93DF-3BE68999EEDC.png)
![](TCP%E5%8D%8F%E8%AE%AE/4732A3E2-D5EF-4679-A58B-AF8C885E77F2.png)
慢启动阀值的每次设置是在发生超时重传或者快重传之后，大小为之前最大拥塞窗口的一半。

如果是重复ack触发的快重传机制的话，拥塞窗口大小不会直接降为1进入慢启动，而是在更新后的慢启动阀值基础上+3，直接进入拥塞避免。

# TCP提高网络效率的几种机制
![](TCP%E5%8D%8F%E8%AE%AE/95A0283F-DAF4-4029-B9C9-D025F6AFDF15.png)
![](TCP%E5%8D%8F%E8%AE%AE/8086F869-470D-4A40-9680-1B0B8B1DFE62.png)
延时确认可以提高网络利用率，因为没有必要收到1个就回复ack，因为很有可能刚收到那一瞬间接收方缓存空间比较小，这时候回复ack会导致滑动窗口也变小，所以可以稍微等一下再回复，一般是收到2个包后才回复（也不一定）。或者接收方如果在某一个时间内（操作系统可以设置，一般是0.2s）还是没收到包，这时候必须回复一个ack包。

![](TCP%E5%8D%8F%E8%AE%AE/385B0AF5-3258-46B6-A2D0-6A6BC5107219.png)
![](TCP%E5%8D%8F%E8%AE%AE/684EAC2E-5EF3-4548-B4CB-D6F608AC982E.png)
捎带应答一般是在交互式数据流中（你发一个，对方也要回一个），这时候接受方收到数据包后也会进行延迟确认，因为TCP是全双工的，所以等到接收方准备好需要发给发送方的数据后，把ack和准备好的数据包一起发过去，这样也挺高了效率。