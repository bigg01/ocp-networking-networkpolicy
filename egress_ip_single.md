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

# scenario 1 x node 1x NS 
## assign IP to 1 NODE :white_check_mark:
```
oc patch hostsubnet ocprouter01 -p '{"egressIPs": ["10.0.0.11"]}'
# ---> not assigned to NODE
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'
# ---> ip assigned to node
ip a show ens18 |grep .11
    inet 10.0.0.11/24 brd 10.0.0.255 scope global secondary ens18
```


:boom:
