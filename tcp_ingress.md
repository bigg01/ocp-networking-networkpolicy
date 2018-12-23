# 

### we want to access TCP thru ingress OCP Router
 - option 1: Ingress CIDR External IP -- external loadbalancer
 
   can't work because external Ip works only on Node Level
   
 - option 2: build uniq ingress Router for TCP - port by annotation
 
   PoC
   
 - option 3: using nodeports on the Router nodes forwarded with a service
   could work

 - option 4: create an openssl Server - and forward over SNI to TCP endpoint
    https://gitlab.com/aleks001/openssl-server/tree/master

 - option 5: uniq port on ingress router per annotation forward
 
 
### auth ba annotation 
https://github.com/bigg01/haproxy-ingress/blob/master/examples/auth/oauth/oauth2-proxy.yaml

```
global
  tune.ssl.default-dh-param 2048

defaults
  timeout connect 5000
  timeout client  50000
  timeout server  50000

frontend ssl
    mode tcp
    bind 0.0.0.0:443
    tcp-request inspect-delay 5s
    tcp-request content accept  if  HTTP
    use_backend ssh             if  { payload(0,7) -m bin 5353482d322e30 }
    use_backend main-ssl        if  { req.ssl_hello_type 1 }
    default_backend openvpn

frontend main
    bind 127.0.0.1:443 ssl crt /some/folder/cert.pem accept-proxy
    mode http
    option forwardfor
    default_backend webserver

frontend http
    bind 0.0.0.0:80
    reqadd X-Forwarded-Proto:\ http
    default_backend webserver

backend main-ssl
    mode tcp
    server main-ssl 127.0.0.1:443 send-proxy

backend openvpn
    mode tcp
    timeout server 2h
    server openvpn-localhost 127.0.0.1:1193

backend ssh
    mode tcp
    timeout server 2h
    server ssh-localhost 127.0.0.1:22

backend webserver
    mode http
    option forwardfor
    redirect scheme https code 301 if !{ ssl_fc }
    server webserver-localhost 127.0.0.1:81
   ```
