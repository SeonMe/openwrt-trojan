前提：此教程仅在原版 OpenWRT 进行实操，所有配置均基于配置文件或者命令行进行配置

## 1、Trojan

首先自行编译 openwrt-trojan，可自行前往 [这里](http://downloads.openwrt.org/releases/) 下载对应 OpenWrt 版本的 SDK 自行编译，如何编译请自行研究，这里不赘述。

如果您是 OpenWrt-19.07.0-rc2 可下载我编译好的版本：[基于 19.07.0-rc2 SDK 编译](https://seon.me/downloads/trojan_1.14.0-1_x86_64.ipk)

至于配置文件，Trojan 文档已有，这里不再赘述，安装 openwrt-trojan 后需要修改的地方有如下：

1.1、开启 Trojan

`vim /etc/config/trojan` 将 `0` 修改为 `1`，既开启 Trojan；

1.2、替换/修改 Trojan 配置

将您的 `trojan.json` 配置文件替换 `/etc/trojan.json` 文件，请使用 A valid nat.json 部份进行修改，地址：[点这里](https://trojan-gfw.github.io/trojan/config)

## 2、OpenWrt

首先需要解决 DNS 问题，这里使用 dnscrypt-proxy 2 作为非大陆无污染 DNS 解析方案，dnsmasq 做国内解析方案，由于 OpenWrt-19.07.0-rc2 官方库里已经有了 dnscrypt-proxy 2，故仅需要在 OpwnWrt 里安装既可。

2.1、安装 dnscrypt-proxy 等相关应用

```
opkg remove dnsmasq
opkg install dnsmasq-full ipset dnscrypt-proxy2 coreutils-base64 ca-certificates ca-bundle curl
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
  stamp = 'sdns://AgcAAAAAAAAABzEuMC4wLjGgENk8mGSlIfMGXMOlIlCcKvq7AVgcrZxtjon911-ep0cg63Ul-I8NlFj4GplQGb_TTLiczclX57DvMV8Q-JdjgRgSZG5zLmNsb3VkZmxhcmUuY29tCi9kbnMtcXVlcnk'
  [static.'google']
  stamp = 'sdns://AgUAAAAAAAAABzguOC44LjigHvYkz_9ea9O63fP92_3qVlRn43cpncfuZnUWbzAMwbkgdoAkR6AZkxo_AEMExT_cbBssN43Evo9zs5_ZyWnftEUKZG5zLmdvb2dsZQovZG5zLXF1ZXJ5'
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
mkdir /opt/shell && cd /opt/shell
# China Domain List
curl -L -o generate_dnsmasq_chinalist.sh https://github.com/cokebar/openwrt-scripts/raw/master/generate_dnsmasq_chinalist.sh
chmod +x generate_dnsmasq_chinalist.sh
```

2.2.3、生成相应的文件

```
# China Domain List
sh generate_dnsmasq_chinalist.sh -d 119.29.29.29 -p 53 -s chinalist-o /etc/dnsmasq.d/accelerated-domains.china.conf
# Restart dnsmasq
/etc/init.d/dnsmasq restart
```

2.2.4、下载中国大陆的 IP 列表

```
mkdir /opt/chnroute
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
iptables -t nat -A trojan -d 你的节点 IP -j RETURN
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
# Delete China
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
iptables -t nat -A trojan -p tcp -j REDIRECT --to-ports POST #你的透明代理端口
iptables -t nat -A PREROUTING -p tcp -j trojan

```

解释：

> `# Delete & Add China Domain List` 部分是删除/添加前面 2.2.3 生成的文件的 IP 集，该文件主要用于国内网址并交由 `119.29.29.29` 去解析，得到的 IP 将会添加到 IP 集 `chinalist` 中，iptables 将会直连该集里的所有的 IP，顾名思义就是国内直连。

> `# Delete & Add China IP` 部分同样是删除/添加，这里是前面 2.2.4 下载的 IP 列表，所有在这个 `chnroute ` 集里的 IP 都将全部直连，既国内 IP 不走代理。

到这里你应该可以使用了，所有国内的 IP 都将采用直连的方式，国外 IP 将会全部走代理，简称国内外分流。

## 3、进阶用法

如果你需要更进一步，例如白名单，黑名单，gfwlist 等等，您可以接着看以下部分。

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

例如你需要某个域名强制走代理，如 Apple 的服务等等，那么您可以在 `/etc/dnsmasq.d` 目录中添加 `proxy-domain.conf`，内容如下：

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
