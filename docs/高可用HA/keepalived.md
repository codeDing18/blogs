[toc]



# 1. Keepalived简介

## 1.1. 概述

Keepalived一个基于VRRP协议来实现的LVS服务高可用方案，可以利用其来避免单点故障。一个LVS服务会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候， 备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。 

## 1.2. keepalived的作用

Keepalived的作用是检测服务器的状态，如果有一台web服务器死机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中。

# 2. 如何实现Keepalived

## 2.1. 基于VRRP协议的理解

Keepalived是以VRRP协议为实现基础的，VRRP全称`Virtual Router Redundancy Protocol`，即`虚拟路由冗余协议`。
       
虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip（该路由器所在局域网内其他机器的默认路由为该vip），master会发组播，当backup收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master。这样的话就可以保证路由器的高可用了。

keepalived主要有三个模块，分别是core、check和vrrp。core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式。vrrp模块是来实现VRRP协议的。

## 2.2. 基于TCP/IP协议的理解

Layer3,4&7工作在IP/TCP协议栈的IP层，TCP层，及应用层,原理分别如下：

**Layer3：**

Keepalived使用Layer3的方式工作式时，Keepalived会定期向服务器群中的服务器发送一个ICMP的数据包（既我们平时用的Ping程序）,如果发现某台服务的IP地址没有激活，Keepalived便报告这台服务器失效，并将它从服务器群中剔除，这种情况的典型例子是某台服务器被非法关机。Layer3的方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。

**Layer4:**

如果您理解了Layer3的方式，Layer4就容易了。Layer4主要以TCP端口的状态来决定服务器工作正常与否。如web server的服务端口一般是80，如果Keepalived检测到80端口没有启动，则Keepalived将把这台服务器从服务器群中剔除。

**Layer7：**

Layer7就是工作在具体的应用层了，比Layer3,Layer4要复杂一点，在网络上占用的带宽也要大一些。Keepalived将根据用户的设定检查服务器程序的运行是否正常，如果与用户的设定不相符，则Keepalived将把服务器从服务器群中剔除。

# 3. Keepalived选举策略

## 3.1. 选举策略   

首先，每个节点有一个初始优先级，由配置文件中的priority配置项指定，MASTER节点的priority应比BAKCUP高。运行过程中keepalived根据vrrp_script的weight设定，增加或减小节点优先级。规则如下：

1. “weight”值为正时,脚本检测成功时”weight”值会加到”priority”上,检测失败是不加

 - 主失败:
      主priority<备priority+weight之和时会切换

 - 主成功:
      主priority+weight之和>备priority+weight之和时,主依然为主,即不发生切换

2. “weight”为负数时,脚本检测成功时”weight”不影响”priority”,检测失败时,Master节点的权值将是“priority“值与“weight”值之差

 - 主失败:
   主priotity-abs(weight) < 备priority时会发生切换

 - 主成功:
   主priority > 备priority 不切换

3. 当两个节点的优先级相同时，以节点发送VRRP通告的IP作为比较对象，IP较大者为MASTER。

## 3.2. priority和weight的设定     

1. 主从的优先级初始值priority和变化量weight设置非常关键，配错的话会导致无法进行主从切换。比如，当MASTER初始值定得太高，即使script脚本执行失败，也比BACKUP的priority + weight大，就没法进行VIP漂移了。
2. 所以priority和weight值的设定应遵循: abs(MASTER priority - BAKCUP priority) < abs(weight)。一般情况下，初始值MASTER的priority值应该比较BACKUP大，但不能超过weight的绝对值。 另外，当网络中不支持多播(例如某些云环境)，或者出现网络分区的情况，keepalived BACKUP节点收不到MASTER的VRRP通告，就会出现脑裂(split brain)现象，此时集群中会存在多个MASTER节点。



# 4. Keepalived的安装

## 4.1. yum install方式

```bash
yum install -y keepalived
```

## 4.2. 安装包编译方式

更多安装包参考：http://www.keepalived.org/download.html

```bash
wget http://www.keepalived.org/software/keepalived-2.0.7.tar.gz
tar zxvf keepalived-2.0.7.tar.gz
cd keepalived-2.0.7
./configure --bindir=/usr/bin --sbindir=/usr/sbin --sysconfdir=/etc --mandir=/usr/share
make && make install
```

# 5. 常用配置

