注意：在下仅在原版 [OpenWrt](https://github.com/openwrt/openwrt) 上进行实践，其他修改版本不保证兼容性，需要一定的 Linux 基础.

## 1、安装 Trojan 服务端

请阁下自行查阅 [Trojan Wiki](https://trojan-gfw.github.io/trojan/)

## 2、编译 Trojan for OpenWrt

#### 2.1、下载对应版本 [OpenWrt SDK](http://downloads.openwrt.org/releases/)

#### 2.2、git clone [openwrt-trojan](https://github.com/trojan-gfw/openwrt-trojan)

#### 2.3、自行研究 or 下载在下编译好的版本。[OpenWrt-19.07.3](https://github.com/SeonMe/openwrt-trojan/raw/master/file/trojan_1.15.1-1_x86_64.ipk)

## 3、安装 Trojan

#### 3.1、SSH 登入 OpenWrt

`opkg install trojan_*.ipk` （如安装失败请下载 [file](https://github.com/SeonMe/openwrt-trojan/tree/master/file) 目录下的所有文件逐个安装。）

#### 3.2、开启 Trojan

将 `/etc/config/trojan` 里的 `0` 改为 `1` 既可。

### 3.3、Trojan 配置

请阁下自行查阅 [Trojan Wiki](https://trojan-gfw.github.io/trojan/)


## 4、解决 DNS 污染问题

首先这是在下使用的 DNS 解析方案：

- **dnscrypt-proxy 2** ：支持 DoH 既 DNS over HTTPS，作为本地无污染上游 DNS 解析。
- **dnsmasq-full** ：作为 dnsmasq-china-list 和 gfwlist 本地解析，前者是交给如 `119.29.29.29` 解析后直连，后者则转发给 `dnscrypt-proxy 2` 解析并代理。
- **ipset** ：将中国大陆 IP 列表加入到 ipset 集合，匹配后直连。

注：OpenWrt-19.07.0 之前仓库没有 dnscrypt-proxy 2

#### 4.1、安装 dnscrypt-proxy 等相关应用

##### 4.1.1、下载 [dnsmasq-full](https://downloads.openwrt.org/releases/19.07.3/packages/x86_64/base/dnsmasq-full_2.80-16.1_x86_64.ipk) 包到 OpenWrt 本地。

##### 4.1.2、安装 dnsmasq-full （从 OpenWrt 仓库里安装，不是上一步下载的。）

```
opkg install dnsmasq-full
```

然后会收到报错信息，原因是文件冲突，自然的 `dnsmasq-full` 也不会安装成功，接下来要做的就是卸载 `dnsmasq` 并安装 `dnsmasq-full`

```
opkg remove dnsmasq
opkg install dnsmasq-full*.ipk （前面下载下来的包）
```

之后再安装其他的包

```
opkg install ipset dnscrypt-proxy2 coreutils-base64 ca-certificates ca-bundle curl iptables-mod-tproxy
```

#### 4.2、配置 dnscrypt-proxy 2

如阁下对 `dnscrypt-proxy 2` 了解，请自行研究。

以下是在下目前使用的配置，可直接使用。

```
rm /etc/dnscrypt-proxy2/*

cat > /etc/dnscrypt-proxy2/dnscrypt-proxy.toml << EOF

listen_addresses = ['127.0.0.1:5353']
max_clients = 250
ipv4_servers = true
ipv6_servers = false
dnscrypt_servers = true
doh_servers = true
require_dnssec = true
require_nolog = true
require_nofilter = true
disabled_server_names = []
force_tcp = true
timeout = 2500
keepalive = 30
refused_code_in_responses = false
lb_strategy = 'p2'
lb_estimator = true
log_level = 0
log_file = 'dnscrypt-proxy.log'
use_syslog = false
cert_refresh_delay = 240
dnscrypt_ephemeral_keys = false
tls_disable_session_tickets = false
tls_cipher_suite = [52392, 49199]
fallback_resolver = '9.9.9.9:53'
ignore_system_dns = true
netprobe_timeout = 60
netprobe_address = "9.9.9.9:53"
log_files_max_size = 10
log_files_max_age = 7
log_files_max_backups = 1
block_ipv6 = true

cache = true
cache_size = 512
cache_min_ttl = 600
cache_max_ttl = 86400
cache_neg_min_ttl = 60
cache_neg_max_ttl = 600

[query_log]
  file = 'dnscrypt-query.log'
  format = 'tsv'

[nx_log]
  file = 'dnscrypt-nxdomain.log'
  format = 'tsv'

[static]
  [static.'cloudflare']
  stamp = 'sdns://AgcAAAAAAAAABzEuMC4wLjEgPaS3FjSkE8HO80qpbSWkAWNNTPNsOxM8dClOSOY3ASoSZG5zLmNsb3VkZmxhcmUuY29tCi9kbnMtcXVlcnk'
  [static.'google']
  stamp = 'sdns://AgUAAAAAAAAABzguOC44LjigHvYkz_9ea9O63fP92_3qVlRn43cpncfuZnUWbzAMwbkgdoAkR6AZkxo_AEMExT_cbBssN43Evo9zs5_ZyWnftEUKZG5zLmdvb2dsZQovZG5zLXF1ZXJ5'
  [static. 'cisco']
  stamp = 'sdns://AQAAAAAAAAAADjIwOC42Ny4yMjAuMjIwILc1EUAgbyJdPivYItf9aR6hwzzI1maNDL4Ev6vKQ_t5GzIuZG5zY3J5cHQtY2VydC5vcGVuZG5zLmNvbQ'
EOF
```

##### 4.2.1、重启 dnscrypt-proxy 2 后完成该部分

```
/etc/init.d/dnscrypt-proxy restart
```

#### 4.3、配置 dnsmasq

新建 `dnsmasq.d` 用于存放 `dnsmasq-china-list` 或者 `gfwlist` 的文件，并开启缓存。

```
mkdir /etc/dnsmasq.d
uci add_list dhcp.@dnsmasq[0].confdir=/etc/dnsmasq.d
uci add_list dhcp.@dnsmasq[0].cachesize=10000
uci commit dhcp
```

##### 4.3.1、下载用于生成 `dnsmasq_chinalist` 脚本和 `gfwlist` 规则脚本

```
mkdir -p /opt/shell && cd /opt/shell

# 大陆白名单模式可用
curl -L -o generate_dnsmasq_chinalist.sh https://github.com/cokebar/openwrt-scripts/raw/master/generate_dnsmasq_chinalist.sh
chmod +x generate_dnsmasq_chinalist.sh

sh generate_dnsmasq_chinalist.sh -d 119.29.29.29 -p 53 -s chinalist -o /etc/dnsmasq.d/accelerated-domains.china.conf
/etc/init.d/dnsmasq restart

# GfwList 强制走代理可用
curl -L -o gfwlist2dnsmasq.sh https://github.com/cokebar/gfwlist2dnsmasq/raw/master/gfwlist2dnsmasq.sh
chmod +x gfwlist2dnsmasq.sh

sh gfwlist2dnsmasq.sh -d 127.0.0.1 -p 5353 -s gfwlist -o /etc/dnsmasq.d/dnsmasq_gfwlist.conf
```

##### 4.3.2、下载中国大陆的 IP 列表，大陆白名单模式可用

```
mkdir -p /opt/chnroute
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /opt/chnroute/chnroute.txt
```

## 5、OpenWrt WebUI 配置

#### 5.1、DNS 转发

![](https://github.com/SeonMe/openwrt-trojan/raw/master/images/1.png)

#### 5.2、注释解析文件

![](https://github.com/SeonMe/openwrt-trojan/raw/master/images/2.png)

#### 5.3、iptables 规则

iptables 规则在 OpenWrt > Network > Firewall > Custom Rules 添加。

添加前请删除里面所有的内容，注意修改阁下的节点 IP 和 Trojan 透明代理端口。

##### 5.3.1、大陆白名单模式

```
iptables -t nat -N trojan
iptables -t nat -A trojan -d 节点 IP -j RETURN

# 本地白名单部分
iptables -t nat -A trojan -d 0.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 10.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 127.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 169.254.0.0/16 -j RETURN
iptables -t nat -A trojan -d 172.16.0.0/12 -j RETURN
iptables -t nat -A trojan -d 192.168.0.0/16 -j RETURN
iptables -t nat -A trojan -d 224.0.0.0/4 -j RETURN
iptables -t nat -A trojan -d 240.0.0.0/4 -j RETURN

# 删除 ipset 集合 chinalist 的引用，即 dnsmasq_chinalist
iptables -t nat -D trojan -m set --match-set chinalist dst -j RETURN

# 删除 ipset 集合 chnroute 的引用，即中国大陆 IP 列表
iptables -t nat -D trojan -m set --match-set chnroute dst -j RETURN

# 清空 ipset 集合并添加新的集合
ipset destroy
ipset create chinalist hash:net
ipset create chnroute hash:net

# 将中国大陆 IP 列表导入集合 chnroute
for i in `cat /opt/chnroute/chnroute.txt`;
do
 ipset add chnroute $i
done

# 增加 ipset 集合 chinalist 的引用，即 dnsmasq_chinalist
iptables -t nat -A trojan -m set --match-set chinalist dst -j RETURN

# 增加 ipset 集合 chnroute 的引用，即中国大陆 IP 列表
iptables -t nat -A trojan -m set --match-set chnroute dst -j RETURN

iptables -t nat -A trojan -p tcp -j REDIRECT --to-ports POST #透明代理端口

iptables -t nat -A PREROUTING -p tcp -j trojan

```

##### 5.3.2、gfwlist 模式

```
iptables -t nat -N trojan
iptables -t nat -A trojan -d 节点 IP -j RETURN

# 本地白名单部分
iptables -t nat -A trojan -d 0.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 10.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 127.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 169.254.0.0/16 -j RETURN
iptables -t nat -A trojan -d 172.16.0.0/12 -j RETURN
iptables -t nat -A trojan -d 192.168.0.0/16 -j RETURN
iptables -t nat -A trojan -d 224.0.0.0/4 -j RETURN
iptables -t nat -A trojan -d 240.0.0.0/4 -j RETURN

# 删除 ipset 集合 gfwlist 的引用，即 gfwlist 规则列表
iptables -t nat -D trojan -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口

# 清空 ipset 集合，并添加 gfwlist 集合
ipset destroy
ipset create gfwlist hash:net

iptables -t nat -A trojan -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口

iptables -t nat -A trojan -p tcp -j RETURN
iptables -t nat -A PREROUTING -p tcp -j trojan
```

### 5.4、附加部分（可选）

请注意 iptables 规则顺序，iptables 规则是由上而下进行匹配，以下规则请阁下清楚新增在哪些位置。

##### 5.4.1、国外白名单

如果阁下不希望有一些国外的域名走代理，那么您可以在 `/etc/dnsmasq.d` 目录中添加 `whitelist.conf`，内容如下：

```
server=/domain1/119.29.29.29#53
ipset=/domain1/whitelist
server=/domain2/119.29.29.29#53
ipset=/domain2/whitelist
```

并在原有的 iptables 规则的基础上，在以下部分进行添加，有 3 条规则：

```
...
iptables -t nat -D trojan -m set --match-set whitelist dst -j RETURN
ipset create whitelist hash:net
iptables -t nat -A trojan -m set --match-set whitelist dst -j RETURN
...
```

##### 5.4.2、国外黑名单

如果阁下使用 gfwlist 模式，可参照 `/etc/dnsmasq.d/dnsmasq_gfwlist.conf` 文件格式添加对应域名既可。

否则则按照以下方式：

```
vim /etc/dnsmasq.d/blacklist.conf

server=/domain1/127.0.0.1#5353
ipset=/domain1/blacklist
server=/domain2/127.0.0.1#5353
ipset=/domain2/blacklist
```

新增 iptables 规则

```
...
iptables -t nat -D trojan -p tcp -m set --match-set blacklist dst -j REDIRECT --to-ports POST #透明代理端口
ipset create blacklist hash:net
iptables -t nat -A trojan -p tcp -m set --match-set blacklist dst -j REDIRECT --to-ports POST #透明代理端口
...
```

##### 5.4.3、内网黑名单，即不走代理的终端

使用场景：PT、BT、室友、女朋友。

```
...
iptables -t nat -A trojan -s IP1/32 -j RETURN
iptables -t nat -A trojan -s IP2/32 -j RETURN
...
```

也可以

```
iptables -t nat -D trojan -m set --match-set localwhitelist src -j RETURN

ipset create localwhitelist hash:net

ipset add localwhitelist IP1/32
ipset add localwhitelist IP2/32

iptables -t nat -A trojan -m set --match-set localwhitelist src -j RETURN
```

#### 5.4.4、内网白名单，即可以走代理的终端

使用场景：如阁下只想自己独食，并分给某些合谋者。

仅允许某个 IP 段走代理，如 10.0.0.1/28

```
# 白名单模式
iptables -t nat -A trojan -p tcp -s 10.0.0.1/28 -j REDIRECT --to-ports POST #透明代理端口

# gfwlist 模式
iptables -t nat -A trojan -p tcp -s 10.0.0.1/28 -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口
```

仅允许特定 IP 走代理

```
# 白名单模式
iptables -t nat -A trojan -p tcp -s IP/32 -j REDIRECT --to-ports POST #透明代理端口

或者

iptables -t nat -D trojan -p tcp -m set --match-set lanproxylist src -j REDIRECT --to-ports POST #透明代理端口

ipset create lanproxylist hash:net
ipset add lanproxylist IP1/32
ipset add lanproxylist IP2/32

iptables -t nat -A trojan -p tcp -m set --match-set lanproxylist src -j REDIRECT --to-ports POST #透明代理端口


# gfwlist 模式
iptables -t nat -D trojan -p tcp -s IP/32 -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口

或者

iptables -t nat -D trojan -p tcp -m set --match-set lanproxylist src -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口

ipset create lanproxylist hash:net
ipset add lanproxylist IP1/32
ipset add lanproxylist IP2/32

iptables -t nat -A trojan -p tcp -m set --match-set lanproxylist src -m set --match-set gfwlist dst -j REDIRECT --to-ports POST #透明代理端口
```

## 更新

2020.8.24 新增 gfwlist 模式，并对以为不完善部分进行修改
2020.5.22 更新 trojan_1.15.1

## 引用脚本

[generate_dnsmasq_chinalist.sh](https://github.com/cokebar/openwrt-scripts/raw/master/generate_dnsmasq_chinalist.sh)
[gfwlist2dnsmasq.sh](https://github.com/cokebar/gfwlist2dnsmasq/raw/master/gfwlist2dnsmasq.sh)
