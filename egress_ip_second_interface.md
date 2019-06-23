### only works with 3.11

## take private range not existing in network
```
oc get netnamespace
NAME                    NETID      EGRESS IPS
dbtest                  8935821    []
default                 0          []
egress-test             12824096   []
egress-test2            14640467   []
egress-test3            3064846    []
egress-v2               9248244    []
gitea                   12235633   []
kube-public             10858048   []
kube-system             9880231    []
management-infra        5610426    []
minio                   6037138    []
mygotty                 8566492    []
openshift               14211522   []
openshift-console       16341554   []
openshift-infra         13958715   []
openshift-logging       1898091    []
openshift-node          5992634    []
openshift-sdn           6592413    []
openshift-web-console   8056251    []
openssl                 9681253    []
placeholder-egress      12824095   []
```
```
[root@ocpmaster01 guo]# oc get hostsubnet
NAME          HOST          HOST IP    SUBNET          EGRESS CIDRS   EGRESS IPS
ocpmaster01   ocpmaster01   10.0.0.3   10.128.0.0/23   []             []
ocprouter01   ocprouter01   10.0.0.2   10.130.0.0/23   []             []
ocprouter02   ocprouter02   10.0.0.7   10.129.0.0/23   []             []
```
# setup nodes DC1 and DC2 with the same range per VZone
```
```
# DC1
oc patch hostsubnet ocprouter02 --type=merge -p '{"egressCIDRs": ["172.0.2.0/24"]}'
# DC2
oc patch hostsubnet ocprouter01 --type=merge -p '{"egressCIDRs": ["172.0.2.0/24"]}'
# patch projects
## when node dies it will atomaticly failover to the DC2 node
oc patch netnamespace egress-test -p '{"egressIPs": ["172.0.2.12"]}'
oc patch netnamespace egress-test2 -p '{"egressIPs": ["172.0.2.13"]}'
```
```

```terminal
# when all Servers are up
oc get hostsubnet
NAME          HOST          HOST IP    SUBNET          EGRESS CIDRS     EGRESS IPS
ocpmaster01   ocpmaster01   10.0.0.3   10.128.0.0/23   []               []
ocprouter01   ocprouter01   10.0.0.2   10.130.0.0/23   [172.0.2.0/24]   [172.0.2.13]
ocprouter02   ocprouter02   10.0.0.7   10.129.0.0/23   [172.0.2.0/24]   [172.0.2.12]
# shutdown  ocprouter02
$  oc get hostsubnet
NAME          HOST          HOST IP    SUBNET          EGRESS CIDRS     EGRESS IPS
ocpmaster01   ocpmaster01   10.0.0.3   10.128.0.0/23   []               []
ocprouter01   ocprouter01   10.0.0.2   10.130.0.0/23   [172.0.2.0/24]   [172.0.2.13, 172.0.2.12]
ocprouter02   ocprouter02   10.0.0.7   10.129.0.0/23   [172.0.2.0/24]   []

$  oc get netnamespace
NAME                    NETID      EGRESS IPS
dbtest                  8935821    []
default                 0          []
egress-test             12824096   [10.0.1.12]
egress-test2            14640467   []
egress-test3            3064846    []
egress-v2               9248244    []
gitea                   12235633   []
kube-public             10858048   []
kube-system             9880231    []
management-infra        5610426    []
minio                   6037138    []
mygotty                 8566492    []
openshift               14211522   []
openshift-console       16341554   []
openshift-infra         13958715   []
openshift-logging       1898091    []
openshift-node          5992634    []
openshift-sdn           6592413    []
openshift-web-console   8056251    []
openssl                 9681253    []
placeholder-egress      12824095   []
```

#get vid hex
```
printf '%02X'  12824096
C3AE20
```

