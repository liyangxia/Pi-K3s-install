# Raspberry Pi 4B部署安装K3s集群

## 写在前面

首先很高兴你能找到这里，因为如果你要使用树莓派4B部署一个简单的K3s集群，看这一个项目就足够了，可以为你省下非常多的时间，这里有一些约定俗成需要注意：

1. 我们鼓励知识的分享，但拒绝复制粘贴的方式，那样会导致求知者的迷茫，请尽可能附带本文链接而不是直接复制粘贴

2. 要做到适应大多数人的实际需求其实很难，所以这里仅使用一个Server，两个Agent的部署方式，比较适用于Agent数量不超过10台的环境

如果你对文章有疑问，欢迎来[Issues](https://github.com/liyangxia/Pi-K3s-install/issues)讨论

关于一些琐碎的文字，可以查看[Tips](./TIPS.md)，这里写了一些FAQ

## 前言

树莓派已经发布到4B版本我才购买第一台，主要是因为生产环节都是大型服务器集群，不适用小型化环境，也就没有购买的欲望，但是因为最近开始关注工业互联网领域，越了解越觉得大型计算集群可能并不适合工业环境，排除AI和大数据计算需要外，工控其实几块树莓派就能解决。基于此购入了几块树莓派，尝试使用最经济有效的方案实现一个工业环境的设备控制

我使用的环境列表如下：

- Raspberry Pi 4B with 8G *1
- Raspberry Pi 4B with 4G *2
- TP-Link TL-SG1005D Ethernet Switch *1
- TP-Link CAT6 cable *1M

系统准备的是CentOS7 ARM32 (armhfp)，每天都和她打交道，已经非常熟悉了，可以减少很大一部分安全风险和学习成本，如使用其他系统遇到了问题也欢迎来[Issues](https://github.com/liyangxia/Pi-K3s-install/issues)讨论

## 环境准备

物理环境的准备比较简单，仅需要注意尽可能使用5V3A的电源，推荐树莓派官方的电源，如果有电源管理的经验自己做一个也是可以的，如果有实现UPS的需求，可以准备一块多口5V3A且支持边冲边放的电源，品牌找可信的

系统可以到[这个页面](https://www.centos.org/centos-linux/)下载，选择7(2009)-ARM32 (armhfp)，选择合适的软件源后下载对应的系统文件，树莓派使用的文件名是`CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-4-2009-sda.raw.xz`

烧录系统可以使用树莓派提供的工具，可以在[这个页面](https://www.raspberrypi.org/software/)下载

网线分别插入交换机、树莓派后插入SD卡，接上电源后等待一会系统就会自动启动

## 系统配置

系统系统后使用`ssh`登陆系统，用户名默认是`root`，密码默认是`centos`，默认端口是22

### 根分区扩容

进入系统后首先查看/root/目录下的README文件，这里记录着如何调整根分区的大小，对应的文件内容是这样：

```txt
== CentOS 7 userland ==

If you want to automatically resize your / partition, just type the following (as root user):
rootfs-expand
```

执行`rootfs-expand`命令扩容根分区，扩容完成后不要立即重启，使用`df -h`查看是否已经扩容，如果少了这步就会触发一个BUG，根分区没有被扩容，且下次再使用`rootfs-expand`命令无效，只能重新烧录系统

重启后执行一次系统更新，更新完成后再重启一次

### Cgroup配置

不管是K8s还是K3s，Cgroup都是必不可少的，这是一个高效使用命名空间管理systemd的工具，详细的介绍可以查看RedHat的[这篇文章](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/6/html/resource_management_guide/ch01)

树莓派上的CentOS7默认是没有开放Cgroup的，需要在/boot/cmdline.txt文件后面追加这些参数（⚠️注意不是换行）

```bash
#/boot/cmdline.txt追加的参数
cgroup_memory=1 cgroup_enable=memory

#我这台机器完整的配置参数
console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p3 rootfstype=ext4 elevator=deadline rootwait cgroup_memory=1 cgroup_enable=memory
```

之后重启即可开启Cgroup

### 静态IP、主机名、hosts配置

CentOS7的树莓派版本静态IP配置和传统方式并不相同，`/etc/sysconfig/network-scripts/`文件夹中并没有对应的路由表，建议在路由器中进行配置IP和MAC绑定，然后重启NetwordManager服务获取IP

部署K3s的时候主机名需要不一致，可以使用`hostnamectl set-hostname HOSTNAME`进行配置，配置完成后需要重启才能生效

之后在/etc/hosts文件中写明集群主机的IP和主机名

```txt
192.168.101.100 MacBook-Pro
192.168.101.101 ThinkPad-X13
192.168.101.150 pi-master
192.168.101.151 pi-node-a
192.168.101.152 pi-node-b
192.168.101.153 pi-node-c
```

### NTP Server服务

对于集群服务，时间同步是非常重要的，所有机器建议使用一台NTP Server服务器进行时间同步，确保集群内的机器时间都是一致的，可以查看下面的操作进行配置

```bash
#安装NTP服务
yum install ntp -y

#配置NTP服务地址
#注释21行到24行的内容
#25、26行新加入下面的NTP服务器地址
server time.cloudflare.com iburst

#国内地址可以使用腾讯云的NTP服务
time1.cloud.tencent.com 
time2.cloud.tencent.com 
time3.cloud.tencent.com
time4.cloud.tencent.com
time5.cloud.tencent.com

#启动NTP服务
systemctl enable --now ntpd

#查看NTP启动状态
ntpq -p
```

### 防火墙和SELinux配置

K3s默认所有出站流量都是允许的，入站规则和需要使用的端口有这些：

| 协议 | 端口 | 源 | 描述 |
| :-----| :---- | :----- | :----- |
| TCP | 6443 | K3s agent 节点 | Kubernetes API Server |
| UDP | 8472 | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要 |
| TCP | 10250 | K3s server 和 agent 节点 | Kubelet metrics |
| TCP | 2379-2380 | K3s server 节点 | 只有嵌入式 etcd 高可用才需要 |

如果仅仅是在内网提供服务，没有做对外，一劳永逸的办法是关闭防火墙和SELinux

```bash
#关闭防火墙
systemctl stop firewalld

#禁止开机启动
systemctl disable firewalld

#关闭SELinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

如果要做对外服务，建议还是做好安全组或防火墙设置，共有云环境可以查看对应的文档，私有云环境可以查看RedHat的文档管理防火墙和SELinux

[防火墙配置](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/security_guide/index)

[SELinux配置](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/index)

### 附加：Wi-Fi配置

我这里是直接连接的有线网，如果有使用Wi-Fi的需求，可以查看RedHat的[这篇文章](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/networking_guide/sec-using_the_networkmanager_command_line_tool_nmcli)，这里把简单的连接方式记录下

```bash
#查看当前使用的网络设备，我这里显示TYPE为wifi的设备是wlan0
nmcli device

#查看附近的Wi-Fi，如果知道SSID可以省略此步骤
nmcli device wifi list

#连接本地网络的无线网络，替换掉SSID\PASSWORD
#⚠️警告：如论何时，工作环境连接到开放网络都是不负责任的行为
nmcli device wifi connect SSID password PASSWORD

#查看当前连接的设备，上一步没有报错的话会出现一个新的连接名称
nmcli connection show

#启用Wi-Fi连接，替换掉SSID
nmcli connection up SSID

#附加项
#查看Wi-Fi连接的详细情况，替换掉SSID
nmcli connection show SSID

#如果需要修改某一项参数，可以这么修改，替换掉SSID\OPTION\VALUE
nmcli connection modify SSID OPTION VALUE

#删除Wi-Fi连接
#首先断开连接，替换掉SSID
nmcli connection down SSID

#接着删除连接，替换掉SSID
nmcli connection delete SSID
```

NetworkManager 是RedHat上自带的网络管理工具，可以管理有线网络或者Wi-Fi网络，甚至还可以实现网络分享，如果觉得使用很麻烦可以使用图形化工具`nmtui`

## 开始部署

### K3s Server部署

我这里的环境选择的是一台Server，多台Agent的管理方式，所以把8G的那台配置Server，如果使用K3s官网给出的安装命令直接执行会少了一些初始配置，我这里在安装的时候就指定一些参数，这样部署后就不用再手动调整

```bash
#官网给出的安装命令
curl -sfL https://get.k3s.io | sh -

#Server的安装选项完整列表
#https://docs.rancher.cn/docs/k3s/installation/install-options/server-config/_index

#我的配置参数
#K3S_NODE_NAME=pi-master 部署时定义节点名称为pi-master
curl -sfL https://get.k3s.io | K3S_NODE_NAME=pi-master sh -
```

待命令执行完成后查看启动状态，确认服务启动后获取token还有server_url，方便下面进行部署Agent

**token**在/var/lib/rancher/k3s/server/token

**server_url**在/etc/rancher/k3s/k3s.yaml

### K3s Agent部署

Agent的部署和Server唯一的区别在于Agent在安装服务的时候需要指定Server的token和server_url才能加入到Server，之后才能使用`kubectl get nodes`查看到对应的nodes

```bash
#Agent的安装选项完整列表
#https://docs.rancher.cn/docs/k3s/installation/install-options/agent-config/_index

#我的参数配置
#K3S_NODE_NAME=pi-node-a 部署时定义节点名称为pi-node-a
#每一台agent需要有不同的K3S_NODE_NAME
#K3S_TOKEN= Server的token，文件在查看上面的Server配置
#K3S_URL= Server的url，文件在查看上面的Server配置
#⚠️注意:需要替换成K3s Server的IP
curl -sfL https://get.k3s.io \
| K3S_TOKEN=*** \
K3S_URL=*** \
K3S_NODE_NAME=pi-node-a sh -
```

没有报错的话k3s-agent服务就会正常启动，此时回到Server端，使用`kubectl get nodes`就能查看到对应的node

其他Agent也可以这样配置，需要注意K3S_NODE_NAME需要保持不一致，如果不想这么麻烦也可以不用这个参数，默认是系统的`hostname`

当Server和Agent都部署好后，`kubectl get nodes`输出是这样的

```bash
[root@pi-master ~]# kubectl get nodes
NAME        STATUS   ROLES                  AGE   VERSION
pi-master   Ready    control-plane,master   15h   v1.20.4+k3s1
pi-node-a   Ready    <none>                 15h   v1.20.4+k3s1
pi-node-b   Ready    <none>                 14h   v1.20.4+k3s1
pi-node-c   Ready    <none>                 14h   v1.20.4+k3s1
```

## 待更新

因为Github Page默认只有一页，内容越来越多就导致主页内容特别长，所以近期将更新把所有内容迁移到[wiki](https://github.com/liyangxia/Pi-K3s/wiki)，预计开始时间：今日睡醒后
