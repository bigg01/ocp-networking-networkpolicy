global
        log /dev/log    local0
        log /dev/log    local1 notice
        #chroot /var/lib/haproxy
        #stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        #user haproxy
        #group haproxy
        #daemon
        # Default SSL material locations
        #ca-base /etc/ssl/certs
        #crt-base /etc/ssl/private
        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL).
        #ssl-default-bind-ciphers kEECDH+aRSA+AES:kRSA+AES:+AES256:RC4-SHA:!kEDH:!LOW:!EXP:!MD5:!aNULL:!eNULL
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
        stats auth admin:password
        stats uri  /haproxy?stats



frontend localnodes
    bind *:8080
    mode http
    default_backend nodes


backend nodes
    mode http
    balance leastconn
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    #option httpchk
    server idx2 192.168.64.2:80 check 

 frontend localnodes-ssl
    bind *:8443
    mode http
    default_backend nodes
backend nodes-ssl
    mode http
    balance leastconn
    option forwardfor
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option http-server-close
    http-request add-header X-CLIENT-IP %[src]
    #option httpchk
    server idx1 192.168.64.2:443 ssl verify none check 
 

#backend tomcat-cp-events
#    mode tcp
#    option tcp-smart-connect
#    server tomcat :54600 send-proxy
