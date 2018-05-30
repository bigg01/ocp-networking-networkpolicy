# ingress

user --> G LB 8080 --> (X-Real-IP) --> 8024 --> 4444 (webserver)

# ip white list


## Global LB
Must forward Clietn IP. We ware setting the source IP to the Header
$ sudo /usr/sbin/haproxy -f /etc/haproxy/haproxy6.cfg -d
### http-request add-header X-CLIENT-IP %[src]
```sh
global
        log 127.0.0.1 local2
        chroot /var/lib/haproxy
        #user root
        #group root
        #daemon
        debug

defaults
        log global
        option dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000

frontend filter_http
        bind :8080
        #bind :8080 transparent
        default_backend filter_app_http

backend filter_app_http
        mode http
        option forwardfor
        option http-server-close
        http-request add-header X-CLIENT-IP %[src]
        #source 0.0.0.0 usesrc clientip
        server filter1 10.0.0.4:8024 check
```


## OCP LB under Global LB
We do a ACL for X-Forwarded-For
$ /usr/sbin/haproxy -f /etc/haproxy/haproxy2.cfg -d
### acl network_allowed hdr_ip(X-Forwarded-For) 10.0.0.4 10.0.0.208 10.0.0.0/24
```sh
global
        # uid 99
        # gid 99
        # log 127.0.0.1 local4
        maxconn 40000
        ulimit-n 80013
defaults
        log global
        option  httplog
        # option  dontlognull
        mode http
        contimeout 4000
        clitimeout 42000
        srvtimeout 43000
        balance roundrobin

frontend www
  bind *:8024
  mode http
  # acl network_allowed hdr_ip(X-Forwarded-For) 10.0.0.0/24
  acl network_allowed hdr_ip(X-Forwarded-For) 10.0.0.4 10.0.0.208
  tcp-request inspect-delay 5s
  http-request deny  if !network_allowed
  use_backend my_backend_server

  #acl network_allowed src 10.0.0.208
  #http-request deny if { req.hdr_ip(X-CLIENT-IP,-1) -f /etc/haproxy-ddos/blacklists/AF.txt }

backend my_backend_server
  server  S1 10.0.0.208:4444 check inter 2000 fall 3
```


## Client forward Ips 
### simple test
$ curl -v  10.0.0.4:8080
```sh
curl -v  10.0.0.4:8080
* Rebuilt URL to: 10.0.0.4:8080/
*   Trying 10.0.0.4...
* TCP_NODELAY set
* Connected to 10.0.0.4 (10.0.0.4) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.0.0.4:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Wed, 30 May 2018 14:31:04 GMT
< Content-Length: 18
< Content-Type: text/plain; charset=utf-8
<
* Connection #0 to host 10.0.0.4 left intact
Hi there, I love !
```
### log Global LB
```sh
00000000:filter_http.accept(0004)=0006 from [10.0.0.208:63442]
00000000:filter_app_http.clireq[0006:ffffffff]: GET / HTTP/1.1
00000000:filter_app_http.clihdr[0006:ffffffff]: Host: 10.0.0.4:8080
00000000:filter_app_http.clihdr[0006:ffffffff]: User-Agent: curl/7.54.0
00000000:filter_app_http.clihdr[0006:ffffffff]: Accept: */*
00000000:filter_app_http.srvrep[0006:0007]: HTTP/1.1 200 OK
00000000:filter_app_http.srvhdr[0006:0007]: Date: Wed, 30 May 2018 14:31:04 GMT
00000000:filter_app_http.srvhdr[0006:0007]: Content-Length: 18
00000000:filter_app_http.srvhdr[0006:0007]: Content-Type: text/plain; charset=utf-8
00000000:filter_app_http.srvhdr[0006:0007]: Connection: close
00000000:filter_app_http.srvcls[0006:0007]
00000001:filter_app_http.clicls[0006:ffffffff]
00000001:filter_app_http.closed[0006:ffffffff]
```

### OCP LB under Global LB
```sh
00000000:www.accept(0004)=0005 from [10.0.0.4:35246]
00000000:www.clireq[0005:ffffffff]: GET / HTTP/1.1
00000000:www.clihdr[0005:ffffffff]: Host: 10.0.0.4:8080
00000000:www.clihdr[0005:ffffffff]: User-Agent: curl/7.54.0
00000000:www.clihdr[0005:ffffffff]: Accept: */*
00000000:www.clihdr[0005:ffffffff]: X-CLIENT-IP: 10.0.0.208
00000000:www.clihdr[0005:ffffffff]: X-Forwarded-For: 10.0.0.208
00000000:www.clihdr[0005:ffffffff]: Connection: close
00000000:my_backend_server.srvrep[0005:0006]: HTTP/1.1 200 OK
00000000:my_backend_server.srvhdr[0005:0006]: Date: Wed, 30 May 2018 14:31:04 GMT
00000000:my_backend_server.srvhdr[0005:0006]: Content-Length: 18
00000000:my_backend_server.srvhdr[0005:0006]: Content-Type: text/plain; charset=utf-8
00000000:my_backend_server.srvhdr[0005:0006]: Connection: close
00000000:my_backend_server.srvcls[0005:0006]
00000000:my_backend_server.clicls[0005:0006]
00000000:my_backend_server.closed[0005:0006]
```

### OCP www server log
```go
&{GET / HTTP/1.1 1 1 map[User-Agent:[curl/7.54.0] Accept:[*/*] X-Client-Ip:[10.0.0.208] X-Forwarded-For:[10.0.0.208] Connection:[close]] {} <nil> 0 [] true 10.0.0.4:8080 map[] map[] <nil> map[] 10.0.0.4:49614 / <nil> <nil> <nil> 0xc4201ce000}
```


### overright REAL IP
$ curl -v --header "X-Client-Ip: 10.0.0.x" 10.0.0.4:8080
```go
&{GET / HTTP/1.1 1 1 map[Accept:[*/*] X-Client-Ip:[10.0.0.x 10.0.0.208] X-Forwarded-For:[10.0.0.208] Connection:[close] User-Agent:[curl/7.54.0]] {} <nil> 0 [] true 10.0.0.4:8080 map[] map[] <nil> map[] 10.0.0.4:49864 / <nil> <nil> <nil> 0xc4201ce300}

```

$ go run http.go
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
    fmt.Println(r)
}

func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":4444", nil))
}
```

### deny example

```sh
curl -v  10.0.0.4:8080
* Rebuilt URL to: 10.0.0.4:8080/
*   Trying 10.0.0.4...
* TCP_NODELAY set
* Connected to 10.0.0.4 (10.0.0.4) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.0.0.4:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 403 Forbidden
< Cache-Control: no-cache
< Content-Type: text/html
<
<html><body><h1>403 Forbidden</h1>
Request forbidden by administrative rules.
</body></html>
* Closing connection 0
```
