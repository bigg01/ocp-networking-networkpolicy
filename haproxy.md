# ingress

# ip white list


## Global LB
$ /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -d
```
#lobal
#    maxconn 100000
#    uid 99
#    gid 99
#    daemon

#https://rafpe.ninja/2016/05/01/haproxy-acl-on-x-forwarded-for/

# https://raymii.org/s/snippets/haproxy_restrict_specific_urls_to_specific_ip_addresses.html

# https://www.haproxy.com/blog/whats-new-in-haproxy-1-6/

# https://www.haproxy.com/blog/haproxy-log-customization/

# https://github.com/joyent/haproxy-1.4/blob/master/examples/acl-content-sw.cfg

defaults
    option forwardfor except 127.0.0.1
    mode    http
    contimeout  5000
    clitimeout  50000
    srvtimeout  50000

listen  myWeb
    bind :8080
    mode http
    balance source
    #option forwardfor
    option http-server-close
    option forwardfor
    option forwardfor header X-Real-IP # real IP
    server  S1 localhost:8024 check inter 2000 fall 3
```


## LB under Global LB
$ /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -d
```
```


## Client forward Ips 
### simple test
$ curl -v  10.0.0.4:8080
```
* Rebuilt URL to: 10.0.0.4:8080/
*   Trying 10.0.0.4...
* TCP_NODELAY set
* Connected to 10.0.0.4 (10.0.0.4) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.0.0.4:8080
> User-Agent: curl/7.51.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Tue, 29 May 2018 20:01:38 GMT
< Content-Length: 18
< Content-Type: text/plain; charset=utf-8
<
* Curl_http_done: called premature == 0
* Connection #0 to host 10.0.0.4 left intact
Hi there, I love !
```
### log
```
0000001:myWeb.accept(0004)=0006 from [127.0.0.1:56688]
00000001:myWeb.clireq[0006:ffffffff]: GET / HTTP/1.1
00000001:myWeb.clihdr[0006:ffffffff]: Host: 10.0.0.4:8080
00000001:myWeb.clihdr[0006:ffffffff]: User-Agent: curl/7.51.0
00000001:myWeb.clihdr[0006:ffffffff]: Accept: */*
00000001:myWeb.clihdr[0006:ffffffff]: X-Real-IP: 10.0.0.4
00000001:myWeb.clihdr[0006:ffffffff]: Connection: close
00000001:myWeb.srvrep[0006:0007]: HTTP/1.1 200 OK
00000001:myWeb.srvhdr[0006:0007]: Date: Tue, 29 May 2018 20:01:38 GMT
00000001:myWeb.srvhdr[0006:0007]: Content-Length: 18
00000001:myWeb.srvhdr[0006:0007]: Content-Type: text/plain; charset=utf-8
00000001:myWeb.srvhdr[0006:0007]: Connection: close
00000001:myWeb.srvcls[0006:0007]
00000001:myWeb.clicls[0006:0007]
00000001:myWeb.closed[0006:0007]
```

### overright REAL IP
$ curl --header "X-Real-IP: 10.0.0.xxx"-v  10.0.0.4:8080
```
&{GET / HTTP/1.1 1 1 map[User-Agent:[curl/7.54.0] Accept:[*/*] X-Real-Ip:[10.0.0.xxx-v 10.0.0.208] Connection:[close]] {} <nil> 0 [] true 10.0.0.4:8080 map[] map[] <nil> map[] 10.0.0.4:56042 / <nil> <nil> <nil> 0xc42012a240}

```
