# 

### we want to access TCP thru ingress OCP Router
 - option 1: Ingress CIDR External IP -- external loadbalancer
 
   can't work because external Ip works only on Node Level
   
 - option 2: build uniq ingress Router for TCP - port by annotation
 
```haproxy
     ############### tkggo tcp raw test
# frontend
{{ range $cfgIdx, $cfg := .State }}

{{- if (isTrue (index $cfg.Annotations "haproxy.router.openshift.io/rawtcp")) }}

  frontend public_tcp_raw_{{$cfgIdx}}

      {{- if ne (env "ROUTER_SYSLOG_ADDRESS") ""}}

    option tcplog

      {{- end }}
    # haproxy.router.openshift.io/bindport: '9200'
    # haproxy.router.openshift.io/rawtcp: 'true'

    {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/bindport")) }}

    bind :{{ index $cfg.Annotations "haproxy.router.openshift.io/bindport" }}

        {{- end }}

    default_backend be_tcp_raw{{$cfgIdx}}


# backend 
 backend be_tcp_raw{{$cfgIdx}}
 mode tcp

 {{- range $serviceUnitName, $weight := $cfg.ServiceUnitNames }}



        {{- with $serviceUnit := index $.ServiceUnits $serviceUnitName }}

          {{- range $idx, $endpoint := processEndpointsForAlias $cfg $serviceUnit ""}}

    server {{$endpoint.ID}} {{$endpoint.IP}}:{{$endpoint.Port}} 

                {{- end }}

              {{- end }}

     {{- end }}
{{end}}
{{end}} # end range
  ################### tkggo eng raw
```
   
 - option 3: using nodeports on the Router nodes forwarded with a service
   could work

 - option 4: create an openssl Server - and forward over SNI to TCP endpoint
    https://gitlab.com/aleks001/openssl-server/tree/master

 - option 5: uniq port on ingress router per annotation forward
### lua auth
https://github.com/TimWolla/haproxy-auth-request/blob/master/auth-request.lua
### basic auth by annotation
https://blog.sleeplessbeastie.eu/2018/03/08/how-to-define-basic-authentication-on-haproxy/
 
### oauth2 by annotation 
https://github.com/bigg01/haproxy-ingress/blob/master/examples/auth/oauth/oauth2-proxy.yaml

### sni routing
https://www.haproxy.com/de/blog/enhanced-ssl-load-balancing-with-server-name-indication-sni-tls-extension/

### modsecurity by annotation?
https://github.com/bigg01/haproxy-ingress/tree/master/examples/modsecurity

```https://www.haproxy.com/de/blog/enhanced-ssl-load-balancing-with-server-name-indication-sni-tls-extension/
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


#### add cusom config OCP HAPROXY
```
4  oc rsh router-1-65hz6 -- cat haproxy-config.template > haproxy-config.template
 1035  oc rsh router-1-65hz6  cat haproxy-config.template > haproxy-config.template
 1036  oc rsh router-1-65hz6 cat haproxy.config > haproxy.config
 1037  oc set env dc/router \
 1038  oc create configmap customrouter --from-file=haproxy-config.template
 1039  oc volume dc/router --add --overwrite     --name=config-volume     --mount-path=/var/lib/haproxy/conf/custom     --source='{"configMap": { "name": "customrouter"}}'
 1040  oc set env dc/router     TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template

```

```haproxy
############# guo
# /var/lib/haproxy/conf/custom/haproxy-config.template
frontend guomysql
    bind :3306
    mode tcp
    default_backend guomysqlbackend

backend guomysqlbackend
    mode tcp
    # IP:172.30.140.51 Hostname: mysql.dbtest.svc
    server guomysqlbackend 172.30.140.51:3306
    
############# guo end
    
```


```
mysql -u test1 -p -h 192.168.42.44 
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 413
Server version: 5.7.21 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> use testdb;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [testdb]> show tables;
+------------------+
| Tables_in_testdb |
+------------------+
| MyGuests         |
+------------------+
1 row in set (0.001 sec)


```


## univversal sni redirect
https://github.com/amlweems/stun/blob/master/cmd/stuntcp/main.go


https://medium.com/@olivier.ragain/haproxy-https-load-balancing-on-sni-207c17398d19

https://github.com/opencoff/go-tunnel


```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  annotations:
    haproxy.router.openshift.io/bindport: '9200'
    haproxy.router.openshift.io/rawtcp: 'true'
    openshift.io/host.generated: 'true'
    
    
```


https://hub.helm.sh/
