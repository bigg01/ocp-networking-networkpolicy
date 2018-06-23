https://github.com/bigg01/haproxycmd


cat haproxy6.cfg
```
global
        log 127.0.0.1:5555 local2 debug
        stats socket /tmp/haproxy.sock mode 666 level admin
        #chroot /var/lib/haproxy
        #user root
        #group root
        #daemon
        #debug


defaults
        log global
        #option dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
        mode tcp
        option tcplog


listen stats # Define a listen section called "stats"
  bind :9003 # Listen on localhost:9000
  mode http
  stats uri		/monitor
  stats refresh	5s
  option httpchk	GET /status
  retries		5
  option redispatch
  stats enable  # Enable stats page
  stats hide-version  # Hide HAProxy version
  stats realm Haproxy\ Statistics  # Title text for popup window
  #stats uri /haproxy_stats  # Stats URI
  #stats auth Username:Password  # Authentication credentials

frontend filter_http
        bind :8080
       # capture request header X-Haproxy-ACL len 64
       # capture request header X-Unique-ID len 64
        #https://www.haproxy.com/de/blog/haproxy-log-customization/
        #log-format "%ci:%cp %si [%t] %ft %b/%s %Tw/%Tc/%Tt %B '%ts' ac/%fc/%bc/%sc/%rc %sq/%bq: %[capture.req.hdr(0)]"
        log-format "Client: %ci:%cp %si [%t] %ft %b/%s %Tw/%Tc/%Tt %B '%ts' ac/%fc/%bc/%sc/%rc %sq/%bq]"
        # -f works but is not reloaded we should use maps
        # acl whitelist src -f /etc/haproxy/acl-map

        acl whitelist src [src,map_ip(/etc/haproxy/acl-map,0)]

# echo "show map /etc/haproxy/acl-map" |         sudo socat stdio /tmp/haproxy.sock
# echo "show acl" |         sudo socat stdio /tmp/haproxy.sock
# 0x564143b12bf0 10.0.0.4

        #tcp-request content reject if !whitelist
        #tcp-request content track-sc0 src
        use_backend not_in_acl if !whitelist
        use_backend filter_app_http if whitelist


backend not_in_acl
        #tcp-request content capture dst len 15
         #log-format "not in ACL: %[capture.req.hdr(0)]"
        tcp-request content reject

backend filter_app_http
        server filter1 10.0.0.208:4444 check
```

## socklog
```
./socklog  inet 0 5555
listening on 0.0.0.0:5555, starting.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy stats started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy filter_http started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy not_in_acl started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy filter_app_http started.
127.0.0.1: local2.info: Jun 23 15:31:01 haproxy[29893]: Client: 10.0.0.4:59410 10.0.0.208 [23/Jun/2018:15:31:01.582] filter_http filter_app_http/filter1 1/7/19 135 '--' ac/0/0/0/0 0/0]
```

# socat
````
echo "show info " |         sudo socat stdio /tmp/haproxy.sock
Name: HAProxy
Version: 1.6.13
Release_date: 2017/06/18
Nbproc: 1
Process_num: 1
Pid: 29893
Uptime: 0d 0h01m03s
Uptime_sec: 63
Memmax_MB: 0
Ulimit-n: 4033
Maxsock: 4033
Maxconn: 2000
Hard_maxconn: 2000
CurrConns: 0
CumConns: 15
CumReq: 15
MaxSslConns: 0
CurrSslConns: 0
CumSslConns: 0
Maxpipes: 0
PipesUsed: 0
PipesFree: 0
ConnRate: 0
ConnRateLimit: 0
MaxConnRate: 1
SessRate: 0
SessRateLimit: 0
MaxSessRate: 1
SslRate: 0
SslRateLimit: 0
MaxSslRate: 0
SslFrontendKeyRate: 0
SslFrontendMaxKeyRate: 0
SslFrontendSessionReuse_pct: 0
SslBackendKeyRate: 0
SslBackendMaxKeyRate: 0
SslCacheLookups: 0
SslCacheMisses: 0
CompressBpsIn: 0
CompressBpsOut: 0
CompressBpsRateLim: 0
ZlibMemUsage: 0
MaxZlibMemUsage: 0
Tasks: 8
Run_queue: 1
Idle_pct: 100
node: fedora-lamp
description:

$ echo "show errors " |         sudo socat stdio /tmp/haproxy.sock
Total events captured on [23/Jun/2018:15:32:08.713] : 0
```