# APPD node (this time master)
# 0xc3ae20
```
 ovs-ofctl -O OpenFlow13 dump-flows br0 table=100
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=11999.551s, table=100, n_packets=0, n_bytes=0, priority=300,udp,tp_dst=4789 actions=drop
 cookie=0x0, duration=11999.551s, table=100, n_packets=0, n_bytes=0, priority=200,tcp,nw_dst=10.0.0.7,tp_dst=53 actions=output:2
 cookie=0x0, duration=11999.551s, table=100, n_packets=0, n_bytes=0, priority=200,udp,nw_dst=10.0.0.7,tp_dst=53 actions=output:2
 cookie=0x0, duration=45.599s, table=100, n_packets=0, n_bytes=0, priority=100,ip,reg0=0xc3ae20 actions=set_field:a2:c6:83:99:c2:f9->eth_dst,set_field:0xc3ae20->pkt_mark,goto_table:101
 cookie=0x0, duration=11999.551s, table=100, n_packets=0, n_bytes=0, priority=0 actions=goto_table:101
```
# egress node
-->  reg0=0xc3ae20 actions=set_field:a2:c6:83:99:c2:f9->eth_dst,set_field:0xc3ae20->pkt_mark,goto_table:101
```
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=12139.703s, table=100, n_packets=0, n_bytes=0, priority=300,udp,tp_dst=4789 actions=drop
 cookie=0x0, duration=12139.703s, table=100, n_packets=0, n_bytes=0, priority=200,tcp,nw_dst=10.0.0.7,tp_dst=53 actions=output:2
 cookie=0x0, duration=12139.703s, table=100, n_packets=0, n_bytes=0, priority=200,udp,nw_dst=10.0.0.7,tp_dst=53 actions=output:2
 cookie=0x0, duration=185.751s, table=100, n_packets=0, n_bytes=0, priority=100,ip,reg0=0xc3ae20 actions=set_field:a2:c6:83:99:c2:f9->eth_dst,set_field:0xc3ae20->pkt_mark,goto_table:101
 cookie=0x0, duration=12139.703s, table=100, n_packets=0, n_bytes=0, priority=0 actions=goto_table:101
```

# add mesquerade
```
# tunnel subnet of the cluster 10.128.0.0/14 
# it looks like its not needed
#iptables -t nat -I OPENSHIFT-MASQUERADE -s 10.128.0.0/14 -m mark --mark 0xc3ae20 -j SNAT --to-source 10.0.1.12
```

```
sh-4.2# ip a show tun0
10: tun0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether a2:c6:83:99:c2:f9 brd ff:ff:ff:ff:ff:ff
    inet 10.129.0.1/23 brd 10.129.1.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::a0c6:83ff:fe99:c2f9/64 scope link
       valid_lft forever preferred_lft forever
sh-4.2#
```
# delete iptables

```
 iptables -t nat -D OPENSHIFT-MASQUERADE 2
 ```
 
 #### interesting IPtable is not neede :-) if we need ports maybe we add here
```
watch -d "iptables -t nat  --list OPENSHIFT-MASQUERADE -n --line-numbers -v"
watch -d " ovs-ofctl dump-flows -O OpenFlow13 br0 table=100;ovs-ofctl dump-flows -O OpenFlow13 br0  table=101"
```


# apiVersion: network.openshift.io/v1
```yaml
kind: EgressNetworkPolicy
metadata:
  name: default
spec:
  egress:
    - to:
        cidrSelector: 10.0.1.0/24
      type: Allow
    - to:
        cidrSelector: 10.0.0.0/24
      type: Allow
    - to:
        cidrSelector: 0.0.0.0/0
      type: Deny
      ```


# helper
IPTABLES: https://github.com/bigg01/go-iptables
OVS: https://github.com/bigg01/ovs-exporter

Kubernetes watches IPS: 
```
oc get clusternetworks
NAME      CLUSTER NETWORKS   SERVICE NETWORK   PLUGIN NAME
default   10.128.0.0/14:9    172.30.0.0/16     redhat/openshift-ovs-networkpolicy
```
https://docs.openshift.com/container-platform/3.11/rest_api/apis-network.openshift.io/v1.EgressNetworkPolicy.html
https://docs.openshift.com/container-platform/3.11/rest_api/apis-network.openshift.io/v1.NetNamespace.html
