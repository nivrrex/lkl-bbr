# lkl-bbr
BBRPlus for OpenVZ with LKL(Linux Kernel Library)

### 说明
lkl的最新5.4和5.10版本，原版基础上打了bbrplus补丁（感谢UJX6N）

编译时仅在默认配置基础上开启了BBR和BBRPlus，同时strip相应信息

主要给自己的OpenVZ小鸡使用，主机需开通TUN/TAP

### 5.4 版本
https://github.com/lkl/linux

https://github.com/UJX6N/bbrplus-5.4

### 5.10 版本
https://github.com/ngi-mptcp/lkl-next/

https://github.com/UJX6N/bbrplus-5.10


### 编译命令
下载解压，进入源码目录，打好补丁后

```
cat << \EOF >> ./arch/lkl/configs/defconfig
CONFIG_TCP_CONG_ADVANCED=y
CONFIG_TCP_CONG_BBR=y
CONFIG_DEFAULT_BBR=y
CONFIG_TCP_CONG_BBRPLUS=y
CONFIG_DEFAULT_BBRPLUS=y
CONFIG_DEFAULT_TCP_CONG="bbrplus"
EOF
```

```
make -C tools/lkl -j 4
strip --strip-unneeded ./tools/lkl/lib/hijack/liblkl-hijack.so
```

### 使用方式
lkl-hijack.json文件方式，调用haproxy进行端口转发

openvz 7 下的 Debian 10 系统测试的最高支持 haproxy 2.0 版本

#### 创建 lkl-haproxy 文件夹
```
mkdir /etc/lkl-haproxy
```

#### Debian 10 下安装 2.0 版本 haproxy
```
curl https://haproxy.debian.net/bernat.debian.org.gpg | gpg --dearmor > /usr/share/keyrings/haproxy.debian.net.gpg
echo deb "[signed-by=/usr/share/keyrings/haproxy.debian.net.gpg]" http://haproxy.debian.net buster-backports-2.0 main > /etc/apt/sources.list.d/haproxy.list

apt-get update
apt-get install haproxy=2.0.\* -y
systemctl stop haproxy
systemctl disable haproxy
```

#### lkl-hijack.json 文件配置
```
cat << \EOF > /etc/lkl-haproxy/lkl-hijack.json
{
  "gateway":"10.0.0.1",
  "singlecpu":"1",
  "sysctl":"net.ipv4.tcp_congestion_control=bbrplus",
  "sysctl":"net.ipv4.tcp_rmem=4096 87380 4194304",
  "sysctl":"net.ipv4.tcp_wmem=4096 16384 4194304",
  "sysctl":"net.ipv4.tcp_mem=94500000 915000000 927000000",
  "sysctl":"net.ipv4.tcp_sack=1",
  "sysctl":"net.ipv4.tcp_slow_start_after_idle=0",
  "boot_cmdline": "mem=256m",
  "interfaces":[
          {
                  "mac":"12:34:56:78:9a:bc",
                  "qdisc":"root|fq",
                  "type":"tap",
                  "param":"lkl-tap",
                  "ip":"10.0.0.2",
                  "masklen":"24",
                  "ifgateway":"10.0.0.1",
                  "offload":"0x8883"
          }
  ]
}
EOF
```

#### haproxy 配置
```
cat << \EOF > /etc/lkl-haproxy/haproxy.cfg
global
defaults
    log global
    mode tcp
    option dontlognull
    option tcpka
    timeout connect 5000
    timeout client  50000
    timeout server  50000
frontend proxy-in
    bind :443
    default_backend proxy-out
backend proxy-out
    server lkl-tap 10.0.0.1 maxconn 20480
EOF
```

#### tap及iptables转发设置 （感谢mzz2017/lkl-haproxy脚本）
```
cat << \EOF > /etc/lkl-haproxy/redirect.sh
#!/bin/sh
ip tuntap del lkl-tap mode tap > /dev/null 2>&1 || true
ip tuntap add lkl-tap mode tap
ip addr add 10.0.0.1/24 dev lkl-tap
ip link set lkl-tap up
sysctl -w net.ipv4.ip_forward=1
iptables -P FORWARD ACCEPT
iptables -t nat -D POSTROUTING -o $(awk '$2 == 00000000 { print $1 }' /proc/net/route) -j MASQUERADE > /dev/null 2>&1 || true
iptables -t nat -A POSTROUTING -o $(awk '$2 == 00000000 { print $1 }' /proc/net/route) -j MASQUERADE
iptables -t nat -I PREROUTING -i venet0 -p tcp --dport 443 -j DNAT --to-destination 10.0.0.2

LKL_HIJACK_CONFIG_FILE=/etc/lkl-haproxy/lkl-hijack.json LD_PRELOAD=/etc/lkl-haproxy/liblkl-hijack.so /usr/sbin/haproxy -f /etc/lkl-haproxy/haproxy.cfg
exit 0
EOF
chmod +x /etc/lkl-haproxy/redirect.sh
```

#### 创建systemd服务
```
cat << \EOF > /etc/systemd/system/lkl-haproxy.service
[Unit]
Description=lkl-haproxy
After=network.target nss-lookup.target
Wants=network.target nss-lookup.target
StartLimitIntervalSec=0

[Service]
Environment=''
ExecStart=/etc/lkl-haproxy/redirect.sh
Type=simple
KillMode=process
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
EOF
```

#### 设置systemd服务
```
systemctl enable lkl-haproxy
systemctl start lkl-haproxy
```

### 备注
以上脚本已经在 OpenVZ 7 虚拟机下 Debian 10 测试成功，不计划提供一键脚本
