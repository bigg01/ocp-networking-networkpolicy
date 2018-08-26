```
global
        stats timeout 30s
defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
       
listen  stats   
        bind            *:1936
        mode            http
        log             global
        maxconn 10
            timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
        stats enable
        stats hide-version
        stats refresh 30s
        stats show-node
        stats uri  /stats



frontend  http-in
    bind *:80 
    mode http

    use_backend ocp
    
frontend  http-in-81
    bind *:81
    mode http

    use_backend ocp-chind



backend ocp
    mode http
    http-request set-header Host httpd-ex-test.192.168.42.219.nip.io
    server  httpd-ex-test.192.168.42.219.nip.io httpd-ex-test.192.168.42.219.nip.io:443 check ssl verify none

backend ocp-chind
    mode http
    http-request set-header Host www.chind-und-chegel.ch
    server  www.chind-und-chegel.ch www.chind-und-chegel.ch:443 check ssl verify none

```
