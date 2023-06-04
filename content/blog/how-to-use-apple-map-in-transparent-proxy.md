+++
title = "如何在透明代理网络环境下得到正常的苹果地图"
date = 2023-06-04
+++


# 背景

是这样的，我家的透明代理策略走的是路由器上配置路由表的策略，这个策略是中国大陆的地址直接走的我家的上游，非中国大陆的地址会被路由到我内网专门的代理机器出去, 剩下的中国大陆地址就会走默认的路由。然后DNS做了分流，非中国大陆的域名会默认走  `8.8.8.8`进行请求，正好这个 `8.8.8.8` 的请求正好会被路由匹配到我的代理机器，这样就会所有的非中国域名全部走远程解析并访问了，在不需要的时候直接把路由撤销所有的请求会自动跳转回正常的默认线路，DNS也完全不需要调整（`8.8.8.8` 正好会被默认路由匹配到）。

# 问题

上面的方案我在大部分情况下都工作的很好，但是我最近遇到了一个问题：我有两个苹果设备，我很喜欢的一个苹果的APP是苹果地图，它接入了很多地图提供商，比如中国大陆接入的是高德地图，这样一个地图就能保持中国大陆内外的一致性体验了（众所周知，在中国的地图提供商没办法看清楚中国大陆以外的地图，海外的地图提供商没办法看清中国大陆境内的地图）。在我家的分流中，苹果地图没办法使用高德德地图提供商，经过我一顿谷歌搜索和抓包，大致有以下结论：

