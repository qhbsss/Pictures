# 使用Vxlan打通模拟桩小网ip和网管主机隧道时失效

## 问题场景：

在使用configVxlan.sh脚本为网管所在环境（本端）和模拟桩所在环境（远端）配置vxlan隧道时，在本端ping远端模拟桩创建的小网ip时，出现ping失败的情况，但是在本端ping远端vxlan隧道的ip时，却又能够ping通

## 问题分析

既然能够ping通远端vxlan隧道的ip，说明vxlan隧道配置没问题，但问题就在模拟桩创建的小网ip上了。原本的原理是在本端加入了远端模拟桩创建的小网ip的路由到vxlan上，同时远端的小网ip是创建在lo网卡下的虚拟网卡（比如lo:1，lo:2这样），为了查看哪边出了问题于是尝试使用tcpdump抓包查看问题。

在本端ping远端的小网ip并进行抓包，结果如下所示：
![](http://image.huawei.com/tiny-lts/v1/images/70ded0f2ff279a62519ee4c9215cc1d9_1042x525.png@900-0-90-f.png)
可以发现在ping远端小网ip时，在进行arp请求获取小网ip对应的mac地址时，无法获取成功。

同时在远端进行抓包：
![](http://image.huawei.com/tiny-lts/v1/images/edc4788ae85c52d190fabcc9c14b9ac7_657x411.png@900-0-90-f.png)
可以看到远端收到了本端发出的arp请求，但是并没有进行应答，所以是远端创建的小网ip不应答arp请求的问题。

## 解决方案

在询问gpt后，它给出了如下解决方案：

### 1. **检查路由和回路**

ARP代理启用后，对端应该能够通过`vxlan88`接口响应ARP请求，但需要确保路由正确，且没有回环问题。具体来说：

- 确认`172.10.21.x`网段的流量确实通过`vxlan88`接口发送，且没有在设备本地循环。
  
  查看路由表，确保路由指向`vxlan88`接口：
  
  ```bash
  ip route show table main
  ```
  
  确保没有指向`lo`接口的默认路由条目覆盖了`vxlan88`的路由。

### 2. **防火墙规则**

防火墙规则可能阻止了ARP请求或响应。确保对端设备上的防火墙配置允许ARP流量通过：

```bash
iptables -L -v -n
```

如果有任何规则阻止ARP流量或与`vxlan88`接口相关的流量，尝试暂时禁用防火墙进行测试：

```bash
iptables -F
```

### 3. **接口的ARP设置**

确认所有相关接口的ARP设置没有问题：

```bash
sysctl net.ipv4.conf.all.arp_accept=1
sysctl net.ipv4.conf.vxlan88.arp_accept=1
sysctl net.ipv4.conf.lo.arp_accept=1
```

此外，确保`arp_ignore`设置为0，以便设备在收到ARP请求时不论接口都能响应：

```bash
sysctl net.ipv4.conf.all.arp_ignore=0
sysctl net.ipv4.conf.vxlan88.arp_ignore=0
sysctl net.ipv4.conf.lo.arp_ignore=0
```

### 4. **手动静态ARP配置**

如果动态ARP请求确实无法通过，你可以考虑在两端手动添加静态ARP条目以确保通信。

在`10.136.156.128`设备上添加静态ARP：

```bash
arp -s 172.10.21.1 <对端设备的MAC地址>
```

在对端设备上也添加静态ARP：

```bash
arp -s 172.10.21.<x> <10.136.156.128设备的MAC地址>
```

### 5. **重启网络服务**

有时，网络堆栈可能需要重启以应用配置更改。你可以尝试重启网络服务：

```bash
systemctl restart networking
```

或者，简单地重启涉及的接口：

```bash
ip link set vxlan88 down
ip link set vxlan88 up
```

根据以上方案，在远端尝试进行如下命令后，发现能够成功在本端ping通远端的小网ip。

```bash
sysctl net.ipv4.conf.all.arp_accept=1
sysctl net.ipv4.conf.vxlan88.arp_accept=1
sysctl net.ipv4.conf.lo.arp_accept=1
sysctl net.ipv4.conf.all.arp_ignore=0
sysctl net.ipv4.conf.vxlan88.arp_ignore=0
sysctl net.ipv4.conf.lo.arp_ignore=0
ip link set vxlan88 down
ip link set vxlan88 up
```

## 总结：

gpt太强大了，但也要能够先定位到问题，这个问题可能是远端的应答arp请求的相关配置没有开启导致的，当然，也可能是重启了vxlan虚拟网卡解决的。

configVxlan.sh脚本如下：

```bash
#!/bin/bash

cur_path=`echo $(cd "$(dirname "$0")"; pwd)`


##本地配置vxlan
function initVxlan() {
    if [ $# != 1 ] ; then
        echo "need one parameters,local ip."
        return 1
    fi
	localIp=$1
	eth=$(ip addr show|grep "$localIp/"|awk '{print $NF}'|head -1)
	if [ -z "$eth" ] ; then
        echo "the ip is not existed in local."
        return 1
    fi
    ip link add vxlan88 type vxlan id 88 dstport 4789 dev $eth
    ip addr add 88.${localIp#*.}/8 dev vxlan88
    ip link set vxlan88 up
}


##删除vxlan
function delVxlan() {
    ip link del vxlan88
}


##添加转发表，用于隧道建连
function addRemote() {
    if [ $# != 1 ] ; then
        echo "need one parameters,remote ip."
        return 1
    fi
    remoteIp=$1
	bridge fdb append to 00:00:00:00:00:00 dst $remoteIp dev vxlan88
}


##显示建立隧道的IP
function showRemote() {
    bridge fdb show|grep vxlan88|grep "00:00:00:00:00:00"|awk '{print $5}'
}



##帮助日志
function errorhelp() {
    echo "参数有误，至少需要1个参数！"
    echo "参数1当前支持initVxlan、delVxlan、addRemote、showRemote"
	echo "当参数1为initVxlan时，参数2为本地ip"
	echo "当参数1为addRemote时，参数2为需要和远端建连的ip"
}


if [[ "showRemote" == $1 ]];then
    showRemote
elif [[ "delVxlan" == $1 ]];then
    delVxlan
elif [[ "addRemote" == $1 ]];then
    addRemote $2
elif [[ "initVxlan" == $1 ]];then
    initVxlan $2
else
    errorhelp
fi
```