keepalived配置文件路径：`/etc/keepalived/keepalived`。

## 5.1. MASTER（主机配置）

```shell
global_defs {
    router_id proxy-keepalived
}

vrrp_script check_nginx {
    script "/etc/keepalived/scripts/check_nginx.sh" 
    interval 3  
    weight 2   
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth2
    virtual_router_id 15
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxx
    }
    track_script {
        check_nginx 
    }
    virtual_ipaddress {
        180.101.115.139
        218.98.38.29
    }
    
	nopreempt
	
	notify_master "/etc/keepalived/keepalived_notify.sh master"
	notify_backup "/etc/keepalived/keepalived_notify.sh backup"
	notify_fault "/etc/keepalived/keepalived_notify.sh fault"
	notify_stop "/etc/keepalived/keepalived_notify.sh stop"
}

```

## 3.2. BACKUP（备机配置）

```shell
global_defs {
    router_id proxy-keepalived
}

vrrp_script check_nginx {
    script "/etc/keepalived/scripts/check_nginx.sh" 
    interval 3  
    weight 2   
}

vrrp_instance VI_1 {
    state BACKUP 
    interface eth2
    virtual_router_id 15
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxx
    }
    track_script {
        check_nginx 
    }
    virtual_ipaddress {
        180.101.115.139
        218.98.38.29
    }
    
	nopreempt
	
	notify_master "/etc/keepalived/keepalived_notify.sh master"
	notify_backup "/etc/keepalived/keepalived_notify.sh backup"
	notify_fault "/etc/keepalived/keepalived_notify.sh fault"
	notify_stop "/etc/keepalived/keepalived_notify.sh stop"
}

```

# 6. 注意事项

1、指定Nginx健康检测脚本：/etc/keepalived/scripts/check_nginx.sh

2、主备配置差别主要为（建议这么配置）：

> 以下两种方式的配置，当其中一台机器keepalived挂掉后会自动VIP切到另一台机器，当挂掉机器keepalived恢复后不会抢占VIP，该方式可以避免机器恢复再次切VIP所带来的影响。

- 主机:(state BACKUP;priority 100)

- 备机：(state BACKUP;priority 99)

- 非抢占：nopreempt

  或者：

- 主机:(state MASTER;priority 100)

- 备机：(state BACKUP;priority 100)

- 默认抢占

3、指定VIP

```
    virtual_ipaddress {
        180.101.115.139
        218.98.38.29
    }
```

4、可以指定为非抢占：nopreempt，即priority高不会抢占已经绑定VIP的机器。

5、制定绑定IP的网卡： interface eth2

6、可以指定keepalived状态变化通知

```
	notify_master "/etc/keepalived/keepalived_notify.sh master"
	notify_backup "/etc/keepalived/keepalived_notify.sh backup"
	notify_fault "/etc/keepalived/keepalived_notify.sh fault"
	notify_stop "/etc/keepalived/keepalived_notify.sh stop"
```

7、virtual_router_id 15值，主备值一致，但建议不应与集群中其他Nginx机器上的相同，如果同一个网段配置的virtual_router_id 重复则会报错，选择一个不重复的0~255之间的值，可以用以下命令查看已存在的vrid。

```bash
tcpdump -nn -i any net 224.0.0.0/8
```

# 7. 常用脚本

## 7.1. Nginx健康检测脚本

在Nginx配置目录下（/etc/nginx/conf.d/）增加health.conf的配置文件,该配置文件用于配置Nginx health的接口。

```shell
server {
    listen       80  default_server;
    server_name  localhost;
    default_type text/html;
    return 200 'Health';  
}
```

Nginx健康检测脚本：/etc/keepalived/scripts/check_nginx.sh

### 7.1.1. 检查接口调用是否为200

```shell
#!/bin/sh
set -x

timeout=30 #指定默认30秒没返回200则为非健康，该值可根据实际调整
 
if [ -n ${timeout} ];then
	httpcode=`curl -sL -w %{http_code} -m ${timeout} http://localhost -o /dev/null`
else
	httpcode=`curl -sL -w %{http_code}  http://localhost -o /dev/null`
fi

if [ ${httpcode} -ne 200 ];then
        echo `date`':  nginx is not healthy, return http_code is '${httpcode} >> /etc/keeperalived/keepalived.log
        killall keepalived
        exit 1
else
        exit 0
fi

