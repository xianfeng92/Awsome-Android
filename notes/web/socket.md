
## Java中对于网络提供的几个关键类

* InetAddress： 用于标识网络上的硬件资源

* URL： 统一资源定位符，通过URL可以直接读取或者写入网络上的数据

* Socket和ServerSocket:使用TCP协议实现网络通信的Socket相关的类

* Datagram：使用UDP协议，将数据保存在数据报中，通过网络进行通信



## 什么是Socket？

Socket 不属于协议范畴，而是一个调用接口（API），Socket 是对 TCP/IP 协议的封装，通过调用 Socket，才能使用TCP/IP协议。Socket连接是长连接，
理论上客户端和服务器端一旦建立连接将不会主动断开此连接。


## Socket通信模型


![Socket通信模型]()


Socket通信实现步骤解析：

1. 创建 ServerSocket 和 Socket

2. 打开连接到的Socket的输入/输出流

3. 按照协议对Socket进行读/写操作

4. 关闭输入输出流，以及Socket
