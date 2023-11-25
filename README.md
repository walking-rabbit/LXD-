# 服务器搭建问题指南2023
>参考网站:<br>
>https://xungejiang.com/2022/07/14/lxd-new/<br>
>https://www.wangt.cc/2021/09/%E5%9F%BA%E4%BA%8Elxd%E6%90%AD%E5%BB%BA%E5%A4%9A%E4%BA%BA%E5%85%B1%E7%94%A8gpu%E6%9C%8D%E5%8A%A1%E5%99%A8%EF%BC%8C%E7%AE%80%E5%8D%95%E6%98%93%E7%94%A8%EF%BC%8C%E5%85%A8%E7%BD%91%E6%9C%80%E8%AF%A6/<br>
>https://zhuanlan.zhihu.com/p/421271405

## 宿主机相关内容
Ubuntu 20.04 安装时，要注意Ubuntu系统的引导方式，若选择uefi引导方式，需要更改BIOS的CMS，使得系统首选uefi引导

<br>

**固定内核版本,防止内核升级造成Nvidia driver不可用**

`sudo apt-mark hold linux-image-generic linux-headers-generic`

<br>

**安装宿主机驱动Nvidia driver时，使用软件包下载安装。若安装失败，注意更改BIOS的安全启动菜单**
```
sudo sh ./NVIDIA-Linux-x86_64-530.30.02.run
nvidia-smi
```
>安装驱动时，参考 https://xungejiang.com/2019/10/08/ubuntu-gpu-driver/

<br>

**注意不要配置宿主机网络**

<br>

**安装lxd,zfs和bridge-utils**
```
sudo snap install lxd
sudo apt install zfsutils-linux bridge-utils
```
<br>

## LXD初始化
**初始化命令**

`sudo lxd init`
> lxd的zfs不挂载到其他硬盘上，按照`sudo lxd init`的默认值，zfs存储池建立在4T的SSD上，lxd将此池作为容器的默认创建池<br>
> 通过命令`lxc storage list`查看创建的存储池<br>
> zfs参考资料 https://linux.cn/article-8750-1.html
>> 我的理解：所有容器都在zfs存储池分配的SSD的空间中开辟

<br>

**lxd初始化过程中关键选项**
```
Create a new ZFS pool? (yes/no) [default=yes]: yes
Would you like to use an existing block device? (yes/no) [default=no]: no
Size in GB of the new loop device (1GiB minimum) [default=30GiB]: 100GiB
```
>其他选项默认即可<br>
>网络配置采用创建桥接网络，宿主机为公网ip 10.193.0.11，其他容器为局域网ip，访问每个容器通过公网ip+端口号的形式<br>
>通过命令`lxc network list`能够查看创建的桥接网络

<br>

## 创建容器 seulab
**通过命令`lxc launch ubuntu:20.04 seulab` 创建名为seulab的容器**
> 容器相关命令：容器列表，进入容器，退出容器，删除容器，停止容器，重启容器
>```
> lxc list  
> lxc exec seulab bash  
> exit  
> lxc delete seulab  
> lxc stop seulab  
> lxc restart seulab  
>```

<br>

**创建共享文件夹**
```
sudo lxc config set seulab security.privileged true
sudo lxc config device add seulab ShareData disk source=/media/seulab/data1/dockershare path=/home/ubuntu/ShareData
```
>服务器将机械硬盘中的dockershare作为用户的共享的数据文件夹<br>

`sudo lxc config set seulab security.privileged false`
>`sudo lxc config set seulab security.privileged false` 该命令很重要，如果不运行，容器重启后将无法使用可视化界面

<br>

**添加GPU硬件**

`lxc config device add seulab gpu gpu`

<br>

**配置网络**
```
sudo lxc config device add seulab proxy1 proxy listen=tcp:10.193.0.11:6002 connect=tcp:10.18.100.xxx:22 bind=host
sudo lxc config device add seulab proxy0 proxy listen=tcp:10.193.0.11:6003 connect=tcp:10.18.100.xxx:3389 bind=host
sudo lxc config device add seulab proxy2 proxy listen=tcp:10.193.0.11:6004 connect=tcp:10.18.100.xxx:6081 bind=host
```
> 10.193.0.11为宿主机ip，10.18.100.xxx 为 `lxc list`后seulab容器的ip，22为ssh端口，3389为rdp端口，6081为可视化界面端口

`sudo lxc config device list seulab`
>查看seulab容器添加的设备

<br>

## 容器内的环境配置

**通过`lxc exec seulab bash`进入容器，容器内的环境配置在默认用户ubuntu的权限下继续**
```
passwd ubuntu
login ubuntu
```
>设置ubuntu账户，账户密码设置为000000

<br>

**更改容器内的软件源，校园网下使用seu的软件源，速度较快**
```
sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak`<br>
sudo vim /etc/apt/sources.list

