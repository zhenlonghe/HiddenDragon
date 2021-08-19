# Gateway

1.  安装 Debian 系统，配置网络信息
```bash
#以下均以 root 运行
#更改为静态ip
ssh-copy-id -i ~/.ssh/id_rsa.pub foo@ip

vi /etc/network/interfaces
allow-hotplug enp1s0
iface enp1s0 inet static
  address 192.168.50.10
  netmask 255.255.255.0
  gateway 192.168.50.1
  dns-nameservers 192.168.50.1 223.5.5.5 223.6.6.6

#重启一下系统
shutdown -r now 

#开启 BBR 加速
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
sysctl -p
#检查是否成功：
sysctl net.ipv4.tcp_available_congestion_control
#显示：
sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = bbr cubic reno
#表示开启成功。

#或执行 
lsmod | grep bbr #检测 BBR 是否开启。
```



2. 下载 clash, 配置文件并开机自启动服务化

# https://github.com/Dreamacro/clash/releases/tag/premium
```bash
cd /usr/local/bin/
# download the right version
wget -c https://github.com/Dreamacro/clash/releases/download/premium/clash-linux-armv8-2021.07.03.gz
tar -xzf clash-linux-armv8.tar.gz -O > clash 
chmod +x clash
clash -v


# config 文件
curl -o /root/.config/clash/config.yaml https://dler.cloud/subscribe/$YOUR_TOKEN\?clash\=love\&area\=hk+sg\&match=IEPL\|BGP\|AIA\&head\=https://gist.githubusercontent.com/zhenlonghe/282bcaa8bf4722f10b2f7552fc2585ab/raw/c9f616e98e504991b40cdd8f0c2626d1cee1e9a0/clashhead.yaml\&custom\=https://gist.githubusercontent.com/zhenlonghe/bf7bd95be0734903957e05be0bde44e5/raw/42c5db5bb99f1847ded5a6f7e4e6b9b20e597e19/clashrule


# download the dashboard
wget -c https://github.com/haishanh/yacd/releases/download/v0.3.3/yacd.tar.xz
tar -xvJf yacd.tar.xz
mv public/ dashboard

# Systemd 服务脚本
vi /lib/systemd/system/clash@.service

[Unit]
Description=A rule based proxy in Go for %i.
After=network.target

[Service]
Type=simple
User=%i
Restart=on-abort
ExecStart=/usr/local/bin/clash

[Install]
WantedBy=multi-user.target

# 重载 systemd 模块
systemctl daemon-reload
# 开机自启
systemctl enable clash@root 
# 启动服务
systemctl start clash@root
# 重启服务
systemctl restart clash@root



```


3. 配置路由表, 为 Debian 10 系统开启 IP 转发


```bash
sudo echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf && sysctl -p
sudo echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf && sysctl -p

cat /proc/sys/net/ipv4/ip_forward




iptables -t nat -N clash
iptables -t nat -N clash_dns

iptables -t nat -A PREROUTING -j clash_dns
iptables -t nat -A PREROUTING -j clash

iptables -t nat -A POSTROUTING -j SNAT --to-source 192.168.50.10

iptables -t nat -A clash_dns -p udp --dport 53 -j DNAT --to-destination 192.168.50.10:1053
iptables -t nat -A clash_dns -p tcp --dport 53 -j DNAT --to-destination 192.168.50.10:1053

iptables -t nat -A clash -d 0.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 10.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 127.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 169.254.0.0/16 -j RETURN
iptables -t nat -A clash -d 172.16.0.0/12 -j RETURN
iptables -t nat -A clash -d 192.168.0.0/16 -j RETURN
iptables -t nat -A clash -d 224.0.0.0/4 -j RETURN
iptables -t nat -A clash -d 240.0.0.0/4 -j RETURN
iptables -t nat -A clash -p tcp -j REDIRECT --to-ports 7892


# 如果配置乱了，下面可初始化 iptables 这将正确地将iptables系统完全重置为非常基本的状态：
iptables-save | awk '/^[*]/ { print $1 } 
                     /^:[A-Z]+ [^-]/ { print $1 " ACCEPT" ; }
                     /COMMIT/ { print $0; }' | iptables-restore
```
---

参考链接

[https://github.com/Dreamacro/clash/releases/tag/premium](https://github.com/Dreamacro/clash/releases/tag/premium) clash 下载

[https://lancellc.gitbook.io/clash/](https://lancellc.gitbook.io/clash/) 手册

[https://pengjiayou.com/debian-10-transparent-gateway](https://pengjiayou.com/debian-10-transparent-gateway)

[https://github.com/Dreamacro/clash/issues/655](https://github.com/Dreamacro/clash/issues/655)
[https://861204.xyz/archives/internet/](https://861204.xyz/archives/internet/)

[https://www.reverberation.cn/300/](https://www.reverberation.cn/300/)

[https://www.rinvay.cc/mip/719](https://www.rinvay.cc/mip/719)

[https://blog.caogo.cn/2020/07/21/基于树莓派4B构建支持透明代理的辅助路由器/](https://blog.caogo.cn/2020/07/21/%E5%9F%BA%E4%BA%8E%E6%A0%91%E8%8E%93%E6%B4%BE4B%E6%9E%84%E5%BB%BA%E6%94%AF%E6%8C%81%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E7%9A%84%E8%BE%85%E5%8A%A9%E8%B7%AF%E7%94%B1%E5%99%A8/)

[https://cherysunzhang.com/2020/05/deploy-clash-as-transparent-proxy-on-raspberry-pi/](https://cherysunzhang.com/2020/05/deploy-clash-as-transparent-proxy-on-raspberry-pi/)

[https://a-wing.top/linux/2019/04/01/debian_network.html](https://a-wing.top/linux/2019/04/01/debian_network.html)

[https://ksana410.github.io/2019/07/17/router/](https://ksana410.github.io/2019/07/17/router/)

clash 配置例子

[https://raw.githubusercontent.com/Hackl0us/SS-Rule-Snippet/master/LAZY_RULES/clash.yaml](https://raw.githubusercontent.com/Hackl0us/SS-Rule-Snippet/master/LAZY_RULES/clash.yaml)


```
aes 算力跑分
```bash
qopenssl speed -multi 8 -evp aes-128-gcm
openssl speed -evp aes-128-gcm

#r2s
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes  16384 bytes
aes-128-gcm      82106.91k   239600.66k   452797.01k   587645.95k   639983.62k   638709.28k

#n4020
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes  16384 bytes
aes-128-gcm     272476.63k   718622.97k  1218789.97k  1445190.66k  1509141.16k  1530768.04k

#i5 macbook
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes
aes-128-gcm     473798.53k  1068703.37k  1608161.10k  1843266.59k  1895455.37k

#m1 macbook
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes
aes-128-gcm     169339.23k   169057.99k   167172.67k   166883.98k   166284.50k
```

---
N4020 Auto Power On

Please set:
1.press F7 repeatedly after power on.
2.Enter BIOS --- BOOT --- Auto Power on --- Enable
3.Press F4 save and exit

[https://store.minisforum.com/products/minisforum-n40-mini-pc](https://store.minisforum.com/products/minisforum-n40-mini-pc)
