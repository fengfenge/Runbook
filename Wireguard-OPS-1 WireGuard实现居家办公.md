# Wireguard-OPS-1 WireGuard实现居家办公

**在公司内网（或一个公网服务器）上部署一台 WireGuard 服务器，家里的电脑作为客户端与其建立加密连接**。连接成功后，电脑就像在公司内网一样，可以安全地访问内部资源。

## 阿里云轻量服务器配置

```bash
#生成密钥对：在服务器上执行以下命令生成私钥和公钥
cd /etc/wireguard
umask 077
wg genkey | tee server_privatekey | wg pubkey > server_publickey

# 编辑服务端配置文件：创建 /etc/wireguard/wg0.conf 文件
vim /etc/wireguard/wg1.conf
[Interface]
Address = 10.20.1.1/24
ListenPort = 51821
PrivateKey = sHhJ+32LvSwmnYXd0VmLKk+Chys1Sh60Xv1gvUCwgEk=

# 开启转发 + NAT（eth0 替换为阿里云实际公网网卡名）
PostUp = iptables -A FORWARD -i wg1 -j ACCEPT; iptables -A FORWARD -o wg1 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg1 -j ACCEPT; iptables -D FORWARD -o wg1 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# ─────────────────────────────────────────────
# Peer 1：堡垒机（IDC 网关，无公网IP）
# ─────────────────────────────────────────────
[Peer]
PublicKey = /t2SHeXdfXpxFqM7xqOepg8buHuosUy98aZfiMyBGhM=
# 关键：192.168.13.0/24 在堡垒机后面，流量转发给它
AllowedIPs = 10.20.1.2/32, 192.168.13.0/24
PersistentKeepalive = 25

# ─────────────────────────────────────────────
# Peer 2：运维人员张三
# ─────────────────────────────────────────────
#[Peer]
#PublicKey = cjyeNkMAKEoQdnPxi4wysreErCb81ngbDEbX1hgqdmQ=
#AllowedIPs = 10.20.1.12/32
#PersistentKeepalive = 25

# ─────────────────────────────────────────────
# Peer 3：开发人员周辉
# ─────────────────────────────────────────────
[Peer]
PublicKey = aptOuH5uQ7hBqXLhDlxvSx6w7dG2BBzbgzSvxBRZEFk=
AllowedIPs = 10.20.1.11/32
PersistentKeepalive = 25

systemctl enable wg-quick@wg1
```

## 内网无公网的wireguard配置

```bash
[Interface]
PrivateKey = 8HEIENufpChx9rW0IWQTsgBJPNLBJkvwUy/LVqc1an4=
Address = 10.20.1.2/24

# 开启 IP 转发 + SNAT
# eth0 替换为gateway连接 192.168.13.0/24 的实际网卡名
PostUp = sysctl -w net.ipv4.ip_forward=1; iptables -t nat -A POSTROUTING -s 10.20.1.0/24 -o eth0 -j MASQUERADE; iptables -A FORWARD -i wg1 -o eth0 -j ACCEPT; iptables -A FORWARD -i eth0 -o wg1 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.20.1.0/24 -o eth0 -j MASQUERADE; iptables -D FORWARD -i wg1 -o eth0 -j ACCEPT; iptables -D FORWARD -i eth0  -o wg1 -m state --state RELATED,ESTABLISHED -j ACCEPT

[Peer]
PublicKey = DWYrjuYZyQsQsopX6eNpDP6ukt6BBm4B6y2EYuxyt2Y=
Endpoint = 47.112.120.205:51821
AllowedIPs = 10.20.1.0/24
PersistentKeepalive = 25
```



## windows客户端配置

```
wg genkey | tee wg1-ops-private.key | wg pubkey > wg1-ops-public.key

[Interface]
PrivateKey = UMLiF0IanT4QRY4PrUE9C4GFHV40KzbvVvfmBh6b1WY=
Address = 10.20.1.11/24
DNS = 192.168.13.49

[Peer]
PublicKey = DWYrjuYZyQsQsopX6eNpDP6ukt6BBm4B6y2EYuxyt2Y=
AllowedIPs = 10.20.1.0/24, 192.168.13.0/24
Endpoint = 47.112.120.205:51821
PersistentKeepalive = 25
```

