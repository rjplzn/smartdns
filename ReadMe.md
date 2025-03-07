# SmartDNS

**[English](ReadMe_en.md)**

![SmartDNS](doc/smartdns-banner.png)  
SmartDNS是一个运行在本地的DNS服务器，SmartDNS接受本地客户端的DNS查询请求，从多个上游DNS服务器获取DNS查询结果，并将访问速度最快的结果返回给客户端，提高网络访问速度。
同时支持指定特定域名IP地址，并高性匹配，达到过滤广告的效果。  
与dnsmasq的all-servers不同，smartdns返回的是访问速度最快的解析结果。 (详细差异请看[FAQ](#faq)) 

支持树莓派，openwrt，华硕路由器，windows等设备。  

## 目录

1. [软件效果展示](#软件效果展示)
1. [特性](#特性)
1. [架构](#架构)
1. [使用](#使用)  
    1. [下载配套安装包](#下载配套安装包)
    1. [标准Linux系统安装](#标准linux系统安装树莓派x86_64系统)
    1. [openwrt/LEDE](#openwrtlede)
    1. [华硕路由器原生固件/梅林固件](#华硕路由器原生固件梅林固件)
    1. [optware/entware](#optwareentware)
    1. [Windows 10 WSL安装/WSL ubuntu](#windows-10-wsl安装wsl-ubuntu)
1. [配置参数](#配置参数)
1. [捐助](#donate)
1. [声明](#声明)
1. [FAQ](#faq)

## 软件效果展示

**阿里DNS**  
使用阿里DNS查询百度IP，并检测结果。  

```shell
pi@raspberrypi:~/code/smartdns_build $ nslookup www.baidu.com 223.5.5.5
Server:         223.5.5.5
Address:        223.5.5.5#53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com.
Name:   www.a.shifen.com
Address: 180.97.33.108
Name:   www.a.shifen.com
Address: 180.97.33.107

pi@raspberrypi:~/code/smartdns_build $ ping 180.97.33.107 -c 2
PING 180.97.33.107 (180.97.33.107) 56(84) bytes of data.
64 bytes from 180.97.33.107: icmp_seq=1 ttl=55 time=24.3 ms
64 bytes from 180.97.33.107: icmp_seq=2 ttl=55 time=24.2 ms

--- 180.97.33.107 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 24.275/24.327/24.380/0.164 ms
pi@raspberrypi:~/code/smartdns_build $ ping 180.97.33.108 -c 2
PING 180.97.33.108 (180.97.33.108) 56(84) bytes of data.
64 bytes from 180.97.33.108: icmp_seq=1 ttl=55 time=31.1 ms
64 bytes from 180.97.33.108: icmp_seq=2 ttl=55 time=31.0 ms

--- 180.97.33.108 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 31.014/31.094/31.175/0.193 ms
```

**smartdns**  
使用SmartDNS查询百度IP，并检测结果。

```shell
pi@raspberrypi:~/code/smartdns_build $ nslookup www.baidu.com
Server:         192.168.1.1
Address:        192.168.1.1#53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com.
Name:   www.a.shifen.com
Address: 14.215.177.39

pi@raspberrypi:~/code/smartdns_build $ ping 14.215.177.39 -c 2
PING 14.215.177.39 (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39: icmp_seq=1 ttl=56 time=6.31 ms
64 bytes from 14.215.177.39: icmp_seq=2 ttl=56 time=5.95 ms

--- 14.215.177.39 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 5.954/6.133/6.313/0.195 ms

```

从对比看出，smartdns找到访问www.baidu.com最快的IP地址，这样访问百度比阿里DNS速度快5倍。

## 特性

1. **多DNS上游服务器**  
   支持配置多个上游DNS服务器，并同时进行查询，即使其中有DNS服务器异常，也不会影响查询。  

1. **返回最快IP地址**  
   支持从域名所属IP地址列表中查找到访问速度最快的IP地址，并返回给客户端，提高网络访问速度。

1. **支持多种查询协议**  
   支持UDP，TCP，TLS, HTTPS查询，以及非53端口查询。

1. **特定域名IP地址指定**  
   支持指定域名的IP地址，达到广告过滤效果，避免恶意网站的效果。

1. **域名高性能后缀匹配**  
   支持域名后缀匹配模式，简化过滤配置，过滤20万条记录时间<1ms

1. **域名分流**  
   支持域名分流，不同类型的域名到不同的DNS服务器查询。

1. **Linux/Windows多平台支持**  
   支持标准Linux系统（树莓派），openwrt系统各种固件，华硕路由器原生固件。以及支持Windows 10 WSL (Windows Subsystem for Linux)。

1. **支持IPV4, IPV6双栈**  
   支持IPV4，IPV6网络，支持查询A, AAAA记录，支持双栈IP速度优化，并支持完全禁用IPV6 AAAA解析。

1. **高性能，占用资源少**  
   多线程异步IO模式，cache缓存查询结果。

## 架构

![Architecture](doc/architecture.png)

1. SmartDNS接收本地网络设备的DNS查询请求，如PC，手机的查询请求。  
2. SmartDNS将查询请求发送到多个上游DNS服务器，可采用标准UDP查询，非标准端口UDP查询，及TCP查询。  
3. 上游DNS服务器返回域名对应的Server IP地址列表。SmartDNS检测与本地网络访问速度最快的Server IP。  
4. 将访问速度最快的Server IP返回给本地客户端。  

## 使用

### 下载配套安装包  

--------------

下载配套版本的SmartDNS安装包，对应安装包配套关系如下。

|系统 |安装包|说明
|-----|-----|-----
|标准Linux系统(树莓派)| smartdns.xxxxxxxx.armhf.deb|支持树莓派Raspbian stretch，Debian 9系统。
|标准Linux系统(Armbian arm64)| smartdns.xxxxxxxx.arm64.deb|支持ARM64的Debian stretch，Debian 9系统。
|标准Linux系统(x86_64)| smartdns.xxxxxxxx.x86_64.tar.gz|支持x86_64 Linux 系统。
|Windows 10 WSL (ubuntu)| smartdns.xxxxxxxx.x86_64.tar.gz|支持Windows 10 WSL ubuntu系统。
|标准Linux系统(x86)| smartdns.xxxxxxxx.x86.tar.gz|支持x86系统。
|华硕原生固件(optware)|smartdns.xxxxxxx.mipsbig.ipk|支持MIPS大端架构的系统，如RT-AC55U, RT-AC66U.
|华硕原生固件(optware)|smartdns.xxxxxxx.mipsel.ipk|支持MIPS小端架构的系统。
|华硕原生固件(optware)|smartdns.xxxxxxx.arm.ipk|支持arm小端架构的系统，如RT-AC68U。
|Padavan|smartdns.xxxxxxx.mipselsf.ipk|padavan固件。
|openwrt 15.01|smartdns.xxxxxxxx.ar71xx.ipk|支持AR71XX MIPS系统。
|openwrt 15.01|smartdns.xxxxxxxx.ramips_24kec.ipk|支持MT762X等小端路由器
|openwrt 15.01(潘多拉)|smartdns.xxxxxxxx.mipsel_24kec_dsp.ipk|支持MT7620系列的潘多拉固件
|openwrt 15.01(潘多拉)|smartdns.xxxxxxxx.mips_74kc_dsp2.ipk|支持AR71xx系列的潘多拉固件
|openwrt 18.06|smartdns.xxxxxxxx.mips_24kc.ipk|支持AR71XX MIPS系统。
|openwrt 18.06|smartdns.xxxxxxxx.mipsel_24kc.ipk|支持MT726X等小端路由器
|openwrt 18.06|smartdns.xxxxxxxx.x86_64.ipk|支持x86_64路由器
|openwrt 18.06|smartdns.xxxxxxxx.i386_pentium4.ipk|支持x86路由器
|openwrt 18.06|smartdns.xxxxxxxxxxx.arm_cortex-a9.ipk|支持arm A9核心CPU的路由器
|openwrt 18.06|smartdns.xxxxxxxxx.arm_cortex-a7_neon-vfpv4.ipk|支持arm A7核心CPU的路由器
|openwrt LUCI|luci-app-smartdns.xxxxxxxxx.xxxx.all.ipk|openwrt管理统一界面

* openwrt系统CPU架构比较多，上述表格未列出所有支持系统，请查看CPU架构后下载。
* merlin梅林固件理论和华硕固件一致，所以根据硬件类型安装相应的ipk包即可。（梅林暂时未验证，有问题提交issue）
* CPU架构可在路由器管理界面找到，查看方法：
    登录路由器，点击`System`->`Software`，点击`Configuration` Tab页面，在opkg安装源中可找到对应软件架构，下载路径中可找到，如下，架构为ar71xx

    ```shell
    src/gz chaos_calmer_base http://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/generic/packages/base
    ```

* 或ssh登录系统后通过如下命令查询软件架构：

  * **openwrt系列命令**

    ```shell
    opkg print_architecture
    ```

  * **optware系列命令**

    ```shell
    ipkg print_architecture
    ```

  * **debian系列命令**

    ```shell
    dpkg --print-architecture
    ```

  * **例如**

    下面的查询结果`arch ar71xx 10`表示ar71xx系列架构，选择`smartdns.xxxxxxxx.ar71xx.ipk`安装包

    ```shell
    root@OpenWrt:~# opkg print_architecture
    arch all 1
    arch noarch 1
    arch ar71xx 10
    ```

* **请在Release页面下载：[点击此处下载](https://github.com/pymumu/smartdns/releases)**

```shell
https://github.com/pymumu/smartdns/releases
```

* 各种设备的安装步骤，请参考后面的章节。

### 标准Linux系统安装/树莓派/X86_64系统

--------------

1. 安装

    下载配套安装包`smartdns.xxxxxxxx.armhf.deb`，并上传到Linux系统中。 执行如下命令安装

    ```shell
    dpkg -i smartdns.xxxxxxxx.armhf.deb
    ```

    x86系统下载配套安装包`smartdns.xxxxxxxx.x86-64.tar.gz`, 并上传到Linux系统中。 执行如下命令安装

    ```shell
    tar zxf smartdns.xxxxxxxx.x86-64.tar.gz
    cd smartdns
    chmod +x ./install
    ./install -i
    ```

1. 修改配置

    安装完成后，可配置smartdns的上游服务器信息。具体配置参数参考`配置参数`说明。  
    一般情况下，只需要增加`server [IP]:port`, `server-tcp [IP]:port`配置项，
    尽可能配置多个上游DNS服务器，包括国内外的服务器。配置参数请查看`配置参数`章节。

    ```shell
    vi /etc/smartdns/smartdns.conf
    ```

1. 启动服务

    ```shell
    systemctl enable smartdns
    systemctl start smartdns
    ```

1. 将DNS请求转发的SmartDNS解析。

    修改本地路由器的DNS服务器，将DNS服务器配置为SmartDNS。
    * 登录到本地网络的路由器中，配置树莓派分配静态IP地址。
    * 修改WAN口或者DHCP DNS为树莓派IP地址。  
    注意：  
    I. 每款路由器配置方法不尽相同，请百度搜索相关的配置方法。  
    II.华为等路由器可能不支持配置DNS为本地IP，请修改PC端，手机端DNS服务器为树莓派IP。

1. 检测服务是否配置成功。

    使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
    看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

    ```shell
    pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
    Server:         192.168.1.1
    Address:        192.168.1.1#53

    Non-authoritative answer:
    0.0.0.0.in-addr.arpa  name = smartdns.
    ```

### openwrt/LEDE

--------------

1. 安装

    将软件使用winscp上传到路由器的/root目录，执行如下命令安装  

    ```shell
    opkg install smartdns.xxxxxxxx.xxxx.ipk
    opkg install luci-app-smartdns.xxxxxxxx.xxxx.all.ipk
    ```

1. 修改配置

    登录openwrt管理页面，打开`Services`->`SmartDNS`进行配置。
    * 在`Upstream Servers`增加上游DNS服务器配置，建议配置多个国内外DNS服务器。
    * 在`Domain Address`指定特定域名的IP地址，可用于广告屏蔽。

1. 启用服务

   SmartDNS服务生效方法有两种，`一种是直接作为主DNS服务`；`另一种是作为dnsmasq的上游`。  
   默认情况下，SmartDNS采用第一种方式。如下两种方式根据需求选择即可。

1. 启用方法一：作为主DNS(默认方案)

    * **启用smartdns的53端口重定向**

        登录路由器，点击`Services`->`SmartDNS`->`redirect`，选择`重定向53端口到SmartDNS`启用53端口转发。

    * **检测转发服务是否配置成功**

        使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
        看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

        ```shell
        pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
        Server:         192.168.1.1
        Address:        192.168.1.1#53

        Non-authoritative answer:
        0.0.0.0.in-addr.arpa  name = smartdns.
        ```

    * **界面提示重定向失败**

        * 检查iptable，ip6table命令是否正确安装。
        * openwrt 15.01系统不支持IPV6重定向，如网络需要支持IPV6，请将DNSMASQ上游改为smartdns，或者将smartdns的端口改为53，并停用dnsmasq。
        * LEDE之后系统，请安装IPV6的nat转发驱动。点击`system`->`Software`，点击`update lists`更新软件列表后，安装`ip6tables-mod-nat`
        * 使用如下命令检查路由规则是否生效。  

        ```shell
        iptables -t nat -L PREROUTING | grep REDIRECT
        ```

        * 如转发功能不正常，请使用方法二：作为DNSMASQ的上游。

1. 方法二：作为DNSMASQ的上游

    * **将dnsmasq的请求发送到smartdns**

        登录路由器，点击`Services`->`SmartDNS`->`redirect`，选择`作为dnsmasq的上游服务器`设置dnsmasq的上游服务器为smartdns。

    * **检测上游服务是否配置成功**

        * 方法一：使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
        看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

        ```shell
        pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
        Server:         192.168.1.1
        Address:        192.168.1.1#53

        Non-authoritative answer:
        0.0.0.0.in-addr.arpa  name = smartdns.
        ```

        * 方法二：使用`nslookup`查询`www.baidu.com`域名，查看结果中百度的IP地址是否`只有一个`，如有多个IP地址返回，则表示未生效，请多尝试几个域名检查。

        ```shell
        pi@raspberrypi:~ $ nslookup www.baidu.com 192.168.1.1
        Server:         192.168.1.1
        Address:        192.168.1.1#53

        Non-authoritative answer:
        www.baidu.com   canonical name = www.a.shifen.com.
        Name:   www.a.shifen.com
        Address: 14.215.177.38
        ```

1. 启动服务

    勾选配置页面中的`Enable(启用)`来启动SmartDNS

1. 注意：

    * 如已经安装chinaDNS，建议将chinaDNS的上游配置为SmartDNS。
    * SmartDNS默认情况，将53端口的请求转发到SmartDNS的本地端口，由`Redirect`配置选项控制。

### 华硕路由器原生固件/梅林固件

--------------

说明：梅林固件派生自华硕固件，理论上可以直接使用华硕配套的安装包使用。但目前未经验证，如有问题，请提交issue。

1. 准备

    在使用此软件时，需要确认路由器是否支持U盘，并准备好U盘一个。

1. 启用SSH登录

    登录管理界面，点击`系统管理`->点击`系统设置`，配置`Enable SSH`为`Lan Only`。  
    SSH登录用户名密码与管理界面相同。

1. 下载`Download Master`

    在管理界面点击`USB相关应用`->点击`Download Master`下载。  
    下载完成后，启用`Download Master`，如果不需要下载功能，此处可以卸载`Download Master`，但要保证卸载前Download Master是启用的。  

1. 安装SmartDNS

    将软件使用winscp上传到路由器的`/tmp/mnt/sda1`目录。（或网上邻居复制到sda1共享目录）

    ```shell
    ipkg install smartdns.xxxxxxx.mipsbig.ipk
    ```

1. 重启路由器生效服务

    待路由器启动后，使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
    看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

    ```shell
    pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
    Server:         192.168.1.1
    Address:        192.168.1.1#53

    Non-authoritative answer:
    0.0.0.0.in-addr.arpa  name = smartdns.
    ```

1. 额外说明

    上述过程，smartdns将安装到U盘根目录，采用optware的模式运行。
    其目录结构如下： （此处仅列出smartdns相关文件）  

    ```shell
    U盘
    └── asusware.mipsbig
            ├── bin
            ├── etc
            |    ├── smartdns
            |    |     └── smartdns.conf
            |    └── init.d
            |          └── S50smartdns
            ├── lib
            ├── sbin
            ├── usr
            |    └── sbin
            |          └── smartdns
            ....
    ```

    如要修改配置，可以ssh登录路由器，使用vi命令修改  

    ```shell
    vi /opt/etc/smartdns/smartdns.conf
    ```

    也可以通过网上邻居修改，网上邻居共享目录`sda1`看不到`asusware.mipsbig`目录，但可以直接在`文件管理器`中输入`asusware.mipsbig\etc\init.d`访问

    ```shell
    \\192.168.1.1\sda1\asusware.mipsbig\etc\init.d
    ```

### optware/entware

--------------

1. 准备

    在使用此软件时，需要确认路由器是否支持U盘，并准备好U盘一个。

1. 安装SmartDNS

    将软件使用winscp上传到路由器的`/tmp`目录。

    ```shell
    ipkg install smartdns.xxxxxxx.mipsbig.ipk
    ```

1. 修改smartdns配置

    ```shell
    vi /opt/etc/smartdns/smartdns.conf
    ```

    另外，如需支持IPV6，可设置工作模式为`2`，将dnsmasq的DNS服务禁用，smartdns为主用DNS服务器。将文件`/opt/etc/smartdns/smartdns-opt.conf`，中的`SMARTDNS_WORKMODE`修改为2.

    ```shell
    SMARTDNS_WORKMODE="2"
    ```

1. 重启路由器生效服务

    待路由器启动后，使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
    看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

    ```shell
    pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
    Server:         192.168.1.1
    Address:        192.168.1.1#53

    Non-authoritative answer:
    0.0.0.0.in-addr.arpa  name = smartdns.
    ```

    注意：若服务没有自动启动，则需要设置optwre/entware自动启动，具体方法参考optware/entware的文档。

### Windows 10 WSL安装/WSL ubuntu

--------------

1. 安装Windows 10 WSL ubuntu系统

    安装Windows 10 WSL运行环境，发行版本选择ubuntu系统。安装步骤请参考[WSL安装说明](https://docs.microsoft.com/en-us/windows/wsl/install-win10)

1. 安装smartdns

    下载安装包`smartdns.xxxxxxxx.x86_64.tar.gz`，并解压到D盘根目录。解压后目录如下：

    ```shell
    D:\SMARTDNS
    ├─etc
    │  ├─default
    │  ├─init.d
    │  └─smartdns
    ├─package
    │  └─windows
    ├─src
    └─systemd

    ```

    双击`D:\smartdns\package\windows`目录下的`install.bat`进行安装。要求输入密码时，请输入`WLS ubuntu`的密码。

1. 修改配置

    记事本打开`D:\smartdns\etc\smartdns`目录中的`smartdns.conf`配置文件配置smartdns。具体配置参数参考`配置参数`说明。  
    一般情况下，只需要增加`server [IP]:port`, `server-tcp [IP]:port`配置项，
    尽可能配置多个上游DNS服务器，包括国内外的服务器。配置参数请查看`配置参数`章节。

1. 重新加载配置

    双击`D:\smartdns\package\windows`目录下的`reload.bat`进行安装。要求输入密码时，请输入`WLS ubuntu`的密码。

1. 将DNS请求转发的SmartDNS解析。

    将Windows的默认DNS服务器修改为`127.0.0.1`，具体步骤参考[IP配置](https://support.microsoft.com/zh-cn/help/15089/windows-change-tcp-ip-settings)

1. 检测服务是否配置成功。

    使用`nslookup -querytype=ptr 0.0.0.0`查询域名  
    看命令结果中的`name`项目是否显示为`smartdns`或`主机名`，如`smartdns`则表示生效  

    ```shell
    pi@raspberrypi:~/code/smartdns_build $ nslookup -querytype=ptr 0.0.0.0
    Server:         192.168.1.1
    Address:        192.168.1.1#53

    Non-authoritative answer:
    0.0.0.0.in-addr.arpa  name = smartdns.
    ```

## 配置参数

|参数|  功能  |默认值|配置值|例子|
|--|--|--|--|--|
|server-name|DNS服务器名称|操作系统主机名/smartdns|符合主机名规格的字符串|server-name smartdns
|bind|DNS监听端口号|[::]:53|IP:PORT|bind 192.168.1.1:53
|bind-tcp|TCP模式DNS监听端口号|[::]:53|IP:PORT|bind-tcp 192.168.1.1:53
|cache-size|域名结果缓存个数|512|数字|cache-size 512
|tcp-idle-time|TCP链接空闲超时时间|120|数字|tcp-idle-time 120
|rr-ttl|域名结果TTL|远程查询结果|大于0的数字|rr-ttl 600
|rr-ttl-min|允许的最小TTL值|远程查询结果|大于0的数字|rr-ttl-min 60
|rr-ttl-max|允许的最大TTL值|远程查询结果|大于0的数字|rr-ttl-max 600
|log-level|设置日志级别|error|fatal,error,warn,notice,info,debug|log-level error
|log-file|日志文件路径|/var/log/smartdns.log|路径|log-file /var/log/smartdns.log
|log-size|日志大小|128K|数字+K,M,G|log-size 128K
|log-num|日志归档个数|2|数字|log-num 2
|audit-enable|设置审计启用|no|[yes\|no]|audit-enable yes
|audit-file|审计文件路径|/var/log/smartdns-audit.log|路径|audit-file /var/log/smartdns-audit.log
|audit-size|审计大小|128K|数字+K,M,G|audit-size 128K
|audit-num|审计归档个数|2|数字|audit-num 2
|conf-file|附加配置文件|无|文件路径|conf-file /etc/smartdns/smartdns.more.conf
|server|上游UDP DNS|无|可重复<br>`[ip][:port]`：服务器IP，端口可选。<br>`[-group [group] ...]`：DNS服务器所属组，比如office, foreign，和nameserver配套使用。<br>`[-exclude-default-group]`：将DNS服务器从默认组中排除| server 8.8.8.8:53 -group g1
|server-tcp|上游TCP DNS|无|可重复<br>`[ip][:port]`：服务器IP，端口可选。<br>`[-group [group] ...]`：DNS服务器所属组，比如office, foreign，和nameserver配套使用。<br>`[-exclude-default-group]`：将DNS服务器从默认组中排除| server-tcp 8.8.8.8:53
|server-tls|上游TLS DNS|无|可重复<br>`[ip][:port]`：服务器IP，端口可选。<br>`[-spki-pin [sha256-pin]]`: TLS合法性校验SPKI值，base64编码的sha256 SPKI pin值<br>`[-host-name]`：TLS SNI名称。<br>`[-group [group] ...]`：DNS服务器所属组，比如office, foreign，和nameserver配套使用。<br>`[-exclude-default-group]`：将DNS服务器从默认组中排除| server-tls 8.8.8.8:853
|server-https|上游HTTPS DNS|无|可重复<br>`https://[host][:port]/path`：服务器IP，端口可选。<br>`[-spki-pin [sha256-pin]]`: TLS合法性校验SPKI值，base64编码的sha256 SPKI pin值<br>`[-host-name]`：TLS SNI名称<br>`[-http-host]`：http协议头主机名。<br>`[-group [group] ...]`：DNS服务器所属组，比如office, foreign，和nameserver配套使用。<br>`[-exclude-default-group]`：将DNS服务器从默认组中排除| server-https https://cloudflare-dns.com/dns-query
|address|指定域名IP地址|无|address /domain/[ip\|-\|-4\|-6\|#\|#4\|#6] <br>`-`表示忽略 <br>`#`表示返回SOA <br>`4`表示IPV4 <br>`6`表示IPV6| address /www.example.com/1.2.3.4
|nameserver|指定域名使用server组解析|无|nameserver /domain/[group\|-], `group`为组名，`-`表示忽略此规则，配套server中的`-group`参数使用| nameserver /www.example.com/office
|ipset|域名IPSET|None|ipset /domain/[ipset\|-], `-`表示忽略|ipset /www.example.com/pass
|ipset-timeout|设置IPSET超时功能启用|auto|[yes]|ipset-timeout yes
|bogus-nxdomain|假冒IP地址过滤|无|[ip/subnet]，可重复| bogus-nxdomain 1.2.3.4/16
|force-AAAA-SOA|强制AAAA地址返回SOA|no|[yes\|no]|force-AAAA-SOA yes
|prefetch-domain|域名预先获取功能|no|[yes\|no]|prefetch-domain yes
|dualstack-ip-selection|双栈IP优选|no|[yes\|no]|dualstack-ip-selection yes
|dualstack-ip-selection-threshold|双栈IP优选阈值|30ms|毫秒|dualstack-ip-selection-threshold [0-1000]

## FAQ

1. SmartDNS和DNSMASQ有什么区别  
    SMARTDNS在设计上并不是替换DNSMASQ的，SMARTDNS主要功能集中在DNS解析增强上，增强部分有：
    * 多上游服务器并发请求，对结果进行测速后，返回最佳结果；
    * address，ipset域名匹配采用高效算法，查询匹配更加快速高效，路由器设备依然高效。
    * 域名匹配支持忽略特定域名，可单独匹配IPv4， IPV6，支持多样化定制。
    * 针对广告屏蔽功能做增强，返回SOA，屏蔽广告效果更佳；
    * IPV4，IPV6双栈IP优选机制，在双网情况下，选择最快的网络通讯。
    * 支持最新的TLS, HTTPS协议，提供安全的DNS查询能力。
    * ECS支持，是查询结果更佳准确。
    * 域名预查询，访问常用网站更加快速。
    * 域名TTL可指定，使访问更快速。
    * 高速缓存机制，使访问更快速。
    * 异步日志，审计机制，在记录信息的同时不影响DNS查询性能。
    * 域名组（group）机制，特定域名使用特定上游服务器组查询，避免隐私泄漏。

1. 如何配置上游服务器最佳。  
    smartdns有测速机制，在配置上游服务器时，建议配置多个上游DNS服务器，包含多个不同区域的服务器，但总数建议在10个左右。推荐配置  
    * 运营商DNS。
    * 国内公共DNS，如`119.29.29.29`, `223.5.5.5`。
    * 国外公共DNS，如`8.8.8.8`, `8.8.4.4`。

1. 如何启用审计日志  
    审计日志记录客户端请求的域名，记录信息包括，请求时间，请求IP，请求域名，请求类型，如果要启用审计日志，在配置界面配置`audit-enable yes`启用，`audit-size`, `audit-file`, `audit-num`分别配置审计日志文件大小，审计日志文件路径，和审计日志文件个数。审计日志文件将会压缩存储以节省空间。

1. 如何避免隐私泄漏  
    smartdns默认情况下，会将请求发送到所有配置的DNS服务器，若上游DNS服务器使用DNS，或记录日志，将会导致隐私泄漏。为避免隐私泄漏，请尽量：  
    * 配置使用可信的DNS服务器。
    * 优先使用TLS查询。
    * 设置上游DNS服务器组。

1. 如何屏蔽广告  
    smartdns具备高性能域名匹配算法，通过域名方式过滤广告非常高效，如要屏蔽广告，只需要配置类似如下记录即可，如，屏蔽`*.ad.com`，则配置：

    ```sh
    address /ad.com/#
    ```

    域名的使后缀模式，过滤*.ad.com，`#`表示返回SOA，使屏蔽广告更加高效，如果要单独屏蔽IPV4， 或IPV6， 在`#`后面增加数字，如`#4`表示对IPV4生效。若想忽略特定子域名的屏蔽，可配置如下，如忽略`pass.ad.com`，可配置如下：

    ```sh
    address /pass.ad.com/-
    ```

1. 如何使用DNS查询分流  
    某些情况下，需要将有些域名使用特定的DNS服务器来查询来做到DNS分流。比如。

    ```sh
    .home -> 192.168.1.1
    .office -> 10.0.0.1
    ```

    .home 结尾的域名发送到192.168.1.1解析  
    .office 结尾的域名发送到10.0.0.1解析
    其他域名采用默认的模式解析。
    这种情况的分流配置如下：

    ```sh
    #配置上游，用-group指定组名，用-exclude-default-group将服务器从默认组中排除。
    server 192.168.1.1 -group home -exclude-default-group
    server 10.0.0.1 -group office -exclude-default-group
    server 8.8.8.8

    #配置解析的域名
    nameserver /.home/home
    nameserver /.office/office
    ```

    通过上述配置即可实现DNS解析分流

1. IPV4, IPV6双栈IP优选功能如何使用  
    目前IPV6已经开始普及，但IPV6网络在速度上，某些情况下还不如IPV4，为在双栈网络下获得较好的体验，smartdns提供来双栈IP优选机制，同一个域名，若IPV4的速度远快与IPV6，那么smartdns就会阻止IPV6的解析，让PC使用IPV4访问，具体配置文件通过`dualstack-ip-selection yes`启用此功能，通过`dualstack-ip-selection-threshold [time]`来修改阈值。如果要完全禁止IPV6 AAAA记录解析，可设置`force-AAAA-SOA yes`。

1. 如何提高cache效率，加快访问速度  
    smartdns提供了域名缓存机制，对查询的域名，进行缓存，缓存时间符合DNS TTL规范。为提高缓存命中率，可采用如下措施：  
    * 适当增大cache的记录数  
    通过`cache-size`来设置缓存记录数。  
    查询压力大的环境下，并且有内存大的机器的情况下，可适当调大。  

    * 适当设置最小TTL值  
    通过`rr-ttl-min`将最低DNS TTL时间设置为一个合理值，延长缓存时间。  
    建议是超时时间设置在10～30分钟，避免服务器域名变化时，查询到失效域名。

    * 开启域名预获取功能  
    通过`prefetch-domain yes`来启用域名预先获取功能，提高查询命中率。  
    配合上述ttl超时时间，smartdns将在域名ttl即将超时使，再次发送查询请求，并缓存查询结果供后续使用。频繁访问的域名将会持续缓存。此功能将在空闲时消耗更多的CPU。

## 声明

* `SmartDNS`著作权归属Nick Peng (pymumu at gmail.com)。
* `SmartDNS`为免费软件，用户可以非商业性地复制和使用`SmartDNS`。
* 禁止将 `SmartDNS` 用于商业用途。
* 使用本软件的风险由用户自行承担，在适用法律允许的最大范围内，对因使用本产品所产生的损害及风险，包括但不限于直接或间接的个人损害、商业赢利的丧失、贸易中断、商业信息的丢失或任何其它经济损失，不承担任何责任。
* 本软件不会未经用户同意收集任何用户信息。

## 说明

目前代码未开源，后续根据情况开源。
