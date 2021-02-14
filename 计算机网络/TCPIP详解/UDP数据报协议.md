# UDP数据报协议
#计算机网络/TCPIP


![](UDP%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE/16B7002B-FE99-4C9D-ABC0-323E2485C7B9.png)
UDP对于应用层下来的数据不挑小也不挑大，封装个头部就交给IP。所以数据包容易超过MTU造成IP对数据包进行分片发送。tcp一般不会超过MTU也不会分片，因为它总是按照合适的存储单元进行封包，太小的就等一会再封包交给IP。

# UDP首部
![](UDP%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE/1749FEC4-FBAD-4CD1-B7FF-A34FEF393B89.png)
源目的端口号各占16位。
UDP长度字段占16位，单位为bety，标示的是UDP首部和UDP数据的总长度，注意和IP首部长度字段区分开，最大65535bety。

![](UDP%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE/5EE4F35E-817B-4E4E-A39F-2634CEB5B9A5.png)
UDP校验和占16位，校验的是UDP首部和数据部分，它是可选的，而TCP是必须的。注意IP首部校验和只校验IP首部。

注意UDP校验和计算需要包含一些伪首部信息。

# UDP发包举例
![](UDP%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE/EC22CAFB-8281-4D64-AA9B-DD7CB48BEEC7.png)




