# 海外同学访问国内应用部署及优化指南
<i class="fab fa-github"></i> am1006
禁止转载，转发随意

`本文最后更新时间：2018.12.27`

[TOC]
##前言
作为一个在海外上学的同学，我们在脱离了GFW的时候发现一个问题：国内大量应用具有IP限制和封锁，这意味着在海外上学没有办法接触到国内的一些服务了。

对此，我们有两个解决方案：
1. 使用UA(User agent)来欺骗主机，使它认为我们的访问是被许可的。
2. 使用代理接入中国大陆网络，继而获得IP地址。也就是我戏称的“翻回来”。

综合考虑，第二种方案是比价可靠的，具有根本性的优点。

我们可以使用的代理有多种协议。大家可以自行选择。

而根据我的了解，海外同学比较常见的是向服务商购买VPN产品。我认为，这是有风险的事情：
- 你无法信任服务商，但是你的数据要明文经过他的服务器。而且你访问国内的产品里往往涉及你的重要信息，你的隐私得不到保障。
- 你访问国内的产品往往是一些很重量的应用，比如音乐或者视频网站。买一个足够流量的产品价格不菲。

那么我推荐的解决方案是什么呢？

**使用自己的服务器，并自建代理。**

接下来是我的搭建笔记和使用心得，这是一个硬核的操作，不建议盲目尝试。建议没有基础的同学可以找一个学习计算机的同学合租服务器使用，或者找一个很懂这个的男朋友/女朋友（逃

##ocserv部署及应用笔记
### 1. 准备
1. 一台国内主机。建议挑选Debian系的系统并且是KVM架构，本笔记的系统是Ubuntu16.04。
2. 一个域名。并且解析到主机。
3. 一个网络代理协议，出于多种考虑，我最终还是选择了VPN，并且用了ocserv这个方案。这样子就可以用Cisco的商用方案，避免了很多不必要的麻烦。（Cisco的anyconnect也是ANU的VPN方案商，客户端可以在ANU官网下载）

### 2. 主机端
####主机基础配置
购买到的主机有一些基础配置要做。
一般有：
- 检查PPP/TUN环境
```
# 首先要检查VPS的TUN是否开启(OpenVZ虚拟化的服务器很可能默认关闭)。
cat /dev/net/tun
# 返回的必须是：
cat: /dev/net/tun: File descriptor in bad state
# 如果返回内容不是指定的结果，请与VPS提供商联系开启TUN权限（一般控制面板有开关）。
```

- 更新系统:
 ```
 apt-get update && apt-get upgrade -y
 ```
 
- 安装或更新git，sudo，vim。
- 我们更新软件包列表，并开始安装依赖：
```
apt-get install pkg-config build-essential libgnutls28-dev libwrap0-dev liblz4-dev libseccomp-dev libreadline-dev libnl-nf-3-dev libev-dev gnutls-bin -y`
```
重启主机后开始证书的签发。

####证书
ocserv 需要 SSL 证书，网上许多教程中使用的是自签发证书，方法复杂且容易被 MITM 攻击，所以我决定使用[Let’s Encrypt](https://letsencrypt.org/) 免费为自己域名添加证书。

使用 certbot 来获取一个Let’s Encrypt证书.

下载certbot：
```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```
其他系统请参考 [Certbot 官方网站](https://certbot.eff.org/).

安装好certbot后使用：
`certbot certonly`
会看到：
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
1: Spin up a temporary webserver (standalone)
2: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

这里选择 1(我只要一个证书)，让certbot来搞定证书获取的过程，之后输入自己的域名，比如我给的例子vpn.example.com，稍等片刻应该可以看到类似如下的输出（**建议保存下来，务必记住证书存放的地址，后面会用到**）：
```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/vpn.example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/vpn.example.com/privkey.pem
   Your cert will expire on 2019-03-26. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```

####安装ocserv
以前手动编译源代码包的方式安装，但是现在Debian系的系统已经有了编译好的软件包^非常感谢维护这个包的：Aron_Xu，Liang_Guo和Mike_Miller ,详情见[Distribution Status](https://ocserv.gitlab.io/www/packages.html)，我们的Ubuntu，一条指令即可：
`sudo apt install ocserv -y`

打开系统的转发功能，在 /etc/sysctl.conf 中加入如下行：
```
# 如果你不需要IPv4转发，注释掉下面这一行
net/ipv4/ip_forward=1

