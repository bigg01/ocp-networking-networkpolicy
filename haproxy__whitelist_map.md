# default map file

cat /etc/haproxy/acl-map
```txt
10.0.1.0/24
10.0.0.23
```

# add dynamic acl when haproxy is running
```sh
$ echo "show acl" |         sudo socat stdio /tmp/haproxy.sock
# id (file) description
0 (/etc/haproxy/acl-map-v12) pattern loaded from file '/etc/haproxy/acl-map-v12' used by acl at file '/etc/haproxy/haproxy6.cfg' line 41
1 (/etc/haproxy/acl-map-v12-route) pattern loaded from file '/etc/haproxy/acl-map-v12-route' used by acl at file '/etc/haproxy/haproxy6.cfg' line 41
2 () acl 'src' file '/etc/haproxy/haproxy6.cfg' line 41

 $  echo "show acl #0" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc677b90 10.0.0.4
0x55bedc677bc0 10.0.0.23

$ echo "show acl #1" |         sudo socat stdio /tmp/haproxy.sock

$ echo "add acl #1 10.0.0.208" |         sudo socat stdio /tmp/haproxy.sock

$  echo "show acl #1" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc658950 10.0.0.208

$ echo "show acl #0" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc677b90 10.0.0.4
0x55bedc677bc0 10.0.0.23

$ echo "del acl #0 10.0.0.208" |         sudo socat stdio /tmp/haproxy.sock
Key not found.

$ echo "show acl #0" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc677b90 10.0.0.4
0x55bedc677bc0 10.0.0.23

$ echo "show acl #1" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc658950 10.0.0.208

$ echo "show acl #0" |         sudo socat stdio /tmp/haproxy.sock
0x55bedc677b90 10.0.0.4
0x55bedc677bc0 10.0.0.23

$ echo "del acl #1 10.0.0.208" |         sudo socat stdio /tmp/haproxy.sock
$ echo "show acl #1" |         sudo socat stdio /tmp/haproxy.sock
```

cat haproxy6.cfg
```yaml
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
        option dontlognull
        #https://www.haproxy.com/de/blog/haproxy-log-customization/
        log-format "%ci:%cp\ [%t]\ %ft\ %b/%s\ %Tw/%Tc/%Tt\ %B\ %ts\  %ac/%fc/%bc/%sc/%rc\ %sq/%bq"

        # default acl for a SecureZone
        acl whitelist src -f /etc/haproxy/acl-map-v12 -f /etc/haproxy/acl-map-v12-roue

        #acl whitelist src map_ip(/etc/haproxy/acl-map,0.0.0.0.0)
        use_backend not_in_acl if !whitelist
        use_backend filter_app_http if whitelist


backend not_in_acl
        option log-separate-errors
        tcp-request content reject

backend filter_app_http
        server filter1 10.0.0.208:4444 check


# dynamic update
# https://www.haproxy.com/de/documentation/aloha/9-5/traffic-management/lb-layer7/lb-update/
```

## socklog
```sh
./socklog  inet 0 5555
listening on 0.0.0.0:5555, starting.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy stats started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy filter_http started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy not_in_acl started.
127.0.0.1: local2.notice: Jun 23 15:30:58 haproxy[29893]: Proxy filter_app_http started.
127.0.0.1: local2.info: Jun 23 15:31:01 haproxy[29893]: Client: 10.0.0.4:59410 10.0.0.208 [23/Jun/2018:15:31:01.582] filter_http filter_app_http/filter1 1/7/19 135 '--' ac/0/0/0/0 0/0]
```

# socat
```sh
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


https://github.com/bigg01/haproxycmd
```
$ go build -a -tags netgo -installsuffix netgo -o haproxycmd

./haproxycmd --socket /tmp/haproxy.sock
Unknown command. Please enter one of the following commands only :
  clear counters : clear max statistics counters (add 'all' for all counters)
  clear table    : remove an entry from a table
  help           : this message
  prompt         : toggle interactive mode with prompt
  quit           : disconnect
  show backend   : list backends in the current running config
  show info      : report information about the running process
  show pools     : report information about the memory pools usage
  show stat      : report counters for each proxy and server
  show stat resolvers [id]: dumps counters from all resolvers section and
                            associated name servers
  show errors    : report last request and response errors for each proxy
  show sess [id] : report the list of current sessions or dump this session
  show table [id]: report table usage stats or dump this table's contents
  show servers state [id]: dump volatile server information (for backend <id>)
  get weight     : report a server's current weight
  set weight     : change a server's weight
  set server     : change a server's state, weight or address
  set table [id] : update or create a table entry's data
  set timeout    : change a timeout setting
  set maxconn    : change a maxconn setting
  set rate-limit : change a rate limiting value
  disable        : put a server or frontend in maintenance mode
  enable         : re-enable a server or frontend which is in maintenance mode
  shutdown       : kill a session or a frontend (eg:to release listening ports)
  show acl [id]  : report available acls or dump an acl's contents
  get acl        : reports the patterns matching a sample for an ACL
  add acl        : add acl entry
  del acl        : delete acl entry
  clear acl <id> : clear the content of this acl
  show map [id]  : report available maps or dump a map's contents
  get map        : reports the keys and values matching a sample for a map
  set map        : modify map entry
  add map        : add map entry
  del map        : delete map entry
  clear map <id> : clear the content of this map
  set ssl <stmt> : set statement for ssl
```