```

### 7.1.2. 检查Nginx进程是否运行

```shell
#!/bin/sh
if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        echo "$(date) nginx pid not found">>/etc/keepalived/keepalived.log
        killall keepalived
fi

```

## 7.2. Keepalived状态通知脚本

```shell
#!/bin/bash
set -x

warn_receiver=$1
ip=$(ifconfig bond0|grep inet |awk '{print $2}')
warningInfo="${ip}_keepalived_changed_status_to_$1"
warn-report --user admin --key=xxxx --target=${warn_receiver} ${warningInfo}
echo $(date)  $1 >> /etc/keepalived/status

```

**说明：**

1. ip获取本机IP，本例中IP获取是bond0的IP，不同机器网卡名称不同需要修改为对应网卡名称。
2. 告警工具根据自己指定。





# 8.详细配置说明

keepalived只有一个配置文件`/etc/keepalived/keepalived.conf`。

里面主要包括以下几个配置区域，分别是:

- global_defs
- static_ipaddress
- static_routes
- vrrp_script
- vrrp_instance
- virtual_server

## 1. global_defs区域

主要是配置故障发生时的通知对象以及机器标识。

```bash
global_defs {
    notification_email {        # notification_email 故障发生时给谁发邮件通知
        a@abc.com
        b@abc.com
        ...
    }
    notification_email_from alert@abc.com  # notification_email_from 通知邮件从哪个地址发出
    smtp_server smtp.abc.com    # smpt_server 通知邮件的smtp地址
    smtp_connect_timeout 30     # smtp_connect_timeout 连接smtp服务器的超时时间
    enable_traps                # enable_traps 开启SNMP陷阱（Simple Network Management Protocol）
    router_id host163 # router_id 标识本节点的字条串，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
```

## 2. static_ipaddress和static_routes区域[可忽略]

static_ipaddress和static_routes区域配置的是是本节点的IP和路由信息。如果你的机器上已经配置了IP和路由，那么这两个区域可以不用配置。其实，一般情况下你的机器都会有IP地址和路由信息的，因此没必要再在这两个区域配置。

```bash
static_ipaddress {
    10.210.214.163/24 brd 10.210.214.255 dev eth0
    ...
}
static_routes {
    10.0.0.0/8 via 10.210.214.1 dev eth0
    ...
}
```

## 3. vrrp_script区域

用来做健康检查的，当时检查失败时会将vrrp_instance的priority减少相应的值。

```bash
vrrp_script chk_http_port {   
    script "</dev/tcp/127.0.0.1/80"       #一句指令或者一个脚本文件，需返回0(成功)或非0(失败)，keepalived以此为依据判断其监控的服务状态。
    interval 1    #健康检查周期
    weight -10   # 优先级变化幅度，如果script中的指令执行失败，那么相应的vrrp_instance的优先级会减少10个点。
}
```

## 4. vrrp_instance和vrrp_sync_group区域

vrrp_instance用来定义对外提供服务的VIP区域及其相关属性。

vrrp_rsync_group用来定义vrrp_intance组，使得这个组内成员动作一致。

```bash
vrrp_sync_group VG_1 {  #监控多个网段的实例
    group {
        inside_network   # name of vrrp_instance (below)
        outside_network  # One for each moveable IP.
        ...
    }
    notify_master /path/to_master.sh      # notify_master表示切换为主机执行的脚本
    notify_backup /path/to_backup.sh      # notify_backup表示切换为备机师的脚本
    notify_fault "/path/fault.sh VG_1"    # notify_fault表示出错时执行的脚本
    notify /path/notify.sh  # notify表示任何一状态切换时都会调用该脚本，且在以上三个脚本执行完成之后进行调用
    smtp_alert  # smtp_alert 表示是否开启邮件通知（用全局区域的邮件设置来发通知）
}

vrrp_instance VI_1 {
    state MASTER # state MASTER或BACKUP，当其他节点keepalived启动时会将priority比较大的节点选举为MASTER，因此该项其实没有实质用途。
    interface eth0  # interface 节点固有IP（非VIP）的网卡，用来发VRRP包
    use_vmac    dont_track_primary # use_vmac 是否使用VRRP的虚拟MAC地址，dont_track_primary 忽略VRRP网卡错误（默认未设置）
    track_interface {# track_interface 监控以下网卡，如果任何一个不通就会切换到FALT状态。（可选项）
        eth0
        eth1
    }
    #mcast_src_ip 修改vrrp组播包的源地址，默认源地址为master的IP
    mcast_src_ip    lvs_sync_daemon_interface eth1 #lvs_sync_daemon_interface 绑定lvs syncd的网卡
    garp_master_delay 10  # garp_master_delay 当切为主状态后多久更新ARP缓存，默认5秒
    virtual_router_id 1   # virtual_router_id 取值在0-255之间，用来区分多个instance的VRRP组播， 同一网段中virtual_router_id的值不能重复，否则会出错
    priority 100 #priority用来选举master的，根据服务是否可用，以weight的幅度来调整节点的priority，从而选取priority高的为master，该项取值范围是1-255（在此范围之外会被识别成默认值100）
    advert_int 1 # advert_int 发VRRP包的时间间隔，即多久进行一次master选举（可以认为是健康查检时间间隔）
    authentication { # authentication 认证区域，认证类型有PASS和HA（IPSEC），推荐使用PASS（密码只识别前8位）
        auth_type PASS  #认证方式
        auth_pass 12345678  #认证密码
    }

    virtual_ipaddress { # 设置vip
        10.210.214.253/24 brd 10.210.214.255 dev eth0
        192.168.1.11/24 brd 192.168.1.255 dev eth1
    }

    virtual_routes { # virtual_routes 虚拟路由，当IP漂过来之后需要添加的路由信息
        172.16.0.0/12 via 10.210.214.1
        192.168.1.0/24 via 192.168.1.1 dev eth1
        default via 202.102.152.1
    }

    track_script {
        chk_http_port
    }

    nopreempt # nopreempt 允许一个priority比较低的节点作为master，即使有priority更高的节点启动
    preempt_delay 300 # preempt_delay master启动多久之后进行接管资源（VIP/Route信息等），并提是没有nopreempt选项
    debug
    notify_master|    notify_backup|    notify_fault|    notify|    smtp_alert
}
```

## 5. virtual_server_group和virtual_server区域

virtual_server_group一般在超大型的LVS中用到，一般LVS用不到这东西。

```bash
virtual_server IP Port {
    delay_loop    # delay_loop 延迟轮询时间（单位秒）
    lb_algo rr|wrr|lc|wlc|lblc|sh|dh  # lb_algo 后端调试算法（load balancing algorithm）
    lb_kind NAT|DR|TUN  # lb_kind LVS调度类型NAT/DR/TUN
    persistence_timeout    #会话保持时间
    persistence_granularity  #lvs会话保持粒度 
    protocol TCP  #使用的协议
    ha_suspend
    virtualhost    # virtualhost 用来给HTTP_GET和SSL_GET配置请求header的
    alpha 
    omega
    quorum   
    hysteresis   
    quorum_up|   
    quorum_down|   
    sorry_server  #备用机，所有realserver失效后启用 
    real_server{   # real_server 真正提供服务的服务器
        weight 1 # 默认为1,0为失效       
        inhibit_on_failure #在服务器健康检查失效时，将其设为0，而不是直接从ipvs中删除
        notify_up|       # real server宕掉时执行的脚本
        notify_down|     # real server启动时执行的脚本
        # HTTP_GET|SSL_GET|TCP_CHECK|SMTP_CHECK|MISC_CHECK
        TCP_CHECK {
            connect_timeout 3 #连接超时时间
            nb_get_retry 3 #重连次数
            delay_before_retry 3 #重连间隔时间
            connect_port 23  #健康检查的端口的端口
            bindto  
        }
        
        HTTP_GET|SSL_GET {
            url {# 检查url，可以指定多个
                path          # path 请求real serserver上的路径  
                digest        # 用genhash算出的摘要信息       
                status_code   # 检查的http状态码       
                }
            connect_port         # connect_port 健康检查，如果端口通则认为服务器正常     
            connect_timeout      # 超时时长       
            nb_get_retry         # 重试次数
            delay_before_retry   #  下次重试的时间延迟  
         }
       
         SMTP_CHECK {
            host {
              connect_ip
              connect_port #默认检查25端口
              bindto
            }
            connect_timeout 5
            retry 3
            delay_before_retry 2
            helo_name | #smtp helo请求命令参数，可选
         }
         
         MISC_CHECK {
            misc_path | #外部脚本路径
            misc_timeout #脚本执行超时时间
            misc_dynamic #如设置该项，则退出状态码会用来动态调整服务器的权重，返回0 正常，不修改；返回1，
 #检查失败，权重改为0；返回2-255，正常，权重设置为：返回状态码-2
          }
    }
}
```





# 9. 常用命令

## 9.1. 查看当前VIP在哪个节点上

```bash
# 查看VIP是否在筛选结果中
ip addr show|grep "scope global"

# 或者 
ip addr show|grep {vip}
```

## 9.2. 查看keepalived的日志

```bash
tail /var/log/messages
```

## 9.3. 抓包命令

```bash
# 抓包
tcpdump -nn vrrp
# 可以用这条命令来查看该网络中所存在的vrid
tcpdump -nn -i any net 224.0.0.0/8
```

```bash
# tcpdump -nn -i any net 224.0.0.0/8
# tcpdump -nn vrrp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:40:00.576387 IP 192.168.98.57 > 224.0.0.18: VRRPv2, Advertisement, vrid 9, prio 99, authtype simple, intvl 1s, length 20
14:40:01.577605 IP 192.168.98.57 > 224.0.0.18: VRRPv2, Advertisement, vrid 9, prio 99, authtype simple, intvl 1s, length 20
14:40:02.578429 IP 192.168.98.57 > 224.0.0.18: VRRPv2, Advertisement, vrid 9, prio 99, authtype simple, intvl 1s, length 20
14:40:03.579605 IP 192.168.98.57 > 224.0.0.18: VRRPv2, Advertisement, vrid 9, prio 99, authtype simple, intvl 1s, length 20
14:40:04.580443 IP 192.168.98.57 > 224.0.0.18: VRRPv2, Advertisement, vrid 9, prio 99, authtype simple, intvl 1s, length 20
```

## 9.4. VIP操作

```bash
# 解绑VIP
ip addr del  dev 
# 绑定VIP
ip addr add  dev 
```

## 9.5. keepalived 切 VIP

例如将 A 机器上的 VIP 迁移到B 机器上。

### 9.5.1. 停止keepalived服务

停止被迁移的机器（A机器）的keepalived服务。

```bash
systemctl stop keepalived
```

### 9.5.2. 查看日志

解绑 A机器 VIP的日志

```bash
Sep 19 14:28:09 localhost systemd: Stopping LVS and VRRP High Availability Monitor...
Sep 19 14:28:09 localhost Keepalived[45705]: Stopping
Sep 19 14:28:09 localhost Keepalived_vrrp[45707]: VRRP_Instance(twemproxy) sent 0 priority
Sep 19 14:28:09 localhost Keepalived_vrrp[45707]: VRRP_Instance(twemproxy) removing protocol VIPs.
Sep 19 14:28:09 localhost Keepalived_healthcheckers[45706]: Stopped
Sep 19 14:28:10 localhost Keepalived_vrrp[45707]: Stopped
Sep 19 14:28:10 localhost Keepalived[45705]: Stopped Keepalived v1.3.5 (03/19,2017), git commit v1.3.5-6-g6fa32f2
Sep 19 14:28:10 localhost systemd: Stopped LVS and VRRP High Availability Monitor.
Sep 19 14:28:10 localhost ntpd[1186]: Deleting interface #10 bond0, 192.168.99.9#123, interface stats: received=0, sent=0, dropped=0, active_time=6755768 secs
```

绑定 B 机器 VIP的日志

```bash
Sep 17 17:20:25 localhost systemd: Starting LVS and VRRP High Availability Monitor...
Sep 17 17:20:26 localhost Keepalived[34566]: Starting Keepalived v1.3.5 (03/19,2017), git commit v1.3.5-6-g6fa32f2
Sep 17 17:20:26 localhost Keepalived[34566]: Opening file '/etc/keepalived/keepalived.conf'.
Sep 17 17:20:26 localhost Keepalived[34568]: Starting Healthcheck child process, pid=34569
Sep 17 17:20:26 localhost Keepalived[34568]: Starting VRRP child process, pid=34570
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Registering Kernel netlink reflector
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Registering Kernel netlink command channel
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Registering gratuitous ARP shared channel
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Opening file '/etc/keepalived/keepalived.conf'.
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Truncating auth_pass to 8 characters
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: VRRP_Instance(twemproxy) removing protocol VIPs.
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: Using LinkWatch kernel netlink reflector...
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: VRRP_Instance(twemproxy) Entering BACKUP STATE
Sep 17 17:20:26 localhost Keepalived_vrrp[34570]: VRRP sockpool: [ifindex(4), proto(112), unicast(0), fd(10,11)]
Sep 17 17:20:26 localhost systemd: Started LVS and VRRP High Availability Monitor.
Sep 17 17:20:26 localhost kernel: IPVS: Registered protocols (TCP, UDP, SCTP, AH, ESP)
Sep 17 17:20:26 localhost kernel: IPVS: Connection hash table configured (size=4096, memory=64Kbytes)
Sep 17 17:20:26 localhost kernel: IPVS: Creating netns size=2192 id=0
Sep 17 17:20:26 localhost kernel: IPVS: Creating netns size=2192 id=1
Sep 17 17:20:26 localhost kernel: IPVS: ipvs loaded.
Sep 17 17:20:26 localhost Keepalived_healthcheckers[34569]: Opening file '/etc/keepalived/keepalived.conf'.
```

# 10. 指定keepalived的输出日志文件

## 10.1. 修改 `/etc/sysconfig/keepalived`

将`KEEPALIVED_OPTIONS="-D"`改为`KEEPALIVED_OPTIONS="-D -d -S 0"`。

```bash
# Options for keepalived. See `keepalived --help' output and keepalived(8) and
# keepalived.conf(5) man pages for a list of all options. Here are the most
# common ones :
#
# --vrrp               -P    Only run with VRRP subsystem.
# --check              -C    Only run with Health-checker subsystem.
# --dont-release-vrrp  -V    Dont remove VRRP VIPs & VROUTEs on daemon stop.
# --dont-release-ipvs  -I    Dont remove IPVS topology on daemon stop.
# --dump-conf          -d    Dump the configuration data.
# --log-detail         -D    Detailed log messages.
# --log-facility       -S    0-7 Set local syslog facility (default=LOG_DAEMON)
#

KEEPALIVED_OPTIONS="-D -d -S 0"
```

## 10.2. 修改rsyslog的配置 /etc/rsyslog.conf

在/etc/rsyslog.conf  添加 keepalived的日志路径

```bash
vi /etc/rsyslog.conf
...
# keepalived log
local0.*                                                /etc/keepalived/keepalived.log
```

## 10.3. 重启rsyslog和keepalived

```bash
# 重启rsyslog
systemctl restart rsyslog
 
#  重启keepalived
systemctl restart keepalived
```

# 11. Troubleshooting

## 11.1. virtual_router_id 同网段重复

日志报错如下：

```bash
Mar 09 21:28:28 k8s4 Keepalived_vrrp[8548]: bogus VRRP packet received on eth0 !!!
Mar 09 21:28:28 k8s4 Keepalived_vrrp[8548]: VRRP_Instance(VI-kube-master) ignoring received advertisment...
Mar 09 21:28:43 k8s4 Keepalived_vrrp[8548]: ip address associated with VRID not present in received packet : 192.168.1.10
Mar 09 21:28:43 k8s4 Keepalived_vrrp[8548]: one or more VIP associated with VRID mismatch actual MASTER advert
```

解决方法:

同一网段内LB节点配置的 `virtual_router_id` 值有重复了，选择一个不重复的0~255之间的值，可以用以下命令查看已存在的vrid。

```bash
tcpdump -nn -i any net 224.0.0.0/8
```

## 11.2. Operation not permitted

问题：

两台主备机器都绑定了VIP，查看日志如下：

```bash
Sep 28 14:28:37 node Keepalived_vrrp[1686]: (VI_1): send advert error 1 (Operation not permitted)
Sep 28 14:28:39 node Keepalived_vrrp[1686]: (VI_1): send advert error 1 (Operation not permitted)
```

原因：

由于iptables vrrp协议没有放通，导致keepalived直接无法互相探测选主。

解决方法：

添加iptabels vrrp协议规则

```bash
iptables -A INPUT -p vrrp -j ACCEPT
iptables -A OUTPUT -p vrrp -j ACCEPT
```

持久化iptables规则，添加规则到文件中/etc/sysconfig/iptables

```bash
# vi /etc/sysconfig/iptables

-A INPUT -p vrrp -j ACCEPT
-A OUTPUT -p vrrp -j ACCEPT
```



