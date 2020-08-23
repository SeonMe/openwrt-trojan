注意：在下仅在原版 [OpenWrt](https://github.com/openwrt/openwrt) 上进行实践，其他修改版本不保证兼容性，需要一定的 Linux 基础.

## 1、安装 Trojan 服务端

请阁下自行查阅 [Trojan Wiki](https://trojan-gfw.github.io/trojan/)

## 2、编译 Trojan for OpenWrt

#### 2.1、下载对应版本 [OpenWrt SDK](http://downloads.openwrt.org/releases/)

#### 2.2、git clone [openwrt-trojan](https://github.com/trojan-gfw/openwrt-trojan)

#### 2.3、自行研究 or 下载在下编译好的版本。[OpenWrt-19.07.3](https://github.com/SeonMe/openwrt-trojan/raw/master/file/trojan_1.15.1-1_x86_64.ipk)

## 3、安装 Trojan

#### 3.1、`opkg install trojan_*.ipk` （如安装失败请下载 `file` 目录下的所有文件逐个安装。）

### 3.2、开启 Trojan

将 `/etc/config/trojan` 里的 `0` 改为 `1` 既可。

### 3.3、Trojan 配置

请阁下自行查阅 [Trojan Wiki](https://trojan-gfw.github.io/trojan/)


## 4、解决 DNS 污染问题

首先这是在下使用的 DNS 解析方案：

- **dnscrypt-proxy 2** ：支持 DoH 既 DNS over HTTPS，作为本地无污染上游 DNS 解析。
- **dnsmasq-full** ：作为 dnsmasq-china-list 和 gfwlist 本地解析，前者是交给如 `119.29.29.29` 解析后直连，后者则转发给 `dnscrypt-proxy 2` 解析并代理。
- **ipset** ：将中国大陆 IP 列表加入到 ipset 集合，匹配后直连。

注：OpenWrt-19.07.0 之前仓库没有 dnscrypt-proxy 2

2.1、安装 dnscrypt-proxy 等相关应用

首先下载 dnsmasq-full 包到本地：[下载地址](https://downloads.openwrt.org/releases/19.07.3/packages/x86_64/base/dnsmasq-full_2.80-16.1_x86_64.ipk)，然后上传至 OpenWrt，您也可以直接下载到 OpenWrt 任一目录，看您方便。

然后先安装 dnsmasq-full （从仓库里安装而不是下载的包）

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

2.1.1、配置 dnscrypt-proxy 2

您可以修改 dnscrypt-proxy 2 的配置文件，或者使用我提供的这份，dnscrypt-proxy 2 配置文件位于 `/etc/dnscrypt-proxy2/dnscrypt-proxy.toml`，您可以删除改目录下的所有文件再新建 `dnscrypt-proxy.toml` 亦可，dnscrypt-proxy 2 仅用于解析非大陆域名，可以做到无污染。

```
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

# forwarding_rules = 'dnscrypt-forwarding-rules.txt'
# cloaking_rules = 'dnscrypt-cloaking-rules.txt'

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

#[blacklist]
#  blacklist_file = 'dnscrypt-blacklist-domains.txt'
#  log_file = 'dnscrypt-blacklist-domains.log'
#  log_format = 'tsv'

#[ip_blacklist]
#  blacklist_file = 'dnscrypt-blacklist-ips.txt'
#  log_file = 'dnscrypt-blacklist-ips.log'
#  log_format = 'tsv'

#[whitelist]
#  whitelist_file = 'dnscrypt-whitelist.txt'
#  log_file = 'dnscrypt-whitelisted.log'
#  log_format = 'tsv'

[static]
  [static.'cloudflare']
  stamp = 'sdns://AgcAAAAAAAAABzEuMC4wLjEgPaS3FjSkE8HO80qpbSWkAWNNTPNsOxM8dClOSOY3ASoSZG5zLmNsb3VkZmxhcmUuY29tCi9kbnMtcXVlcnk'
  [static.'google']
  stamp = 'sdns://AgUAAAAAAAAABzguOC44LjigHvYkz_9ea9O63fP92_3qVlRn43cpncfuZnUWbzAMwbkgdoAkR6AZkxo_AEMExT_cbBssN43Evo9zs5_ZyWnftEUKZG5zLmdvb2dsZQovZG5zLXF1ZXJ5'
  [static. 'cisco']
  stamp = 'sdns://AQAAAAAAAAAADjIwOC42Ny4yMjAuMjIwILc1EUAgbyJdPivYItf9aR6hwzzI1maNDL4Ev6vKQ_t5GzIuZG5zY3J5cHQtY2VydC5vcGVuZG5zLmNvbQ'
```

2.1.2、重启 dnscrypt-proxy

```
/etc/init.d/dnscrypt-proxy restart
```

2.2、配置 dnsmasq

由于还需要使用 dnsmasq 来做国内的解析，所以需要做如下设置。

新建一个目录用于存放国内域名信息的文件，给予 dnsmasq 来解析：

```
mkdir /etc/dnsmasq.d
```

2.2.1、应用设置并添加缓存

```
uci add_list dhcp.@dnsmasq[0].confdir=/etc/dnsmasq.d
uci add_list dhcp.@dnsmasq[0].cachesize=10000
uci commit dhcp
```

2.2.2、下载相应的脚本

```
mkdir -p /opt/shell && cd /opt/shell
# China Domain List
curl -L -o generate_dnsmasq_chinalist.sh https://github.com/cokebar/openwrt-scripts/raw/master/generate_dnsmasq_chinalist.sh
chmod +x generate_dnsmasq_chinalist.sh
```

2.2.3、生成相应的文件

```
# China Domain List
sh generate_dnsmasq_chinalist.sh -d 119.29.29.29 -p 53 -s chinalist -o /etc/dnsmasq.d/accelerated-domains.china.conf
# Restart dnsmasq
/etc/init.d/dnsmasq restart
```

2.2.4、下载中国大陆的 IP 列表

```
mkdir -p /opt/chnroute
wget -O- 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /opt/chnroute/chnroute.txt
```

2.3、登入 OpwnWrt WEB 端修改相应选项

登入您的 openwrt 网页端进行修改以下选项：

2.3.1、DNS 转发

![](https://github.com/SeonMe/openwrt-trojan/raw/master/images/1.png)

2.3.2、注释解析文件

![](https://github.com/SeonMe/openwrt-trojan/raw/master/images/2.png)

2.4、添加 iptables 规则

iptables 规则可以在 OpenWrt > Network > Firewall > Custom Rules 添加，添加前请删除里面所有的内容，注意修改您的节点 IP 和 Trojan 透明代理端口。

```
iptables -t nat -N trojan
iptables -t nat -A trojan -d 您的节点 IP -j RETURN
iptables -t nat -A trojan -d 0.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 10.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 127.0.0.0/8 -j RETURN
iptables -t nat -A trojan -d 169.254.0.0/16 -j RETURN
iptables -t nat -A trojan -d 172.16.0.0/12 -j RETURN
iptables -t nat -A trojan -d 192.168.0.0/16 -j RETURN
iptables -t nat -A trojan -d 224.0.0.0/4 -j RETURN
iptables -t nat -A trojan -d 240.0.0.0/4 -j RETURN
# Delete China Domain List
iptables -t nat -D trojan -m set --match-set chinalist dst -j RETURN
# Delete China IP
iptables -t nat -D trojan -m set --match-set chnroute dst -j RETURN
ipset destroy
ipset create chinalist hash:net
ipset create chnroute hash:net
for i in `cat /opt/chnroute/chnroute.txt`;
do
 ipset add chnroute $i
done
# Add China Domain List
iptables -t nat -A trojan -m set --match-set chinalist dst -j RETURN
# Add China IP
iptables -t nat -A trojan -m set --match-set chnroute dst -j RETURN
iptables -t nat -A trojan -p tcp -j REDIRECT --to-ports POST #您的透明代理端口
iptables -t nat -A PREROUTING -p tcp -j trojan

```

解释：

> `# Delete & Add China Domain List` 部分是删除/添加前面 2.2.3 生成的文件的 IP 集，该文件主要用于国内网址并交由 `119.29.29.29` 去解析，得到的 IP 将会添加到 IP 集 `chinalist` 中，iptables 将会直连该集里的所有的 IP，顾名思义就是国内直连。

> `# Delete & Add China IP` 部分同样是删除/添加，这里是前面 2.2.4 下载的 IP 列表，所有在这个 `chnroute ` 集里的 IP 都将全部直连，既国内 IP 不走代理。

到这里您应该可以使用了，所有国内的 IP 都将采用直连的方式，国外 IP 将会全部走代理，简称国内外分流。

## 3、进阶用法

如果您需要更进一步，例如白名单，黑名单等等，您可以接着看以下部分。

3.1、白名单

顾名思义就是您不希望有一些国外的域名走代理，那么您可以在 `/etc/dnsmasq.d` 目录中添加 `whitelist.conf`，内容如下：

```
server=/domain1/119.29.29.29#53
ipset=/domain1/whitelist
server=/domain2/119.29.29.29#53
ipset=/domain2/whitelist
```

并在原有的 iptables 规则的基础上，在以下部分进行添加，有 3 条规则：

```
...
# Delete China IP
iptables -t nat -D trojan -m set --match-set chnroute dst -j RETURN
# Delete Whitelist
iptables -t nat -D trojan -m set --match-set whitelist dst -j RETURN
ipset destroy
ipset create chinalist hash:net
ipset create whitelist hash:net
...
# Add China IP
iptables -t nat -A trojan -m set --match-set chnroute dst -j RETURN
# Add Whitelist
iptables -t nat -A trojan -m set --match-set whitelist dst -j RETURN
...
```

3.2、黑名单

这里的黑名单是指内网的黑名单，默认情况下局域网内所有的连接都将会进行国内外分流，如果您希望您的某个设备直连国内外，那么您只需要在原有的 iptables 基础上添加以下部分：

```
...
# Delete China IP
iptables -t nat -D trojan -m set --match-set chnroute dst -j RETURN
# Delete Blacklist
iptables -t nat -D trojan -m set --match-set blacklist src -j RETURN
...
ipset create chinalist hash:net
ipset create blacklist hash:net
ipset add blacklist 内网 IP/32
...
# Add China IP
iptables -t nat -A trojan -m set --match-set chnroute dst -j RETURN
# Add Blacklist
iptables -t nat -A trojan -m set --match-set blacklist src -j RETURN
...
```

3.3、强制走代理

例如您需要某个域名强制走代理，如 Apple 的服务等等，那么您可以在 `/etc/dnsmasq.d` 目录中添加 `proxy-domain.conf`，内容如下：

```
server=/domain1/127.0.0.1#5353
ipset=/domain1/proxydomain
server=/domain2/127.0.0.1#5353
ipset=/domain2/proxydomain
```

以及在原有的 iptables 规则上修改如下：

```
...
iptables -t nat -A trojan -p tcp -m set --match-set proxydomain dst -j REDIRECT --to-ports POST # POST 为您的透明代理端口
...
```

3.4、仅允许内网某些 IP 或者 IP 段走代理

这里仅需要修改原有 iptables 规则既可，和前面 3.2 黑名单类似，修改如下：

```
# 仅允许某个 IP 段走代理，如 10.0.0.1/28
iptables -t nat -A trojan -p tcp -s 10.0.0.1/28 -j REDIRECT --to-ports POST # POST 为您的透明代理端口

# 仅允许特定的内网 IP 走代理
...
# Delete China IP
iptables -t nat -D trojan -m set --match-set chnroute dst -j RETURN
# Delete Blacklist
iptables -t nat -D trojan -m set --match-set lanproxylist src -j RETURN
...
ipset create chinalist hash:net
ipset create lanproxylist hash:net
ipset add lanproxylist 内网 IP/32
...
iptables -t nat -A trojan -p tcp -m set --match-set lanproxylist src -j REDIRECT --to-ports POST # POST 为您的透明代理端口
...
```
## 更新

2020.5.22 更新 trojan_1.15.1

## 引用

[generate_dnsmasq_chinalist.sh](https://github.com/cokebar/openwrt-scripts/raw/master/generate_dnsmasq_chinalist.sh)