# 如果你不需要IPv6转发，注释掉下面这两行
net/ipv6/conf/default/forwarding=1
net/ipv6/conf/all/forwarding=1
```
通过`sysctl -p`保存.

再来点增加地址伪装：
`iptables -t nat -A POSTROUTING -j MASQUERADE`

####配置ocserv
默认安装好后在`/etc/ocserv/`下有一个配置文件`ocserv.conf`，注意以下配置字段：

```
#  这个是登陆方式，plain[passwd=/etc/ocserv/ocpasswd] 代表使用密码登陆并且从 /etc/ocserv/ocpasswd 文件中读取用户名和密码 
auth = "plain[/etc/ocserv/ocpasswd]"

# 允许同时连接的客户端数量
# 整个VPN最大链接客户端数量，比如下面的4就是最多只能4台设备同时使用,改成0就是不作限制
max-clients = 4

# 限制同一客户端的并行登陆数量
# 限制的是同账号链接VPN最大客户端数量。改成0就是不作限制
max-same-clients = 2

# 服务监听的TCP/UDP端口（默认为443,请把它给改了，因为我不确定服务器会不会用来跑web服务)
tcp-port = 443
udp-port = 443

# 自动优化 MTU，尝试改善网络性能
try-mtu-discovery = true

# 服务器证书与密钥，就是上一步中生成的证书和私钥的位置
server-cert = /etc/letsencrypt/live/vpn.example.com/fullchain.pem
server-key = /etc/letsencrypt/live/vpn.example.com/privkey.pem

# 服务器域名
default-domain = vpn.example.com

# 客户端连上 vpn 后使用的 DNS，为了安全，这里使用 Cloudflare 的 1.1.1.1
# 你也可以使用其他的，比如Google的8.8.8.8
dns = 1.1.1.1

# 注释掉所有的 route 和 no-route，让服务器成为gateway，这样服务器就会转发所有的包；如果配置好了国内外分流路由表，可以打开，这样就可以实现自动让国内应用走这个VPN
# 否则服务器只会转发目的网段在route之中的包。
#route = 192.168.1.0/255.255.255.0
#no-route = 192.168.5.0/255.255.255.0

# 启用 Cisco 客户端兼容性支持
cisco-client-compat = true

# Anyconnect有一个设置连接欢迎信息的功能
# 在连接的时候会弹出一个提示框
# 提示框的内容就可以自行设置
banner = "Welcome LALA.IM"

# ocserv监听的IP地址，千万别动动了就爆炸
# 一定要把这个的#注释保留！！！
#listen-host = [IP|HOSTNAME]

# 让服务器读取用户证书
# 二选一，UID不行就用CN，我这里默认的UID是可以的
# The object identifier that will be used to read the user ID in the client 
# certificate. The object identifier should be part of the certificate's DN
# Useful OIDs are: 
#  CN = 2.5.4.3, UID = 0.9.2342.19200300.100.1.1
cert-user-oid = 0.9.2342.19200300.100.1.1

```

> route可以配置多条，从而实现智能分流（不在route列表里的不走VPN），[配置举例](https://github.com/don-johnny/anyconnect-routes/blob/master/routes)。由于OpenConnect的原理所限，不可能像socks那样直接配置域名甚至使用PAC。
>不管因为什么原因，如果你不幸没有/etc/ocserv/ocserv.conf这个文件，可以去ocserv的repo上去下载sample.conf，然后在此基础上进行改动。




由于使用用户名密码登录，我们需要生成一个密码文件，指令如下：
`$ ocpasswd -c /etc/ocserv/ocpasswd <用户名>`
此时会要求你输入两边密码，如果需要再添加用户只需重复上述指令即可.
配置好后启动 VPN：
`ocserv -c /etc/ocserv/ocserv.conf`
确认已经开启：
```
#443是端口号
$ netstat -tulpn | grep 443
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      1987/ocserv         
tcp6       0      0 :::443                  :::*                    LISTEN      1987/ocserv         
udp        0      0 0.0.0.0:443             0.0.0.0:*                           1987/ocserv         
udp6       0      0 :::443                  :::*                                1987/ocserv
```

设置Ocserv开机启动：
`systemctl enable ocserv`


####Q&A or Debug
1. 如果直接用浏览器去访问 VPN 网关（比如我的例子：https://vpn.example.com）的话，返回的是如下 HTML 内容：

    ```