1. v2ex里面提到苹果的  [`ls.apple.com`](http://ls.apple.com/) 这个域名是负责苹果地图判断地理位置数据集加载的接口，但是提到的解决方案不适用于我，因为我的代理策略并没有经过TLS Sniffer。

[使用 ClashX 代理， macOS 地图.app 无法调用高德的数据。需要添加什么规则才可以使用高德数据？ - V2EX](https://www.v2ex.com/t/682766)

1. 于是我尝试在 Adguard-Home 里面调了下DNS的规则

```
[/*.ls.apple.com/]114.114.114.114
```

然后发现根本不起作用，然后我看了下 DNS 的日志，发现苹果的DNS有很多 CNAME 到各个 CDN，这样我们就没办法通过 DNS 规则来指定上游解析了，因为我们并不知道苹果有多少 CNAME，并且这些CNAME什么时候会变。并且我就算匹配到了上游，但是因为很多 CDN 在中国大陆并没有接入点，所以最终的访问还是会走代理，这也有可能会影响苹果服务器的判断。

```
➜  ~ (root@archlinux-router) dig gsp-ssl.ls.apple.com @1.1.1.1

; <<>> DiG 9.18.15 <<>> gsp-ssl.ls.apple.com @1.1.1.1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9585
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;gsp-ssl.ls.apple.com.		IN	A

;; ANSWER SECTION:
gsp-ssl.ls.apple.com.	3582	IN	CNAME	gsp-ssl.ls-apple.com.akadns.net.
gsp-ssl.ls-apple.com.akadns.net. 12 IN	CNAME	gsp-ssl-geomap.ls-apple.com.akadns.net.
gsp-ssl-geomap.ls-apple.com.akadns.net.	42 IN CNAME gspx-ssl.ls.apple.com.
gspx-ssl.ls.apple.com.	3582	IN	CNAME	get-bx.g.aaplimg.com.
get-bx.g.aaplimg.com.	12	IN	A	17.253.5.208
get-bx.g.aaplimg.com.	12	IN	A	17.253.17.211

;; Query time: 129 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Sun Jun 04 14:44:12 CST 2023
;; MSG SIZE  rcvd: 209
```

# 解决方案

于是乎问题就变成了， 需要在我家的环境下，让访问苹果  `*.ls.apple.com` 域名的所有请求（包含 DNS 解析）不走任何的代理，咨询了一下朋友们，大致有以下几个方案

1. 写个脚本，手动解析`*.ls.apple.com`那几个域名的 IP 地址，然后放到 ip rule 或者其他路由表里。
    
    我并不能枚举所有的 `*.ls.apple.com` 的域名，而且他们的 CNAME 我更没办法枚举，他们后面的IP地址大概率是CDN的地址，我如果肆意的给这些地址使用特殊规则的话可能会影响我的其他请求访问（比如 Cloudflare 这种全部都是 BGP Anycast IP 的 CDN 厂商，给他的一个IP设置特殊的规则可能会影响到很大比例的网站访问）。
    
2. 使用 DNS 解析结果打 fwmark 的插件，然后给它特定的路由，就是那些人只用域名，不用 IP 列表来分流的方案。
    
    dnsmasq 支持操作  [ipset](https://man.archlinux.org/man/dnsmasq.8.en#ipset=/_domain__/_domain_..._/_ipset__,_ipset_..._) ，这样就可以根据DNS结果来让 iptables 给流量打上标记，然后指定查询路由表，但是这个方案还是有上面一样的问题，CDN不友好。
    
3. 内网搭建个 SNI Proxy, 在路由器上面 hosts 或者 DNS 服务配置里面将目标域名解析到我的的 SNI Proxy 服务上。
    
    这个方案看起来是可以的，于是我就采用了这个方案。
    

要实施上面的方案三，大致需要这几个步骤

1. 在内网完全不走任何代理的机器上部署一个SNI Proxy。
    
    使用 Nginx 的来搭建一个 SNI Proxy 服务器，重定向 80 和 443 的流量到需要的服务地址
    
    ```
    user http;
    worker_processes  1;
    
    error_log   /var/log/nginx/error.log;
    
    events {
        worker_connections  1024;
    }
    
    http {
        map $host $backend_name {
            default $host;
        }
        resolver 114.114.114.114 ipv6=off;
    
        access_log /var/log/nginx/sni_http_access.log;
    
        server {
            listen 80;
            location / {
                proxy_pass http://$backend_name;
                proxy_set_header Host $backend_name;
            }
        }
    }
    
    stream {
        map $ssl_preread_server_name $backend_name {
            default $ssl_preread_server_name;
        }
    
        resolver 114.114.114.114 ipv6=off;
    
        log_format proxy '$remote_addr [$time_local] $ssl_preread_server_name '
                   '$protocol $status $bytes_sent $bytes_received '
                   '$session_time "$upstream_addr" '
                   '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
        access_log /var/log/nginx/sni_https_access.log proxy;
    
        server {
           listen 443;
           ssl_preread on;
           proxy_timeout 5s;
           proxy_pass $backend_name:443;
        }
    }
    ```
    
    在路由器上关掉本机器的透明代理
    
    ```bash
    ip route add default via 10.0.150.254 dev enp3s0 table 8888
    ip rule add from 192.168.233.171 lookup 8888
    ```
    
1. 使用 DNS 转发器的 DNS 重写功能把 `*.ls.apple.com` 的请求全部重定向到上面那台机器。
    
    我直接在 Adguard-home 写入如下重定向规则
    
    ```
    *.ls.apple.com 192.168.233.171（
    ```
    

最后看检查一下日志和苹果设备上的地图，一切按照预期工作正常。

```bash
[root@archlinux-tool ~]# tail -f /var/log/nginx/sni_https_access.log
192.168.233.193 [04/Jun/2023:07:09:42 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.582 "23.34.208.4:443" "856" "4624" "0.185"
192.168.233.193 [04/Jun/2023:07:09:57 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.569 "23.34.208.4:443" "856" "4624" "0.180"
192.168.233.193 [04/Jun/2023:07:10:36 +0000] gspe1-ssl.ls.apple.com TCP 200 4773 598 7.844 "23.34.208.4:443" "1115" "4773" "0.189"
192.168.233.193 [04/Jun/2023:07:11:18 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.603 "23.34.208.4:443" "856" "4624" "0.192"
192.168.233.193 [04/Jun/2023:07:12:34 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.051 "23.45.60.145:443" "856" "4624" "0.034"
192.168.233.193 [04/Jun/2023:07:13:38 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.062 "23.45.60.145:443" "856" "4624" "0.057"
192.168.233.193 [04/Jun/2023:07:13:43 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.059 "23.45.60.145:443" "856" "4624" "0.053"
192.168.233.193 [04/Jun/2023:07:15:21 +0000] gsp64-ssl.ls.apple.com TCP 200 3851 1424 1.867 "17.36.206.7:443" "1941" "3851" "0.221"
192.168.233.193 [04/Jun/2023:07:16:15 +0000] gspe1-ssl.ls.apple.com TCP 200 4624 339 5.618 "23.34.208.4:443" "856" "4624" "0.197"
192.168.233.193 [04/Jun/2023:07:17:51 +0000] gsp64-ssl.ls.apple.com TCP 200 3853 1114 1.970 "17.36.206.8:443" "1631" "3853" "0.199"
```
