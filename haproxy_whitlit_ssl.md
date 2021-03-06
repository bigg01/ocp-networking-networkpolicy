# A10 setup
https://files.a10networks.com/vadc/forums/topic/determine-source-ip-and-port/

```
# with port
when HTTP_REQUEST {
HTTP::header insert "X-Test-Client" [IP::client_addr]:[TCP::local_port]
}

#without port
when HTTP_REQUEST {
HTTP::header insert “X-Forwarded-For” [IP::client_addr]
}
```


# IP Whitelisting:
```sh
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  annotations:
    haproxy.router.openshift.io/ip_whitelist: 10.0.0.4
    openshift.io/host.generated: 'true'
```


# GSLB LB:

```
PEM
```

```sh
global
        #log /dev/log    local0
        #log /dev/log    local1 notice
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
        #mode    http
        #option  httplog
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
    option  httplog
    default_backend nodes


backend nodes
    mode http
    balance leastconn
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-CLIENT-IP %[src]
    #option httpchk
    server idx2 192.168.64.2:80 check 

 frontend localnodes-ssl
    bind *:8443 ssl crt /Users/guo/server.pem no-sslv3
    #mode tcp

    # set HTTP Strict Transport Security (HTST) header
    rspadd  Strict-Transport-Security:\ max-age=15768000
    default_backend nodes-ssl

 frontend localnodes-ocp
    bind *:8555 ssl crt /Users/guo/ocp.pem no-sslv3
    #mode tcp

    # set HTTP Strict Transport Security (HTST) header
    rspadd  Strict-Transport-Security:\ max-age=15768000
    default_backend nodes-ssl-ocp

    

backend nodes-ssl-ocp
    mode http
    balance leastconn
    option forwardfor
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option http-server-close
    capture request header X-Forwarded-For len 50
    http-request set-header X-Client-IP %[src] #if !is_cf
    
    #http-request set-header X-Client-IP %[hdr(cf-connecting-ip)] if is_cf
    #option httpchk
    #server idx1 192.168.64.2:443 check #ssl verify none#ssl #verify none check 
    server idx1 192.168.64.2:443 check ssl verify none



backend nodes-ssl
    mode http
    balance leastconn
    option forwardfor
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option http-server-close
    
    capture request header X-Forwarded-For len 50

    #acl is_cf req.hdr(cf-connecting-ip) -m found

    http-request set-header X-Client-IP %[src] #if !is_cf
    #http-request set-header X-Client-IP %[hdr(cf-connecting-ip)] if is_cf
    #option httpchk
    #server idx1 192.168.64.2:443 check #ssl verify none#ssl #verify none check 
    server idx1 10.0.0.208:8444 check ssl verify none
 

#backend tomcat-cp-events
#    mode tcp
#    option tcp-smart-connect
#    server tomcat :54600 send-proxy
```

# Custom Router ACL
## ConfigMAP
```sh
## customrouter


$ oc create configmap customrouter --from-file=haproxy-config.template

$ oc volume dc/router --add --overwrite \
    --name=config-volume \
    --mount-path=/var/lib/haproxy/conf/custom \
    --source='{"configMap": { "name": "customrouter"}}'
$ oc set env dc/router \
    TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template
```

```sh
## customrouter


$ oc create configmap customrouter --from-file=haproxy-config.template

$ oc volume dc/router --add --overwrite \
    --name=config-volume \
    --mount-path=/var/lib/haproxy/conf/custom \
    --source='{"configMap": { "name": "customrouter"}}'
$ oc set env dc/router \
    TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template
```
{{- with $ip_whiteList := firstMatch $cidrListPattern (index $cfg.Annotations "haproxy.router.openshift.io/ip_whitelist") }}
  #acl whitelist src {{ $ip_whiteList }}
  acl whitelist hdr_ip(X-Forwarded-For) {{ $ip_whiteList }}
  tcp-request content reject if !whitelist
  
  
    {{- with $ip_whiteList := firstMatch $cidrListPattern (index $cfg.Annotations "haproxy.router.openshift.io/ip_whitelist") }}
  #acl whitelist src {{$ip_whiteList}}
  acl whitelist hdr_ip(X-Forwarded-For) {{ $ip_whiteList }}
  tcp-request content reject if !whitelist

