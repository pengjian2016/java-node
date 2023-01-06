### 计算机网络七层体系结构（OSI七层结构）、TCP/IP四层模型、网络五层体系结构

 **七层体系结构（OSI七层结构）** ：为了使全世界不同体系结构的计算机能够互联，国际化标准组织ISO提出开放系统互联基本参考模型，简称OSI，即所谓的7层协议体系结构。

 **TCP/IP四层模型** ：是由实际应用发展总结出来的，它包含了应用层、传输层、网际层和网络接口层

 **五层体系结构** ：五层模型只出现在计算机网络学习教学过程中，他是对七层模型和四层模型的一个折中，及综合了OSI和TCP/IP 体系结构的优点，这样既简洁又能将概念阐述清楚，（主要是因为官方的7层模型太过麻烦复杂）因此主要差别是去掉了会话层和表示层，而传输层改为了运输层，因为他们觉得运输名字更贴切。
 
 ![输入图片说明](network.png)
 
### Http在哪一层？tcp/udp在哪一层？ip在哪一层？你还能说出哪些协议？

以下图片来自[javaguide之OSI 和 TCP/IP 网络分层模型详解（基础）](https://javaguide.cn/cs-basics/network/osi&tcp-ip-model.html)，可以直观的看到每一层对应的协议。

![输入图片说明](image.png) 

|     | 对应协议                     |
|-----|--------------------------|
| 应用层 | http 超文本传输协议 <br> smtp 电子邮件协议 <br> ftp 文件传输协议 <br> telnet 远程登陆协议<br> dns域名系统 |
| 运输层 | tcp ,udp                  |
| 网络层 | ip                       |


即http在应用层，tcp/udp在运输层（传输层），ip在网络层，各个层还有很多其他协议，有兴趣的可以研究一下，接下来主要针对Http，tcp/udp进行研究。

### TCP/UDP

- TCP 的三次握手和四次挥手描述一下？

- TCP 报文结构了解吗？

- TCP如何保证可靠性的？

- SYN攻击是什么？

- UDP传输过程描述？为什么是不可靠的？

### http
- http介绍
- http格式描述？
- get，post区别
- https是什么？


参考：

https://zhuanlan.zhihu.com/p/163691647

https://blog.csdn.net/liuchengzimozigreat/article/details/100169829