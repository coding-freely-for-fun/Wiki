# shadowsocks-quick-guide

## ss 服务器

### 工欲善其事，必先利其器

1. 租用一台国外的vps,(本人使用的是banwangong,原因便宜,小白一键免配置)。其它的厂家还有digitalocean，vultr，linode和lightsail。
2. 远程ssh工具

### 安装和配置

#### install

> 安装python实现，其他的还有go,c,c++等实现，详见[shadowsocks-server](http://www.shadowsocks.org/en/download/servers.html)

```sh
# Debian / Ubuntu:
$ apt-get install python-pip
$ pip install shadowsocks

# centos
$ yum install python-setuptools && easy_install pip
$ pip install shadowsocks
```

#### config file

|属性|描述|可选值|
|:---|:---|:----|
|server|服务器ip地址|自定义|
|server_port|Shadowsocks监听的端口，默认8388|自定义|
|local_address|本机IP，127.0.0.1.|回环地址|
|local_port|本机端口，默认1080|自定义|
|password|连接密码|自定义|
|timeout|超时时间|自定义|
|method|加密方法|推荐使用"aes-256-cfb"|
|fast_open|Reduces latency when turned on. Can only be used with kernel versions 3.7.1 or higher. Check your kernel version with umame -r|true, false|
|nameserver|Name servers for internal DNS resolver.|User determined|

```json
{
    "server":"192.0.0.1",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open": false
}
```

若是多用户模式，将server_port和password合并为port_password：

```json
{
"server":"",
"local_address":"127.0.0.1",
"local_port":1080,
"port_password":{
"8000":"123456", //对应端口设定不同的密码
"8001":"123456"
},
"timeout":300,
"method":"aes-256-cfb",
"fast_open":false
}
```

#### 运行

```sh
# 命令行启动关闭
$ ssserver -c /etc/shadowsocks/config.json -d start # 后台启动
$ ssserver -c /etc/shadowsocks/config.json -d stop # 后台停止

# 设置开机启动
## systemd

$ sudo vi /etc/systemd/system/shadowsocks-server.service
$ sudo systemctl start shadowsocks-server # 启动Shadowsocks
$ sudo systemctl enable shadowsocks-server # 设置开机启动Shadowsocks
```

vi写入下面的内容：

```sh
[Unit]
Description=Shadowsocks Server
After=network.target

[Service]
ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks/config.json
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

#### 打开防火墙

```sh
# iptables
iptables -4 -A INPUT -p tcp --dport 8388 -m comment --comment "Shadowsocks server listen port" -j ACCEPT

# UFW
ufw allow proto tcp to 0.0.0.0/0 port 8388 comment "Shadowsocks server listen port"

# FirewallD
firewall-cmd --permanent --zone=public --add-rich-rule='
    rule family="ipv4"
    port protocol="tcp" port="8388" accept'

    firewall-cmd --reload
```

### 优化

#### 内核参数优化

1. 将 Linux 内核升级到 3.5 或以上。

```sh
$ sudo apt update
$ sudo apt-cache showpkg linux-image  # 查看可用的Linux内核版本
$ sudo apt install linux-image-4.10.0-22-generic
$ sudo reboot
```

2. 增加系统文件描述符的最大限数

```sh
vi /etc/security/limits.conf
# 增加以下两行
# soft nofile 51200
# hard nofile 51200
```

3. 启动shadowsocks服务器之前，设置以下参数

```sh
$ sudo vim /etc/systemd/system/shadowsocks-server.service
# 在ExecStart前插入一行，内容为：
# ExecStartPre=/bin/sh -c 'ulimit -n 51200'
```
4. 调整内核参数

修改配置文件 `/etc/sysctl.conf`

```
fs.file-max = 51200
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.netdev_max_backlog = 250000
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = hybla

# 修改后执行 sysctl -p 使配置生效
```

#### 开启BBR

BBR: Google最新开发的TCP拥塞控制算法，目前有着较好的带宽提升效果，甚至不比老牌的锐速差。

BBR在Linux kernel 4.9引入。首先检查服务器kernel版本：

```sh
$ uname -r # 如果其显示版本在4.9.0之下，则需要升级Linux内核

$ sudo apt update
$ sudo apt-cache showpkg linux-image  # 查看可用的Linux内核版本
$ sudo apt install linux-image-4.10.0-22-generic
$ sudo reboot
$ sudo purge-old-kernels # 删除老的Linux内核：

# 运行lsmod | grep bbr，如果结果中没有tcp_bbr，则先运行：
$ modprobe tcp_bbr
$ echo "tcp_bbr" >> /etc/modules-load.d/modules.conf

$ echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
$ echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
$ sysctl -p

$ sysctl net.ipv4.tcp_available_congestion_control
$ sysctl net.ipv4.tcp_congestion_control
# 若均有bbr，则开启BBR成功。
```

##### 优化吞吐量

```sh
$ sudo vi /etc/sysctl.d/local.conf
```

vi 中写入以下内容

```sh
# max open files
fs.file-max = 51200
# max read buffer
net.core.rmem_max = 67108864
# max write buffer
net.core.wmem_max = 67108864
# default read buffer
net.core.rmem_default = 65536
# default write buffer
net.core.wmem_default = 65536
# max processor input queue
net.core.netdev_max_backlog = 4096
# max backlog
net.core.somaxconn = 4096

# resist SYN flood attacks
net.ipv4.tcp_syncookies = 1
# reuse timewait sockets when safe
net.ipv4.tcp_tw_reuse = 1
# turn off fast timewait sockets recycling
net.ipv4.tcp_tw_recycle = 0
# short FIN timeout
net.ipv4.tcp_fin_timeout = 30
# short keepalive time
net.ipv4.tcp_keepalive_time = 1200
# outbound port range
net.ipv4.ip_local_port_range = 10000 65000
# max SYN backlog
net.ipv4.tcp_max_syn_backlog = 4096
# max timewait sockets held by system simultaneously
net.ipv4.tcp_max_tw_buckets = 5000
# turn on TCP Fast Open on both client and server side
net.ipv4.tcp_fastopen = 3
# TCP receive buffer
net.ipv4.tcp_rmem = 4096 87380 67108864
# TCP write buffer
net.ipv4.tcp_wmem = 4096 65536 67108864
# turn on path MTU discovery
net.ipv4.tcp_mtu_probing = 1

net.ipv4.tcp_congestion_control = bbr
```

运行：

```sh
$ sysctl --system
$ sudo vim /etc/systemd/system/shadowsocks-server.service
# 在ExecStart前插入一行，内容为：
# ExecStartPre=/bin/sh -c 'ulimit -n 51200'
$ sudo systemctl daemon-reload # 重载shadowsocks-server.service服务
￥sudo systemctl restart shadowsocks-server # 重启Shadowsocks
```

#### 锐速

锐速是一款非常不错的TCP底层加速软件，可以非常方便快速地完成服务器网络的优化，配合 ShadowSocks 效果奇佳。目前锐速官方也出了永久免费版本，适用带宽20M、3000加速连接，个人使用是足够了。如果需要，先要在锐速官网注册个账户。

然后确定自己的内核是否在锐速的支持列表里，如果不在，请先更换内核，如果不确定，请使用 手动安装。

确定自己的内核版本在支持列表里，就可以使用以下命令快速安装了。

```sh
wget http://my.serverspeeder.com/d/ls/serverSpeederInstaller.tar.gz
tar xzvf serverSpeederInstaller.tar.gz
bash serverSpeederInstaller.sh
```

输入在官网注册的账号密码进行安装，参数设置直接回车默认即可，
最后两项输入 y 开机自动启动锐速，y 立刻启动锐速。之后可以通过lsmod查看是否有appex模块在运行。

到这里还没结束，我们还要修改锐速的3个参数，`vi /serverspeeder/etc/config`

```sh
rsc="1" #RSC网卡驱动模式
advinacc="1" #流量方向加速
maxmode="1" #最大传输模式
digitalocean vps的网卡支持rsc和gso高级算法，所以可以开启rsc="1"，gso="1"。
```

### net-speeder

net-speeder 原理非常简单粗暴，就是发包翻倍，这会占用大量的国际出口带宽，本质是损人利己，不建议使用。

#### Ubuntu/Debian 下安装依赖包

```sh
apt-get install libnet1
apt-get install libpcap0.8
apt-get install libnet1-dev
apt-get install libpcap0.8-dev
```

#### Centos 下安装依赖包

需要配置 epel 第三方源。下载 epel ：[http://dl.fedoraproject.org/pub/epel/](http://dl.fedoraproject.org/pub/epel/)。例如，Centos 7 x64：

```sh
wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
rpm -ivh epel-release-7-5.noarch.rpm
yum repolist
```

然后安装依赖包：

```
yum install libnet libpcap libnet-devel libpcap-devel
```

3. 下载官方的 tar.gz 压缩包。解压安装运行：

```sh
wget http://net-speeder.googlecode.com/files/net_speeder-v0.1.tar.gz 
tar zxvf net_speeder-v0.1.tar.gz
cd net_speeder
chmod 777 *
sh build.sh -DCOOKED
```

首先你需要知道你的网卡设备名，可以使用 ifconfig 查看。假设是eth0，那么运行方法是:

```sh
./net_speeder eth0 "ip"
killall net_speeder # 关闭 net-speeder
```

#### 开启TCP Fast Open

TCP Fast Open可以降低Shadowsocks服务器和客户端的延迟。实际上在上一步已经开启了TCP Fast Open，现在只需要在Shadowsocks配置中启用TCP Fast Open。

```sh
$ sudo vi /etc/shadowsocks/config.json
$ sudo systemctl restart shadowsocks-server
```

注意：TCP Fast Open同时需要客户端的支持，即客户端Linux内核版本为3.7.1及以上；你可以在Shadowsocks客户端中启用TCP Fast Open。

### socks转http/https代理方案

* [privoxy](https://www.privoxy.org/)
* [proxychains](https://github.com/haad/proxychains)
* [polipo](https://www.irif.fr/~jch/software/polipo/)

## ss 客户端

<div>
   <h3>Windows</h3>
   <p><strong>GUI Client</strong></p>
   <ul>
      <li>shadowsocks-win: <a href="https://github.com/shadowsocks/shadowsocks-windows/releases">GitHub</a></li>
      <li>Shadowsocks-Qt5: <a href="https://github.com/shadowsocks/shadowsocks-qt5/releases">GitHub</a></li>
      <li>
         Outline Windows
         <ul>
            <li><a href="https://github.com/Jigsaw-Code/outline-client/">GitHub</a></li>
            <li><a href="https://raw.githubusercontent.com/Jigsaw-Code/outline-releases/master/client/Outline-Client.exe">Direct Download</a></li>
         </ul>
      </li>
   </ul>
   <p><strong>Command-line Client</strong></p>
   <ul>
      <li><code>pip install shadowsocks</code></li>
   </ul>
</div>
<div>
   <h3>Mac OS X</h3>
   <p><strong>GUI Client</strong></p>
   <ul>
      <li>ShadowsocksX-NG: <a href="https://github.com/shadowsocks/ShadowsocksX-NG/releases">GitHub</a></li>
      <li>
         Outline macOS
         <ul>
            <li><a href="https://github.com/Jigsaw-Code/outline-client/">GitHub</a></li>
            <li><a href="https://itunes.apple.com/app/outline-app/id1356178125">App Store</a></li>
         </ul>
      </li>
   </ul>
   <p><strong>Command-line Client</strong></p>
   <ul>
      <li><code>pip install shadowsocks</code></li>
      <li><code>brew install shadowsocks-libev</code></li>
      <li><code>cpan Net::Shadowsocks</code></li>
   </ul>
</div>
<div>
   <h3>Linux</h3>
   <p><strong>GUI Client</strong></p>
   <ul>
      <li>Shadowsocks-Qt5: <a href="https://github.com/shadowsocks/shadowsocks-qt5/wiki/Installation">GitHub</a></li>
   </ul>
   <p><strong>Command-line Client</strong></p>
   <ul>
      <li><code>pip install shadowsocks</code></li>
      <li><code>apt-get install shadowsocks-libev</code></li>
      <li><code>cpan Net::Shadowsocks</code></li>
   </ul>
</div>
<div>
   <h3>Android</h3>
   <ul>
      <li>
         shadowsocks-android:
         <ul>
            <li><a href="https://play.google.com/store/apps/details?id=com.github.shadowsocks">Google Play</a> (<a href="https://play.google.com/apps/testing/com.github.shadowsocks">beta</a>)</li>
         </ul>
      </li>
      <li>
         Outline Android
         <ul>
            <li><a href="https://github.com/Jigsaw-Code/outline-client/">GitHub</a></li>
            <li><a href="https://play.google.com/store/apps/details?id=org.outline.android.client">Play Store</a></li>
            <li><a href="https://github.com/Jigsaw-Code/outline-releases/blob/master/client/Outline.apk?raw=true">Direct Download</a></li>
         </ul>
      </li>
   </ul>
</div>
<div>
   <h3>iOS</h3>
   <ul>
      <li>
         Wingy:
         <ul>
            <li><a href="https://itunes.apple.com/us/app/wingy-http-s-socks5-proxy-utility/id1178584911">App Store</a></li>
         </ul>
      </li>
      <li>
         MobileShadowSocks:
         <ul>
            <li><a href="http://apt.thebigboss.org/onepackage.php?bundleid=com.linusyang.shadowsocks">Big Boss</a></li>
         </ul>
      </li>
      <li>
         Outline iOS
         <ul>
            <li><a href="https://github.com/Jigsaw-Code/outline-client/">GitHub</a></li>
            <li><a href="https://itunes.apple.com/app/outline-app/id1356177741">App Store</a></li>
         </ul>
      </li>
   </ul>
</div>
<div >
   <h3>OpenWRT</h3>
   <ul>
      <li>shadowsocks-libev</li>
      <ul>
         <li><code>opkg install shadowsocks-libev</code></li>
      </ul>
      <li>shadowsocks-libev-polarssl</li>
      <ul>
         <li><code>opkg install shadowsocks-libev-polarssl</code></li>
      </ul>
   </ul>
</div>

---

[首页Wiki](../README.md)