deb http://mirrors.seu.edu.cn/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.seu.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.seu.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb http://mirrors.seu.edu.cn/ubuntu/ focal-security main restricted universe multiverse
```

<br>

**依次安装Nvidia driver，Anaconda，Vscode，Cuda**
```
sudo sh ./NVIDIA-Linux-x86_64-530.30.02.run  --no-kernel-module
nvidia-smi

bash Anaconda3-2021.11-Linux-x86_64.sh 

sudo apt install ./code_1.84.1-1699275408_amd64.deb 

sudo apt install gcc g++ make
sudo sh cuda_12.1.0_530.30.02_linux.run
sudo vim ~/.bashrc
export PATH=/usr/local/cuda-12.1/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
nvcc -V
```

<br>

**配置SSH连接**
```
vim /etc/ssh/sshd_config

PermitRootLogin yes
PasswordAuthentication yes

/etc/init.d/ssh restart
```

<br>

## 安装可视化界面

```
sudo apt install xfce4
sudo apt install tigervnc-standalone-server
```

<br>

**设置远程桌面密码 `vncpasswd`**
>远程桌面密码默认设置为000000<br>
>远程桌面密码为网页端登录密码，ubuntu账号密码为ssh登录密码

<br>

**创建开机启动图形化界面脚本**
```
nano ~/.vnc/xstartup

#!/bin/bash
startxfce4 &
```

<br>

**创建vnc开机启动方法**
```
sudo nano /etc/systemd/system/vncserver@.service


[Unit]
Description=Systemd VNC server startup script for Ubuntu 20.04
After=syslog.target network.target

[Service]
Type=forking
User=ubuntu
ExecStart=/usr/bin/vncserver -depth 24 -geometry 1920x1080 :%i

[Install]
WantedBy=multi-user.target
```

<br>

**系统重新加载服务列表 `sudo systemctl daemon-reload`**

**启动开机自启动服务 `sudo systemctl enable vncserver@1.service`**

**启动vnc服务 `sudo systemctl start vncserver@1.service`**
>`sudo systemctl status vncserver@1.service` 检查未正常启动原因

<br>

**安装配置novnc**
```
sudo snap install novnc
sudo snap set novnc services.n6081.listen=6081 services.n6081.vnc=localhost:5901
```
>创建一个侦听6081端口，并将6081连接到VNC服务器，VNC服务器在localhost的5901端口上运行
<br>

## 管理相关容器
**基于seulab容器，创建lab快照 `sudo lxc snapshot seulab lab`**

**查看容器的快照 `lxc info seulab`**

**用快照生成容器XXX `sudo lxc copy seulab/lab XXX`**

<br>

**删除容器XXX proxy配置**
```
sudo lxc config device remove XXX proxy1
sudo lxc config device remove XXX proxy0
sudo lxc config device remove XXX proxy2

sudo lxc config edit XXX
sudo lxc config device list XXX
```
>copy后XXX容器的proxy配置是seulab中的配置，需要重新分配

<br>

**查看容器XXX的新ip地址,即lxc list中的IPV4,假设为10.18.100.xxx**
```
sudo lxc start XXX
sudo lxc list
```
<br>

**重新给容器XXX分配端口**
```
sudo lxc config device add XXX proxy1 proxy listen=tcp:10.193.0.11:60n2 connect=tcp:10.18.100.xxx:22 bind=host
sudo lxc config device add XXX proxy0 proxy listen=tcp:10.193.0.11:60n3 connect=tcp:10.18.100.xxx:3389 bind=host
sudo lxc config device add XXX proxy2 proxy listen=tcp:10.193.0.11:60n4 connect=tcp:10.18.100.xxx:6081 bind=host
```
>初始容器seulab 10.193.0.11的三个端口为6002，6003，6004，每次新建容器后这三个端口需要+10,若XXX为从seulab复制创建的第一个容器，则对应的三个端口为6012，6013，6014<br>
>若从seulab创建第n个容器，则该容器的三个端口分别对应n*10+2/3/4
>>第n个容器不包括初始容器seulab

>10.18.100.xxx为容器XXX的新ip地址

<br>

**设置ssh密码**
```
sudo lxc exec XXX bash
passwd ubuntu
```
>复制XXX容器后，需要进入容器创建用户ubuntu的密码000000，用于ssh远程连接<br>
>若要更改可视化界面的登录密码，运行`vncpasswd`

<br>

**以从seulab复制创建的第一个容器为例**

http://10.193.0.11:6014/vnc.html 是接入校园网的情况下，浏览器中的可视化界面，网页初始登录密码为000000，更改密码`vncpasswd`

`ssh -p 6012 ubuntu@10.193.0.11` 是ssh的远程连接，登录初始密码是000000，更改密码`passwd ubuntu`

<br>

**对于第n个容器**

可视化界面网址 http://10.193.0.11:60n4/vnc.html

ssh远程连接 `ssh -p 60n2 ubuntu@10.193.0.11`
>第n个容器不包括seulab