```sh


{{/*
    haproxy-config.cfg: contains the main config with helper backends that are used to terminate
    					encryption before finally sending to a host_be which is the backend that is the final
    					backend for a route and contains all the endpoints for the service
*/}}
{{- define "/var/lib/haproxy/conf/haproxy.config" }}
{{- $workingDir := .WorkingDir }}
{{- $defaultDestinationCA := .DefaultDestinationCA }}
{{- $router_ip_v4_v6_mode := env "ROUTER_IP_V4_V6_MODE" "v4" }}


{{- /* A bunch of regular expressions.  Each should be wrapped in (?:) so that it is safe to include bare */}}
{{- /* quadPattern: Match a quad in an IP address; e.g. 123 */}}
{{- $quadPattern := `(?:[0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])` -}}

{{- /* ipPattern: Match an IPv4 address; e.g. 192.168.21.23 */}}
{{- $ipPattern := printf `(?:%s\.%s\.%s\.%s)` $quadPattern $quadPattern $quadPattern $quadPattern -}}

{{- /* cidrPattern: Match an IP and network size in CIDR form; e.g. 192.168.21.23/24 */}}
{{- $cidrPattern := printf `(?:%s(?:/(?:[0-9]|[1-2][0-9]|3[0-2]))?)` $ipPattern -}}

{{- /* cidrListPattern: Match a space separated list of CIDRs; e.g. 192.168.21.23/24 192.10.2.12 */}}
{{- $cidrListPattern := printf `(?:%s(?: +%s)*)` $cidrPattern $cidrPattern -}}

{{- /* cookie name pattern: */}}
{{- $cookieNamePattern := `[a-zA-Z0-9_-]+` -}}

{{- $timeSpecPattern := `[1-9][0-9]*(us|ms|s|m|h|d)?` }}

{{- /* hsts header in response: */}}
{{- $hstsOptionalTokenPattern := `(?:includeSubDomains|preload)` }}
{{- $hstsPattern := printf `(?:%[1]s[;])*max-age=(?:\d+|"\d+")(?:[;]%[1]s)*`  $hstsOptionalTokenPattern -}}

global
  maxconn {{env "ROUTER_MAX_CONNECTIONS" "20000"}}

  daemon
{{- with (env "ROUTER_SYSLOG_ADDRESS") }}
  log {{.}} {{env "ROUTER_LOG_FACILITY" "local1"}} {{env "ROUTER_LOG_LEVEL" "debug"}}
{{- end}}
  ca-base /etc/ssl
  crt-base /etc/ssl
  stats socket /var/lib/haproxy/run/haproxy.sock mode 600 level admin expose-fd listeners
  stats timeout 2m

  # Increase the default request size to be comparable to modern cloud load balancers (ALB: 64kb), affects
  # total memory use when large numbers of connections are open.
  tune.maxrewrite 8192
  tune.bufsize 32768

  # Prevent vulnerability to POODLE attacks
  ssl-default-bind-options no-sslv3

# The default cipher suite can be selected from the three sets recommended by https://wiki.mozilla.org/Security/Server_Side_TLS,
# or the user can provide one using the ROUTER_CIPHERS environment variable.
# By default when a cipher set is not provided, intermediate is used.
{{- if eq (env "ROUTER_CIPHERS" "intermediate") "modern" }}
  # Modern cipher suite (no legacy browser support) from https://wiki.mozilla.org/Security/Server_Side_TLS
  tune.ssl.default-dh-param 2048
  ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256
{{ else }}

  {{- if eq (env "ROUTER_CIPHERS" "intermediate") "intermediate" }}
  # Intermediate cipher suite (default) from https://wiki.mozilla.org/Security/Server_Side_TLS
  tune.ssl.default-dh-param 2048
  ssl-default-bind-ciphers ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS
  {{ else }}

    {{- if eq (env "ROUTER_CIPHERS" "intermediate") "old" }}

  # Old cipher suite (maximum compatibility but insecure) from https://wiki.mozilla.org/Security/Server_Side_TLS
  tune.ssl.default-dh-param 1024
  ssl-default-bind-ciphers ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:DES-CBC3-SHA:HIGH:SEED:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!RSAPSK:!aDH:!aECDH:!EDH-DSS-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA:!SRP

    {{- else }}
  # user provided list of ciphers (Colon separated list as seen above)
  # the env default is not used here since we can't get here with empty ROUTER_CIPHERS
  tune.ssl.default-dh-param 2048
  ssl-default-bind-ciphers {{env "ROUTER_CIPHERS" "ECDHE-ECDSA-CHACHA20-POLY1305"}}
    {{- end }}
  {{- end }}
{{- end }}

defaults
  maxconn {{env "ROUTER_MAX_CONNECTIONS" "20000"}}

  # Add x-forwarded-for header.
{{- if ne (env "ROUTER_SYSLOG_ADDRESS") "" }}
  {{- if ne (env "ROUTER_SYSLOG_FORMAT") "" }}
  log-format {{env "ROUTER_SYSLOG_FORMAT"}}
  {{- else }}
  option httplog
  {{- end }}
  log global
{{- end }}

  # To configure custom default errors, you can either uncomment the
  # line below (server ... 127.0.0.1:8080) and point it to your custom
  # backend service or alternatively, you can send a custom 503 error.
  #
  # server openshift_backend 127.0.0.1:8080
  errorfile 503 /var/lib/haproxy/conf/error-page-503.http

  timeout connect {{firstMatch $timeSpecPattern (env "ROUTER_DEFAULT_CONNECT_TIMEOUT") "5s"}}
  timeout client {{firstMatch $timeSpecPattern (env "ROUTER_DEFAULT_CLIENT_TIMEOUT") "30s"}}
  timeout client-fin {{firstMatch $timeSpecPattern (env "ROUTER_CLIENT_FIN_TIMEOUT") "1s"}}
  timeout server {{firstMatch $timeSpecPattern (env "ROUTER_DEFAULT_SERVER_TIMEOUT") "30s"}}
  timeout server-fin {{firstMatch $timeSpecPattern (env "ROUTER_DEFAULT_SERVER_FIN_TIMEOUT") "1s"}}
  timeout http-request {{firstMatch $timeSpecPattern (env "ROUTER_SLOWLORIS_TIMEOUT") "10s" }}
  timeout http-keep-alive {{firstMatch $timeSpecPattern (env "ROUTER_SLOWLORIS_HTTP_KEEPALIVE") "300s" }}

  # Long timeout for WebSocket connections.
  timeout tunnel {{firstMatch $timeSpecPattern (env "ROUTER_DEFAULT_TUNNEL_TIMEOUT") "1h" }}

{{- if isTrue (env "ROUTER_ENABLE_COMPRESSION") }}
  compression algo gzip
  compression type {{env "ROUTER_COMPRESSION_MIME" "text/html text/plain text/css"}}
{{- end }}

{{ if (gt .StatsPort -1) }}
listen stats
  bind :{{if (gt .StatsPort 0)}}{{.StatsPort}}{{else}}1936{{end}}
  mode http
  # Health check monitoring uri.
  monitor-uri /healthz


  # Add your custom health check monitoring failure condition here.
  # monitor fail if <condition>
  stats enable
  stats hide-version
  stats realm Haproxy\ Statistics
  stats uri /

{{- end }}

{{ if .BindPorts -}}
frontend public
    {{ if eq "v4v6" $router_ip_v4_v6_mode }}
  bind :::{{env "ROUTER_SERVICE_HTTP_PORT" "80"}} v4v6
    {{- else if eq "v6" $router_ip_v4_v6_mode }}
  bind :::{{env "ROUTER_SERVICE_HTTP_PORT" "80"}} v6only
    {{- else }}
  bind :{{env "ROUTER_SERVICE_HTTP_PORT" "80"}}
    {{- end }}
    {{- if isTrue (env "ROUTER_USE_PROXY_PROTOCOL") }} accept-proxy{{ end }}
  mode http
  tcp-request inspect-delay 5s
  tcp-request content accept if HTTP

  {{- if (eq .StatsPort -1) }}
  monitor-uri /_______internal_router_healthz
  {{- end }}

  # Strip off Proxy headers to prevent HTTpoxy (https://httpoxy.org/)
  http-request del-header Proxy

  # DNS labels are case insensitive (RFC 4343), we need to convert the hostname into lowercase
  # before matching, or any requests containing uppercase characters will never match.
  http-request set-header Host %[req.hdr(Host),lower]

  # check if we need to redirect/force using https.
  acl secure_redirect base,map_reg(/var/lib/haproxy/conf/os_route_http_redirect.map) -m found
  redirect scheme https if secure_redirect

  # Check if it is an edge or reencrypt route exposed insecurely.
  acl route_http_expose base,map_reg(/var/lib/haproxy/conf/os_route_http_expose.map) -m found
  use_backend %[base,map_reg(/var/lib/haproxy/conf/os_route_http_expose.map)] if route_http_expose

  # map to http backend
  # Search from most specific to general path (host case).
  # Note: If no match, haproxy uses the default_backend, no other
  #       use_backend directives below this will be processed.
  use_backend be_http:%[base,map_reg(/var/lib/haproxy/conf/os_http_be.map)]

  default_backend openshift_default

# public ssl accepts all connections and isn't checking certificates yet certificates to use will be
# determined by the next backend in the chain which may be an app backend (passthrough termination) or a backend
# that terminates encryption in this router (edge)
frontend public_ssl
    {{ if eq "v4v6" $router_ip_v4_v6_mode }}
  bind :::{{env "ROUTER_SERVICE_HTTPS_PORT" "443"}} v4v6
    {{- else if eq "v6" $router_ip_v4_v6_mode }}
  bind :::{{env "ROUTER_SERVICE_HTTPS_PORT" "443"}} v6only
    {{- else }}
  bind :{{env "ROUTER_SERVICE_HTTPS_PORT" "443"}}
    {{- end }}
    {{- if isTrue (env "ROUTER_USE_PROXY_PROTOCOL") }} accept-proxy{{ end }}
  tcp-request  inspect-delay 5s
  tcp-request content accept if { req_ssl_hello_type 1 }

  # if the connection is SNI and the route is a passthrough don't use the termination backend, just use the tcp backend
  # for the SNI case, we also need to compare it in case-insensitive mode (by converting it to lowercase) as RFC 4343 says
  acl sni req.ssl_sni -m found
  acl sni_passthrough req.ssl_sni,lower,map_reg(/var/lib/haproxy/conf/os_sni_passthrough.map) -m found
  use_backend be_tcp:%[req.ssl_sni,lower,map_reg(/var/lib/haproxy/conf/os_tcp_be.map)] if sni sni_passthrough

  # if the route is SNI and NOT passthrough enter the termination flow
  use_backend be_sni if sni

  # non SNI requests should enter a default termination backend rather than the custom cert SNI backend since it
  # will not be able to match a cert to an SNI host
  default_backend be_no_sni

##########################################################################
# TLS SNI
#
# When using SNI we can terminate encryption with custom certificates.
# Certs will be stored in a directory and will be matched with the SNI host header
# which must exist in the CN of the certificate.  Certificates must be concatenated
# as a single file (handled by the plugin writer) per the haproxy documentation.
#
# Finally, check re-encryption settings and re-encrypt or just pass along the unencrypted
# traffic
##########################################################################
backend be_sni
  server fe_sni 127.0.0.1:{{env "ROUTER_SERVICE_SNI_PORT" "10444"}} weight 1 send-proxy

frontend fe_sni
  # terminate ssl on edge
  bind 127.0.0.1:{{env "ROUTER_SERVICE_SNI_PORT" "10444"}} ssl no-sslv3
    {{- if isTrue (env "ROUTER_STRICT_SNI") }} strict-sni {{ end }}
    {{- ""}} crt {{firstMatch ".+" .DefaultCertificate "/var/lib/haproxy/conf/default_pub_keys.pem"}}
    {{- ""}} crt-list /var/lib/haproxy/conf/cert_config.map accept-proxy
  mode http

  # Strip off Proxy headers to prevent HTTpoxy (https://httpoxy.org/)
  http-request del-header Proxy

  # DNS labels are case insensitive (RFC 4343), we need to convert the hostname into lowercase
  # before matching, or any requests containing uppercase characters will never match.
  http-request set-header Host %[req.hdr(Host),lower]

  # check re-encrypt backends first - from most specific to general path.
  acl reencrypt base,map_reg(/var/lib/haproxy/conf/os_reencrypt.map) -m found

  # Search from most specific to general path (host case).
  use_backend be_secure:%[base,map_reg(/var/lib/haproxy/conf/os_reencrypt.map)] if reencrypt

  # map to http backend
  # Search from most specific to general path (host case).
  # Note: If no match, haproxy uses the default_backend, no other
  #       use_backend directives below this will be processed.
  use_backend be_edge_http:%[base,map_reg(/var/lib/haproxy/conf/os_edge_http_be.map)]

  default_backend openshift_default

##########################################################################
# END TLS SNI
##########################################################################

##########################################################################
# TLS NO SNI
#
# When we don't have SNI the only thing we can try to do is terminate the encryption
# using our wild card certificate.  Once that is complete we can either re-encrypt
# the traffic or pass it on to the backends
##########################################################################
# backend for when sni does not exist, or ssl term needs to happen on the edge
backend be_no_sni
  server fe_no_sni 127.0.0.1:{{env "ROUTER_SERVICE_NO_SNI_PORT" "10443"}} weight 1 send-proxy

frontend fe_no_sni
  # terminate ssl on edge
  bind 127.0.0.1:{{env "ROUTER_SERVICE_NO_SNI_PORT" "10443"}} ssl no-sslv3 crt {{firstMatch ".+" .DefaultCertificate "/var/lib/haproxy/conf/default_pub_keys.pem"}} accept-proxy
  mode http

  # Strip off Proxy headers to prevent HTTpoxy (https://httpoxy.org/)
  http-request del-header Proxy

  # DNS labels are case insensitive (RFC 4343), we need to convert the hostname into lowercase
  # before matching, or any requests containing uppercase characters will never match.
  http-request set-header Host %[req.hdr(Host),lower]

  # check re-encrypt backends first - path or host based.
  acl reencrypt base,map_reg(/var/lib/haproxy/conf/os_reencrypt.map) -m found

  # Search from most specific to general path (host case).
  use_backend be_secure:%[base,map_reg(/var/lib/haproxy/conf/os_reencrypt.map)] if reencrypt

  # map to http backend
  # Search from most specific to general path (host case).
  # Note: If no match, haproxy uses the default_backend, no other
  #       use_backend directives below this will be processed.
  use_backend be_edge_http:%[base,map_reg(/var/lib/haproxy/conf/os_edge_http_be.map)]

  default_backend openshift_default

##########################################################################
# END TLS NO SNI
##########################################################################

backend openshift_default
  mode http
  option forwardfor
  #option http-keep-alive
  option http-pretend-keepalive

##-------------- app level backends ----------------
{{/*
       1. If termination is not set: This is plain http -> http.  Create a be_http:<service> backend.
          Incoming http traffic is terminated and sent as http to the pods.

       2. If termination is type 'edge': This is https -> http.  Create a be_edge_http:<service> backend.
          Incoming https traffic is terminated and sent as http to the pods.

       3. If termination is type 'reencrypt': This is https -> https.  Create a be_secure:<service> backend.
          Incoming https traffic is terminated and then sent as https to the pods.

       4. If termination is type 'passthrough': This is https (or any SNI TLS connection) passthrough.
          Create a be_tcp:<service> backend.
          Incoming traffic is inspected to get the hostname from the SNI header, but then all traffic is
          passed through to the backend pod by just looking at the TCP headers.
*/}}
{{- range $cfgIdx, $cfg := .State }}
  {{- if matchValues (print $cfg.TLSTermination) "" "edge" "reencrypt" }}
    {{- if (eq $cfg.TLSTermination "") }}

# Plain http backend
backend be_http:{{$cfgIdx}}
    {{- else if (eq $cfg.TLSTermination "edge") }}

# Plain http backend but request is TLS, terminated at edge
backend be_edge_http:{{$cfgIdx}}
    {{ else if (eq $cfg.TLSTermination "reencrypt") }}

# Secure backend which requires re-encryption
backend be_secure:{{$cfgIdx}}
    {{- end }}{{/* end chceck for router type */}}
  mode http
  option redispatch
  option forwardfor

    {{- with $balanceAlgo := firstMatch "roundrobin|leastconn|source" (index $cfg.Annotations "haproxy.router.openshift.io/balance") (env "ROUTER_LOAD_BALANCE_ALGORITHM") }}
  balance {{ $balanceAlgo }}
    {{- else }}
  balance {{ if gt $cfg.ActiveServiceUnits 1 }}roundrobin{{ else }}leastconn{{ end }}
    {{- end }}
    {{- with $ip_whiteList := firstMatch $cidrListPattern (index $cfg.Annotations "haproxy.router.openshift.io/ip_whitelist") }}
  #acl whitelist src {{ $ip_whiteList }}
  acl whitelist hdr_ip(X-Forwarded-For) {{ $ip_whiteList }}
  tcp-request content reject if !whitelist
    {{- end }}
    {{- with $value := firstMatch $timeSpecPattern (index $cfg.Annotations "haproxy.router.openshift.io/timeout")}}
  timeout server  {{$value}}
    {{- end }}

    {{- if isTrue (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections") }}
  stick-table type ip size 100k expire 30s store conn_cur,conn_rate(3s),http_req_rate(10s)
  tcp-request content track-sc2 src
      {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp")) }}
  tcp-request content reject if { src_conn_cur ge  {{ index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp" }} }
      {{- else }}
  # concurrent TCP connections not restricted
      {{- end }}

      {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-tcp")) }}
  tcp-request content reject if { src_conn_rate ge {{ index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-tcp" }} }
      {{- else }}
  #TCP connection rate not restricted
      {{- end }}

      {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-http")) }}
  tcp-request content reject if { src_http_req_rate ge {{ index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-http" }} }
      {{- else }}
  #HTTP request rate not restricted
      {{- end }}
    {{- end }}

  timeout check 5000ms
  http-request set-header X-Forwarded-Host %[req.hdr(host)]
  http-request set-header X-Forwarded-Port %[dst_port]
  http-request set-header X-Forwarded-Proto http if !{ ssl_fc }
  http-request set-header X-Forwarded-Proto https if { ssl_fc }
  {{- if matchPattern "(v4)?v6" $router_ip_v4_v6_mode }}
  # See the quoting rules in https://tools.ietf.org/html/rfc7239 for IPv6 addresses (v4 addresses get translated to v6 when in hybrid mode)
  http-request set-header Forwarded for="[%[src]]";host=%[req.hdr(host)];proto=%[req.hdr(X-Forwarded-Proto)]
  {{- else }}
  http-request set-header Forwarded for=%[src];host=%[req.hdr(host)];proto=%[req.hdr(X-Forwarded-Proto)]
  {{- end }}

  {{- if not (isTrue (index $cfg.Annotations "haproxy.router.openshift.io/disable_cookies")) }}
  cookie {{firstMatch $cookieNamePattern (index $cfg.Annotations "router.openshift.io/cookie_name") (env "ROUTER_COOKIE_NAME" "") $cfg.RoutingKeyName}} insert indirect nocache httponly
    {{- if and (matchValues (print $cfg.TLSTermination) "edge" "reencrypt") (ne $cfg.InsecureEdgeTerminationPolicy "Allow") }} secure
    {{- end }}
  {{- end }}{{/* end disable cookies check */}}

  {{- if matchValues (print $cfg.TLSTermination) "edge" "reencrypt" }}
    {{- with $hsts := firstMatch $hstsPattern (index $cfg.Annotations "haproxy.router.openshift.io/hsts_header") }}
  http-response set-header Strict-Transport-Security {{$hsts}}
    {{- end }}{{/* hsts header */}}
  {{- end }}{{/* is "edge" or "reencrypt" */}}

  {{- range $serviceUnitName, $weight := $cfg.ServiceUnitNames }}
    {{- if ne $weight 0 }}
      {{- with $serviceUnit := index $.ServiceUnits $serviceUnitName }}
        {{- range $idx, $endpoint := processEndpointsForAlias $cfg $serviceUnit (env "ROUTER_BACKEND_PROCESS_ENDPOINTS" "") }}
  server {{$endpoint.ID}} {{$endpoint.IP}}:{{$endpoint.Port}} cookie {{$endpoint.IdHash}} weight {{$weight}}
          {{- if (eq $cfg.TLSTermination "reencrypt") }} ssl
            {{- if $cfg.VerifyServiceHostname }} verifyhost {{ $serviceUnit.Hostname }}
            {{- end }}
            {{- if gt (len (index $cfg.Certificates (printf "%s_pod" $cfg.Host)).Contents) 0 }} verify required ca-file {{ $workingDir }}/cacerts/{{$cfgIdx}}.pem
            {{- else }}
              {{- if gt (len $defaultDestinationCA) 0 }} verify required ca-file {{ $defaultDestinationCA }}
              {{- else }} verify none
              {{- end }}
            {{- end }}

          {{- else if or (eq $cfg.TLSTermination "") (eq $cfg.TLSTermination "edge") }}
          {{- end }}{{/* end type specific options*/}}

            {{- if and (not $endpoint.NoHealthCheck) (gt $cfg.ActiveEndpoints 1) }} check inter {{firstMatch $timeSpecPattern (index $cfg.Annotations "router.openshift.io/haproxy.health.check.interval") (env "ROUTER_BACKEND_CHECK_INTERVAL") "5000ms"}}
            {{- end }}{{/* end else no health check */}}
          {{- end }}{{/* end if cg.TLSTermination */}}
        {{- end }}{{/* end range processEndpointsForAlias */}}
      {{- end }}{{/* end get serviceUnit from its name */}}
  {{- end }}{{/* end range over serviceUnitNames */}}

  {{- end }}{{/* end if tls==edge/none/reencrypt */}}

  {{- if eq $cfg.TLSTermination "passthrough" }}

# Secure backend, pass through
backend be_tcp:{{$cfgIdx}}
{{- if ne (env "ROUTER_SYSLOG_ADDRESS") ""}}
  option tcplog
{{- end }}
    {{- with $balanceAlgo := firstMatch "roundrobin|leastconn|source" (index $cfg.Annotations "haproxy.router.openshift.io/balance") (env "ROUTER_LOAD_BALANCE_ALGORITHM") }}
  balance {{ $balanceAlgo }}
    {{- else }}
  balance {{ if gt $cfg.ActiveServiceUnits 1 }}roundrobin{{ else }}source{{ end }}
    {{- end }}
    {{- with $ip_whiteList := firstMatch $cidrListPattern (index $cfg.Annotations "haproxy.router.openshift.io/ip_whitelist") }}
  #acl whitelist src {{$ip_whiteList}}
  acl whitelist hdr_ip(X-Forwarded-For) {{ $ip_whiteList }}
  tcp-request content reject if !whitelist
    {{- end }}
    {{- with $value := firstMatch $timeSpecPattern (index $cfg.Annotations "haproxy.router.openshift.io/timeout")}}
  timeout tunnel  {{$value}}
    {{- end }}

{{- if isTrue (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections") }}
  stick-table type ip size 100k expire 30s store conn_cur,conn_rate(3s),http_req_rate(10s)
  tcp-request content track-sc2 src
  {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp")) }}
  tcp-request content reject if { src_conn_cur ge  {{ index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp" }} }
  {{- else }}
  # concurrent TCP connections not restricted
  {{- end }}

  {{- if (isInteger (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-tcp")) }}
  tcp-request content reject if { src_conn_rate ge {{ index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections.rate-tcp" }} }
  {{- else }}
  #TCP connection rate not restricted
  {{- end }}
{{- end }}

  hash-type consistent
  timeout check 5000ms
    {{- range $serviceUnitName, $weight := $cfg.ServiceUnitNames }}
      {{- if ne $weight 0 }}
        {{- with $serviceUnit := index $.ServiceUnits $serviceUnitName }}
          {{- range $idx, $endpoint := processEndpointsForAlias $cfg $serviceUnit (env "ROUTER_BACKEND_PROCESS_ENDPOINTS" "") }}
  server {{$endpoint.ID}} {{$endpoint.IP}}:{{$endpoint.Port}} weight {{$weight}}
            {{- if and (not $endpoint.NoHealthCheck) (gt $cfg.ActiveEndpoints 1) }} check inter {{firstMatch $timeSpecPattern (index $cfg.Annotations "router.openshift.io/haproxy.health.check.interval") (env "ROUTER_BACKEND_CHECK_INTERVAL") "5000ms"}}
            {{- end }}{{/* end else no health check */}}
          {{- end }}{{/* end range processEndpointsForAlias */}}
        {{- end }}{{/* end get ServiceUnit from serviceUnitName */}}
      {{- end }}{{/* end if weight != 0 */}}
    {{- end }}{{/* end iterate over services*/}}
  {{- end }}{{/*end tls==passthrough*/}}

{{- end }}{{/* end loop over routes */}}
{{- else }}
# Avoiding binding ports until routing configuration has been synchronized.
{{- end }}{{/* end bind ports after sync */}}
{{ end }}{{/* end haproxy config template */}}

{{/*--------------------------------- END OF HAPROXY CONFIG, BELOW ARE MAPPING FILES ------------------------*/}}
{{/*
    os_wildcard_domain.map: contains a mapping of wildcard hosts for a
			[sub]domain regexps. This map is used to check if
			a host matches a [sub]domain with has wildcard support.
*/}}
{{ define "/var/lib/haproxy/conf/os_wildcard_domain.map" -}}
{{     if isTrue (env "ROUTER_ALLOW_WILDCARD_ROUTES") -}}
{{       range $idx, $cfg := .State -}}
{{         if ne $cfg.Host "" -}}
{{           if $cfg.IsWildcard -}}
{{generateRouteRegexp $cfg.Host "" true}} 1
{{           end -}}
{{         end -}}
{{       end -}}
{{     end -}}{{/* end if router allows wildcard routes */}}
{{ end -}}{{/* end wildcard domain map template */}}

{{/*
    os_http_be.map: contains a mapping of www.example.com -> <service name>.  This map is used to discover the correct backend
                        by attaching a prefix (be_http:) by use_backend statements if acls are matched.
*/}}
{{ define "/var/lib/haproxy/conf/os_http_be.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (eq $cfg.TLSTermination "") -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} {{$idx}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end http host map template */}}

{{/*
    os_edge_http_be.map: same as os_http_be.map but allows us to separate tls from non-tls routes to ensure we don't expose
                            a tls only route on the unsecure port
*/}}
{{ define "/var/lib/haproxy/conf/os_edge_http_be.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (eq $cfg.TLSTermination "edge") -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} {{$idx}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end edge http host map template */}}

{{/*
    os_route_http_expose.map: contains a mapping of www.example.com -> <service name>.
    Map is used to also expose edge terminated and reencrypt routes via an insecure scheme
    (http) if acls match for routes with insecure option set to expose.
*/}}
{{ define "/var/lib/haproxy/conf/os_route_http_expose.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (and (matchValues (print $cfg.TLSTermination) "edge" "reencrypt") (eq $cfg.InsecureEdgeTerminationPolicy "Allow")) -}}
{{         if (eq $cfg.TLSTermination "edge") -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} be_edge_http:{{$idx}}
{{         else -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} be_secure:{{$idx}}
{{         end -}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end edge and reencrypt expose http host map template */}}

{{/*
    os_route_http_redirect.map: contains a mapping of www.example.com -> <service name>.
    Map is used to redirect insecure traffic to use a secure scheme (https)
    if acls match for routes that have the insecure option set to redirect.
*/}}
{{ define "/var/lib/haproxy/conf/os_route_http_redirect.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (eq $cfg.InsecureEdgeTerminationPolicy "Redirect") -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} {{$idx}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end redirect http host map template */}}


{{/*
    os_tcp_be.map: contains a mapping of www.example.com -> <service name>.  This map is used to discover the correct backend
                        by attaching a prefix (be_tcp: or be_secure:) by use_backend statements if acls are matched.
*/}}
{{ define "/var/lib/haproxy/conf/os_tcp_be.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (eq $cfg.Path "") (and (ne $cfg.Host "") (matchValues (print $cfg.TLSTermination) "passthrough" "reencrypt")) -}}
{{generateRouteRegexp $cfg.Host "" $cfg.IsWildcard}} {{$idx}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end tcp host map template */}}

{{/*
    os_sni_passthrough.map: contains a mapping of routes that expect to have an sni header and should be passed
    					through to the host_be.  Driven by the termination type of the ServiceAliasConfigs
*/}}
{{ define "/var/lib/haproxy/conf/os_sni_passthrough.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (eq $cfg.Path "") (eq $cfg.TLSTermination "passthrough") -}}
{{generateRouteRegexp $cfg.Host "" $cfg.IsWildcard}} 1
{{       end -}}
{{     end -}}
{{ end -}}{{/* end sni passthrough map template */}}


{{/*
    os_reencrypt.map: marker that the host is configured to use a secure backend, allows the selection of a backend
                    that does specific checks that avoid mitm attacks: http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#5.2-ssl
*/}}
{{ define "/var/lib/haproxy/conf/os_reencrypt.map" -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (eq $cfg.TLSTermination "reencrypt") -}}
{{generateRouteRegexp $cfg.Host $cfg.Path $cfg.IsWildcard}} {{$idx}}
{{       end -}}
{{     end -}}
{{ end -}}{{/* end reencrypt map template */}}

{{/*
    cert_config.map: contains a mapping of <cert-file> -> example.org
                     This map is used to present the appropriate cert
                     based on the sni header.
    Note: It is sort of a reverse map for our case but the order
          "<cert>: <domain-set>" is important as this allows us to use
         wildcards and/or use a deny set with !<domain> in the future.
*/}}
{{ define "/var/lib/haproxy/conf/cert_config.map" -}}
{{     $workingDir := .WorkingDir -}}
{{     range $idx, $cfg := .State -}}
{{       if and (ne $cfg.Host "") (matchValues (print $cfg.TLSTermination) "edge" "reencrypt") -}}
{{         $cert := index $cfg.Certificates $cfg.Host -}}
{{         if ne $cert.Contents "" -}}
{{$workingDir}}/certs/{{$idx}}.pem {{genCertificateHostName $cfg.Host $cfg.IsWildcard}}
{{         end -}}
{{      end -}}
{{     end -}}
{{ end }}{{/* end cert_config map template */}}

```


