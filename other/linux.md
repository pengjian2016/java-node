### linux  常用命令
作为一个高级工程师，像cd、ls、top这些简单的命令，咱就不说了。下面介绍一些部署、运维中常用到的需要了解的命令。

#### ssh 登录远程服务器

```
# 使用user登录 192.168.1.1服务器
ssh user@192.168.1.1
# 端口默认不是22时，指定端口
ssh -p 1022 user@192.168.1.1
```

#### scp 拷贝文件到服务器或从服务器下载文件

```
# 上传文件到服务器
scp test.txt user@192.168.1.1:/home
# 从服务器下载文件
scp user@192.168.1.1:/home/test.txt /temp

```
#### find 查找命令

```
# 根据文件名称查找
find . -name test.class
# 查找当前目录及子目录下文件长度为0的文件并列出
find . -type f -size 0 -exec ls -l {} \;

```

#### pwd 列出当前目录
```
$ pwd
/home/test
```

#### tail 查看文件内容

```
# 实时显示tomcat日志
tail -f tomcat.log

# 实时显示最近100行的日志
tail -f tomcat.log -n 100

# 显示nginx访问日志中 请求时间大于1的日志（其中$1 在nginx请求日志配置中的第一项为$request_time）
tail -f access.log  | awk '{if ($1 > 1) print $0}'

```

#### awk 字符分割、uniq 去除重复行、sort对文件的文本内容排序、wc统计文件的行数、单词数或字节数

```
# 根据ip 统计访问量（$1 表示nginx配置的第一项为remote_addr，awk默认以空格分割）

awk '{print $1}'  access.log|sort | uniq -c |wc -l

```

#### ping 测试当前服务器到指定ip是否通顺

```
$ ping www.baidu.com
PING www.a.shifen.com (180.101.50.242) 56(84) bytes of data.
64 bytes from 180.101.50.242 (180.101.50.242): icmp_seq=1 ttl=48 time=41.7 ms
64 bytes from 180.101.50.242 (180.101.50.242): icmp_seq=2 ttl=48 time=38.7 ms
64 bytes from 180.101.50.242 (180.101.50.242): icmp_seq=3 ttl=48 time=35.7 ms
64 bytes from 180.101.50.242 (180.101.50.242): icmp_seq=4 ttl=48 time=32.8 ms
64 bytes from 180.101.50.242 (180.101.50.242): icmp_seq=5 ttl=48 time=26.5 ms
^C
--- www.a.shifen.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 26.562/35.141/41.788/5.220 ms

```

#### curl 发送请求
```
$ curl www.baidu.com
# 返回的时百度首页html内容
```

#### netstat 查看网络状态

```
# 查看服务器上tcp连接情况
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

```

#### tcpdump tcp抓包工具

```
# 抓取http包
tcpdump -XvvennSs 0 -i eth0 tcp[20:2]=0x4745 or tcp[20:2]=0x4854 -w /tmp/capture.pcap

# 抓取 80 端口的包
tcpdump port 80
```
