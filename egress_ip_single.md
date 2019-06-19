

# 1 scenario 1xegressIP Node, 1xegressIP for Namespace :white_check_mark:
```bash
# assign IP to Node
oc patch hostsubnet ocprouter01 -p '{"egressIPs": ["10.0.0.11"]}'
# claim IP for NS
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'
```
## IP assinged to  Node 
```
ip a show ens18 |grep .11
    inet 10.0.0.11/24 brd 10.0.0.255 scope global secondary ens18
 ```
### test 
```
 curl -v 10.0.0.4:8080
* About to connect() to 10.0.0.4 port 8080 (#0)
*   Trying 10.0.0.4...
* Connected to 10.0.0.4 (10.0.0.4) port 8080 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.0.4:8080
> Accept: */*
>
< HTTP/1.1 200 OK
```
# 2 scenario 1xegressIP Node, same egressIP for 2 Namespaces :boom:
```bash
oc patch netnamespace egress-test2 -p '{"egressIPs": ["10.0.0.11"]}'
```
## LOG
```
LOG egressip.go:356] Multiple namespaces (7305526, 14640467) claiming EgressIP 10.0.0.11
```
## IP remove from the Node 
```
ip a show ens18 |grep .1 :heavy_exclamation_mark:
```
### test
```
 curl -v 10.0.0.4:8080
* About to connect() to 10.0.0.4 port 8080 (#0)
*   Trying 10.0.0.4...
```

# 3 scenario same egressIP on 2 Nodes, 1xegressIP for 1 Namespace :boom:
```
oc patch hostsubnet ocprouter01 -p '{"egressIPs": ["10.0.0.11"]}'
oc patch hostsubnet ocprouter02 -p '{"egressIPs": ["10.0.0.11"]}'
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'
```
```
oc get netnamespace |grep egress
egress-test             7305526    [10.0.0.11]
egress-test2            14640467   []
egress-test3            3064846    []
egress-v2               9248244    []
```
```
[root@ocpmaster01 guo]# oc get hostsubnet |grep .11
ocprouter01   ocprouter01   10.0.0.2   10.130.0.0/23   []             [10.0.0.11]
ocprouter02   ocprouter02   10.0.0.7   10.129.0.0/23   []             [10.0.0.11]
```
## LOG
```
egressip.go:356] Multiple nodes (10.0.0.2, 10.0.0.7) claiming EgressIP 10.0.0.11
```

## IP remove from the Node 
```
ip a show ens18 |grep .1 :heavy_exclamation_mark:
```



### clean 
```
oc patch hostsubnet ocpmaster01 -p '{"egressIPs": []}'
oc patch hostsubnet ocprouter01 -p '{"egressIPs": []}'
oc patch hostsubnet ocprouter02 -p '{"egressIPs": []}'
```

```
for n in egress-test egress-test2 egress-test3 egress-v2
do 
oc patch netnamespace $n -p '{"egressIPs": []}'
done
```
```
oc patch netnamespace egress-test2 -p '{"egressIPs": ["10.0.0.13"]}'
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'
```
```
oc get netnamespace |grep -i egress
NAME                    NETID      EGRESS IPS
egress-test             7305526    []
egress-test2            14640467   []
egress-test3            3064846    []
egress-v2               9248244    []
```