<?xml version="1.0" encoding="UTF-8"?>
<config-auth client="vpn" type="auth-request">
<version who="sg">0.1(1)</version>
<auth id="main">
<message>Please enter your username.</message>
<form method="post" action="/auth">
<input type="text" name="username" label="Username:" />
</form></auth>
</config-auth>
    ```

    并且你会看到HTTPS加密是有效的，证书签发就是Let’s Encrypt。

2. 如果开启了 ocserv 之后在  sudo 的时候卡住并提示：`sudo: unable to resolve host vpn: Resource temporarily unavailable`着重关注一下自己的/etc/hosts文件中是否包含一行：
    ```
    127.0.0.1	localhost
    ```
    
3. 为什么不使用一键脚本？
    因为他们都太久没更新了。而且自己一步步来也可以在未来debug的时候有点数。
    
4. Firewall怎么开？
    ```
    # 默认TCP/UDP端口都是 443，如果你改了,那么改为其他端口即可
    iptables -I INPUT -p tcp --dport 443 -j ACCEPT
    iptables -I INPUT -p udp --dport 443 -j ACCEPT
    # 如果你以后要删除规则，那么把 -I 改成 -D 即可。
    iptables -D INPUT -p tcp --dport 443 -j ACCEPT
    iptables -D INPUT -p udp --dport 443 -j ACCEPT
    ```
    
    国内主机一般可以在控制台开，因为在bash里开了，一般默认 iptbales 关机后并不会保存规则，这样开机后防火墙规则也全都清空了，所以需要设置一下。
    ```
# Debian/Ubuntu 系统：
iptables-save > /etc/iptables.up.rules
echo -e '#!/bin/bash\n/sbin/iptables-restore < /etc/iptables.up.rules' > /etc/network/if-pre-up.d/iptables
chmod +x /etc/network/if-pre-up.d/iptables
# 以后需要保存防火墙规则只需要执行：
iptables-save > /etc/iptables.up.rules
    ```

5. 常见命令
```
/etc/init.d/ocserv start
# 启动 ocserv 
/etc/init.d/ocserv stop
# 停止 ocserv 
/etc/init.d/ocserv restart
# 重启 ocserv 
/etc/init.d/ocserv status
# 查看 ocserv 运行状态 
/etc/init.d/ocserv log
# 查看 ocserv 运行日志
/etc/init.d/ocserv test
# 测试 ocserv 配置文件是否正确
# 配置文件：
/etc/ocserv/ocserv.conf
```

6. 优化建议
    建议在运行ocserv前，执行一下这个命令`ulimit -n 51200`，作用是提高系统的文件符同时打开数量，对于TCP连接过多的时候系统默认的`1024`就会成为速度瓶颈。
    这个命令只有临时有效，重启后失效，如果想要永久有效，请执行：
    ```
    echo "* soft nofile 51200
    * hard nofile 51200" >> /etc/security/limits.conf
    ```
    然后最后再执行一下`ulimit -n 51200`即可。

7. 客户增减命令
    ```
添加用户:
ocpasswd -c /etc/ocserv/ocpasswd <用户名>
添加分组用户:
ocpasswd -c /etc/ocserv/ocpasswd -g <用户组名> <用户名>
执行下面的命令删除指定的用户:
ocpasswd -c /etc/ocserv/ocpasswd -d <用户名>
锁定 VPN用户:
ocpasswd -c /etc/ocserv/ocpasswd -l <用户名>
解锁 VPN用户:
ocpasswd -c /etc/ocserv/ocpasswd -u <用户名>
注意：删除/锁定/解锁 VPN用户都没有任何提示！
```
可以用`man ocpasswd`查看

8. 能不能装优化软件？
    锐速不可以，bbr可以。


### 3. 客户端 
下载：直接去ANU官网啦~商业级别的客户端哦
[ANU VPN](https://services.anu.edu.au/information-technology/login-access/virtual-private-network-0)
自己按需装好即可。

注意：
OS X系统安装**只安装VPN组件**,其他功能**都不需要安装**(红框内的都要取消)
![配置说明](https://www.catpaw2012.net/docs/wp-content/uploads/2015/11/20160629171230.jpg)

打开客户端之后，将自己的账号密码填入。这些都很简单。
   

##Reference
1. [ANU VPN](https://services.anu.edu.au/information-technology/login-access/virtual-private-network-0)
2. [ocserv gitlab](https://gitlab.com/openconnect/ocserv)
3. [ocserv](https://ocserv.gitlab.io/www/)
4. [Certbot](https://certbot.eff.org/docs/intro.html)
5. [十大VPN类型在常见操作系统中的支持情况](https://www.igfw.net/archives/13535)
6. [基于OpenConnect 构建的SSL VPN解决方案 - 知乎](https://zhuanlan.zhihu.com/p/30982033)
7. [新鲜出炉一份完整AnyConnect教程！ – 1024++](https://tech.msla.top/2017/09/13/%E6%96%B0%E9%B2%9C%E5%87%BA%E7%82%89%E4%B8%80%E4%BB%BD%E5%AE%8C%E6%95%B4anyconnect%E6%95%99%E7%A8%8B%EF%BC%81/)
8. [CentOS7使用Ocserv搭建CiscoAnyconnect服务器-荒岛](https://lala.im/3684.html)
9. [Openconnect VPN 服务端 ocserv 的部署 - 飞羽博客](https://cokebar.info/archives/1363)
10. [Cisco AnyConnect VPN Windows/Android 平台客户端使用教程 | 逗比根据地](https://doubmirror.cf/hc5a0yn5.html)
11. [使用Ocserv 手动搭建 Cisco AnyConnect VPN服务端 | 逗比根据地](https://doubmirror.cf/k9994-k3.html)
12. [使用openconnect代替cisco anyconnect - DTeam的团队日志 - SegmentFault 思否](https://segmentfault.com/a/1190000011530974)
13. [在Ubuntu上配置OpenConnect - GitHub.io by cosmozhang1995](http://www.cosmozhang.com/2018/04/18/deployopenconnectserveronubuntu.html)
14. [使用 ocserv 搭建企业级 OpenConnect VPN 网关并使用 Let's Encrypt 证书 | Nova Kwok's Awesome Blog](https://nova.moe/deploy-openconnect-ocserv-with-letsencrypt/)
15. [Linux簡易配置Let's Encrypt證書及其在Anyconnect中的使用方法 | Touko's diary](https://touko.moe/blog/letsencrypt-ocserv-or-more)
16. [从零开始：手把手教你搭建私人 Cisco AnyConnect VPN 服务 - 老盐鸡汤面](https://blog.ysoup.org/tech/cisco-anyconnect-vpn.html)
17. [Cisco Anyconnect: Roaming Security profile is missing – FINKOTEK](https://finkotek.com/cisco-anyconnect-roaming-security-profile-is-missing/)
18. [OpenConnect VPN server · iMeiji/shadowsocks_install Wiki](https://github.com/iMeiji/shadowsocks_install/wiki/OpenConnect-VPN-server)
19. [Cisco AnyConnect软件下载及更新 – 猫的梯子](https://www.catpaw2012.net/docs/?p=420)
20. [AnyConnect第一次连接证书安装标准作业说明 – 猫的梯子](https://www.catpaw2012.net/docs/?p=3393)



<head> 
<script defer src="https://use.fontawesome.com/releases/v5.6.1/js/all.js"></script> 
<script defer src="https://use.fontawesome.com/releases/v5.6.1/js/v4-shims.js"></script> 
<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.6.1/css/all.css" integrity="sha384-gfdkjb5BdAXd+lj+gudLWI+BXq4IuLW5IT+brZEZsLFm++aCMlF1V92rMkPaX4PP" crossorigin="anonymous">
</head> 
