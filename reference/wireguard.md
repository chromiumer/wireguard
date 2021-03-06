### Server

1.Install packages(Centos7)

```
curl -Lo /etc/yum.repos.d/wireguard.repo https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
yum -y install epel-release
yum -y install wireguard-dkms wireguard-tools
yum -y update
```

check kernel mod.
```
[root@localhost ~]# modprobe wireguard && lsmod | grep wireguard
wireguard             217088  0
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             16384  1 wireguard
```

2.config file.
```
cat <<EOF>> /etc/wireguard/wg0.conf
[Interface]
Address = 1.1.1.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE;
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE;
ListenPort = 12222
PrivateKey = server-private-key
EOF
```
>generate private&public key: wg genkey | tee wireguard-private.key | wg pubkey > wireguarrd-public.key

open route forward.
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

3.start&stop wg0
```
wg-quick up wg0
wg-quick down wg0
```

wg //check wg server
```
interface: wg0
  public key: server-public-key
  private key: (hidden)
  listening port: 12222
```

after client config complete. set server peer like this.

>wg set wg0 peer client-public-key  allowed-ips 1.1.1.2/32  //add client

>wg set wg0 peer client-public-key remove //delete client

---

### Client

1.Install packages(MacOS)
```
brew install wireguard-tools
```

2.config file.
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
Endpoint = server-ip:12222
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 30
EOF
```
>generate private&public key: wg genkey | tee wireguard-private.key | wg pubkey > wireguarrd-public.key

3.start&stop wg0
```
wg-quick up wg0
wg-quick down wg0
```

ps:
By default, all client routes will be proxied out, and the routes will need to be modified so that only the remote host will take the wg0 network card

```
route -n delete -inet 0.0.0.0/1
route -q -n add -inet 10.0.0/24 -interface utun5
```


Ref???https://www.wireguard.com/install/
