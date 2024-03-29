# wireguard

### 说明：wireguard使用udp协议通信，服务端和客户端只通过交换公钥实现数据加解密。

### 服务端

安装(CentOS7/8)

##### CentOS 7

```
yum install epel-release elrepo-release
yum install yum-plugin-elrepo
yum install kmod-wireguard wireguard-tools
reboot
```
##### CentOS 8
```
yum install elrepo-release epel-release
yum install kmod-wireguard wireguard-tools
```

##### 加载内核模块
```
[root@localhost ~]# modprobe wireguard && lsmod | grep wireguard
wireguard             217088  0
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             16384  1 wireguard
```

配置文件
```
生成公私钥密钥: wg genkey | tee wireguard-private.key | wg pubkey > wireguard-public.key
私钥文件: wireguard-private.key
公钥文件: wireguard-public.key
```
```
cat <<EOF>> /etc/wireguard/wg0.conf
[Interface]
Address = 1.1.1.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE;
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE;
ListenPort = 666
PrivateKey =  wireguard-private.key //私钥文件内容
EOF
```

##### 开启路由转发
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

iptables -t nat -A POSTROUTING -j MASQUERADE
```

##### 服务起停
```
wg-quick up wg0
wg-quick down wg0
```

##### wg 检查服务
```
interface: wg0
  public key: server-public-key
  private key: (hidden)
  listening port: 666
```
##### 管理client

```
添加client
wg set wg0 peer client-public-key  allowed-ips 1.1.1.2/32
删除client
wg set wg0 peer client-public-key remove
```
---

客户端

#### 安装wireguard client (MacOS)
```
brew install wireguard-tools
```

#### 配置文件

```
cat <<EOF>> /etc/wireguard/wg0.conf
[Interface]
PrivateKey = client-private-key
Address = 1.1.1.2/32
DNS = 8.8.8.8
DNS = 8.8.4.4
DNS = 114.114.114.114

[Peer]
PublicKey = server-public-key
Endpoint = server-ip:666
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 30
EOF
```

##### 服务起停
```
wg-quick up wg0
wg-quick down wg0
```

#### 默认所有流量都会通过wg0转发出去，通过以下方式管理本地路由。
```
route -n delete -inet 0.0.0.0/1
route -q -n add -inet 10.0.0/24 -interface utun5
```

---

中继路由器方案（wifi分享给办公网小伙伴）
办公网1台PC或工控机（安装CentOS）。

#### 开启路由转发
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -A POSTROUTING -j MASQUERADE
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables-save
```
#### crontab 上班时间开启，下班时间关闭（减少连接时间，避免ip或port被ban。）
```
0 9 * * * sudo wg-quick up wg0
0 17 * * * sudo wg-quick down wg0
```

```
                                             \ | /  <----- Hi I'm wifi !!
Wireguard Client                              \|/ 
工控机<------------------------>(LAN)Tp-Link-Router(wireless)(WAN)<---------->(Lan)office site ...... <----------->Wireguard Server
static 192.168.1.103                    192.168.1.253
  ^                                DHCP DefaultGW 192.168.1.103
  |_____________________________________________|

1.工控机配置静态(必须是静态)ip
2.路由器Web配置DHCP服务网关指向192.168.1.103
3.工控机运行Wireguard Client全局流量走向Wireguard Server
4.用户(手机/PC)连接Wifi全局出国学习。

```

Doc Ref：https://www.wireguard.com/install

Ennnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnd
