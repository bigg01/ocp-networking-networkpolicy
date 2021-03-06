

https://github.com/openshift/origin/blob/c68d654128cc4ec776a183d20db1d24b51db07d5/pkg/network/node/iptables.go#L236

https://github.com/openshift/origin/blob/c68d654128cc4ec776a183d20db1d24b51db07d5/pkg/network/node/egressip.go#L171

https://github.com/openshift/origin/blob/release-3.9/pkg/network/node/ovscontroller.go#L714


# egress by hand
```console
oc new-project egress-test3
oc get netnamespace|grep egress-test3
egress-test3            3064846    []
```

`printf '%02X' 3064846 ; echo`
2EC40E

# tun0 interface
## NODE A
```console
root@ocpmaster01 origin]# ip a show|grep "tun" -A +2
10: tun0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 22:e9:5c:34:a4:83 brd ff:ff:ff:ff:ff:ff
    inet 10.128.0.1/23 brd 10.128.1.255 scope global tun0
       valid_lft forever preferred_lft forever
11: veth80959efb@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group defaultff
```

NODE_B_TUN_MAC="22:e9:5c:34:a4:83"

## NODE B
```sh
ip a show|grep "tun" -A +2
10: tun0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 22:3b:a9:a4:8e:0c brd ff:ff:ff:ff:ff:ff
    inet 10.130.0.1/23 brd 10.130.1.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::203b:a9ff:fea4:8e0c/64 scope link
```    
NODE_B_TUN_MAC="22:3b:a9:a4:8e:0c"

```bash
 docker exec -it k8s_openvswitch_ovs-xrtfv_openshift-sdn_d038855b-1ce4-11e9-be6e-9a2f895abae0_5 bash
 -d',' -f3,6,7-01 origin]# ovs-ofctl -O OpenFlow13 dump-flows br0 table=100| cut
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=100,ip,reg0=0xdf6553 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=0 actions=goto_table:101
 
 
 # add flow with vid from Node A to B
 ## vid = 0x2EC40E , node b 10.0.0.2
 ovs-ofctl add-flow br0 "table=100, priority=100,ip,reg0=0x2EC40E actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1" -O OpenFlow13
 
 
 -d',' -f3,6,7-01 origin]# ovs-ofctl -O OpenFlow13 dump-flows br0 table=100| cut
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=100,ip,reg0=0xdf6553 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=100,ip,reg0=0x2ec40e actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1 # added 
 table=100, priority=0 actions=goto_table:101
[root@ocpmaster01 origin]#
 
 # on node B add IP
 ip addr add 10.0.0.12/24  dev ens18
 ip link|grep ens -A+1
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 8a:03:f9:72:e3:07 brd ff:ff:ff:ff:ff:ff
    
    IP=10.0.0.12
    mac=8a:03:f9:72:e3:07
    
    
 
 # on node B add OVS FLOW
 NODE_B_TUN_MAC="22:3b:a9:a4:8e:0c"
 
 ## check flow
 ovs-ofctl -O OpenFlow13 dump-flows br0 table=100|cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101
 table=100, priority=100,ip,reg0=0xdf6553 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x1df6552->pkt_mark,goto_table:101
 table=100, priority=0 actions=goto_table:101

## add flow - VID to Tunnel Mac
ovs-ofctl add-flow br0 "table=100,priority=123,ip,reg0=0x2EC40E actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x2EC40E->pkt_mark,goto_table:101" -O OpenFlow13


# node B - # add vid to vid

ovs-ofctl -O OpenFlow13 dump-flows br0 table=100|cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101
 table=100, priority=100,ip,reg0=0xdf6553 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x1df6552->pkt_mark,goto_table:101
 table=100, priority=123,ip,reg0=0x2ec40e actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x2ec40e->pkt_mark,goto_table:101 # add vid to vid
 table=100, priority=0 actions=goto_table:101

# iptable create SNAT mark VID
iptables -t nat -I OPENSHIFT-MASQUERADE -s 10.128.0.0/14 -m mark --mark 0x2EC40E -j SNAT --to-source 10.0.0.12
[root@ocprouter01 origin]# iptables -t nat  --list OPENSHIFT-MASQUERADE -n --line-numbers
Chain OPENSHIFT-MASQUERADE (1 references)
num  target     prot opt source               destination
1    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x2ec40e to:10.0.0.12
2    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.11
3    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.12
4    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x1df6552 to:10.0.0.12
5    OPENSHIFT-MASQUERADE-2  all  --  10.128.0.0/14        0.0.0.0/0            /* masquerade pod-to-external traffic */


# go to pod and check curl external traffic 

# tcpdump on egress node

tcpdump -i any -e -nn|egrep "vni|10.0.0.12"
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
22:40:29.085961  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 126: 10.0.0.3.39053 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.086277 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 76: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [S], seq 966681133, win 28200, options [mss 1410,sackOK,TS val 2729734694 ecr 0,nop,wscale 7], length 0
22:40:29.090887  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 76: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [S.], seq 2425471942, ack 966681134, win 28960, options [mss 1460,sackOK,TS val 5348049 ecr 2729734694,nop,wscale 7], length 0
22:40:29.091007 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 126: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.091347  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.091384 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 1, win 221, options [nop,nop,TS val 2729734700 ecr 5348049], length 0
22:40:29.091444  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 195: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.091470 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 145: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [P.], seq 1:78, ack 1, win 221, options [nop,nop,TS val 2729734700 ecr 5348049], length 77: HTTP: GET / HTTP/1.1
22:40:29.098757  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 1:1399, ack 78, win 227, options [nop,nop,TS val 5348057 ecr 2729734700], length 1398: HTTP: HTTP/1.1 200 OK
22:40:29.098786 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.098986  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.099010 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 1399, win 243, options [nop,nop,TS val 2729734707 ecr 5348057], length 0
22:40:29.099030  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 1399:2797, ack 78, win 227, options [nop,nop,TS val 5348057 ecr 2729734700], length 1398: HTTP
22:40:29.099045 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.099105  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 2797:4195, ack 78, win 227, options [nop,nop,TS val 5348057 ecr 2729734700], length 1398: HTTP
22:40:29.099123 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.099159  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.099178 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 2797, win 264, options [nop,nop,TS val 2729734708 ecr 5348057], length 0
22:40:29.099221  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.099228  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 4195:5593, ack 78, win 227, options [nop,nop,TS val 5348057 ecr 2729734700], length 1398: HTTP
22:40:29.099243 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.099265 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 4195, win 286, options [nop,nop,TS val 2729734708 ecr 5348057], length 0
22:40:29.099348  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.099365 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 5593, win 308, options [nop,nop,TS val 2729734708 ecr 5348057], length 0
22:40:29.101403  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 5593:6991, ack 78, win 227, options [nop,nop,TS val 5348057 ecr 2729734700], length 1398: HTTP
22:40:29.101416 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.101544  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.101564 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 6991, win 330, options [nop,nop,TS val 2729734710 ecr 5348057], length 0
22:40:29.103344  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1466: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [.], seq 6991:8389, ack 78, win 227, options [nop,nop,TS val 5348058 ecr 2729734700], length 1398: HTTP
22:40:29.103357 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.103441  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 1589: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [P.], seq 8389:9910, ack 78, win 227, options [nop,nop,TS val 5348058 ecr 2729734700], length 1521: HTTP
22:40:29.103459 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 1516: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.103461 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 241: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.103468  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.103479 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 8389, win 352, options [nop,nop,TS val 2729734712 ecr 5348058], length 0
22:40:29.103559  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.103577 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 9910, win 376, options [nop,nop,TS val 2729734712 ecr 5348058], length 0
22:40:29.103853  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.103873 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [F.], seq 78, ack 9910, win 376, options [nop,nop,TS val 2729734712 ecr 5348058], length 0
22:40:29.108734  In 84:2b:2b:b1:7c:04 ethertype IPv4 (0x0800), length 68: 10.0.0.4.8080 > 10.0.0.12.39602: Flags [F.], seq 9910, ack 79, win 227, options [nop,nop,TS val 5348068 ecr 2729734712], length 0
22:40:29.108756 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 118: 10.0.0.2.35487 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:29.108948  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 118: 10.0.0.3.51006 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
22:40:29.108969 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 68: 10.0.0.12.39602 > 10.0.0.4.8080: Flags [.], ack 9911, win 376, options [nop,nop,TS val 2729734717 ecr 5348068], length 0
22:40:34.405965 Out 8a:03:f9:72:e3:07 ethertype IPv4 (0x0800), length 94: 10.0.0.2.45709 > 10.0.0.3.4789: VXLAN, flags [I] (0x08), vni 0
22:40:34.406611  In 9a:2f:89:5a:ba:e0 ethertype IPv4 (0x0800), length 94: 10.0.0.3.45931 > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 3064846
^C259 packets captured
343 packets received by filter

```

## works with second interface too
```sh
12: ens19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 2e:12:1d:8c:90:47 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.200/24 brd 10.0.0.255 scope global noprefixroute dynamic ens19
       valid_lft 86031sec preferred_lft 86031sec
    inet6 2a02:120b:c3d8:30a0:8031:5204:8258:224a/64 scope global noprefixroute dynamic
       valid_lft 86368sec preferred_lft 14368sec
    inet6 fe80::8f91:6286:5627:6be9/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
e-numbersrouter01 origin]# iptables -t nat  --list OPENSHIFT-MASQUERADE -n --line
Chain OPENSHIFT-MASQUERADE (1 references)
num  target     prot opt source               destination
1    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x2ec40e to:10.0.0.200
2    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.11
3    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.12
4    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x1df6552 to:10.0.0.12
5    OPENSHIFT-MASQUERADE-2  all  --  10.128.0.0/14        0.0.0.0/0            /* masquerade pod-to-external traffic */
[root@ocprouter01 origin]#
```


## 2 target with diffrent prios
`0x2ec40e -> priority=100 set_field:10.0.0.2`
`0x2ec40e -> priority=101 set_field:10.0.0.7`
```sh
# ovs-ofctl -O OpenFlow13 dump-flows br0 table=100|cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=100,ip,reg0=0x2ec40e actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.7->tun_dst,output:1
 table=100, priority=101,ip,reg0=0x2ec40e actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 table=100, priority=100,reg0=0xdf6553 actions=drop
 table=100, priority=100,reg0=0x2ec40e actions=drop
 table=100, priority=0 actions=goto_table:101
```

```console
ovs-ofctl -O OpenFlow13 show br0
OFPT_FEATURES_REPLY (OF1.3) (xid=0x2): dpid:0000669d59a55246
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS GROUP_STATS QUEUE_STATS
OFPST_PORT_DESC reply (OF1.3) (xid=0x3):
 1(vxlan0): addr:06:84:b7:b3:1c:f6
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 2(tun0): addr:d6:de:2d:d3:a9:70
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 3(veth70b8cc96): addr:82:49:a8:2b:dc:37
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 4(vethcd8eac3e): addr:a6:e0:aa:d2:a9:30
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 5(veth2519fc46): addr:1a:54:fe:95:a7:4c
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 6(veth765d118d): addr:ba:11:76:3f:b2:69
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 7(veth7ad73a21): addr:5e:65:6c:9f:b2:14
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 8(veth079af6c0): addr:1e:51:44:46:9b:8b
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 9(veth0c212d3e): addr:0e:c4:be:a1:9a:fe
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 10(veth9e0031e9): addr:f6:a2:9d:63:2e:ff
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 11(veth1b479575): addr:52:90:84:72:61:8b
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(br0): addr:66:9d:59:a5:52:46
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (OF1.3) (xid=0x5): frags=nx-match miss_send_len=0



ovs-vsctl show
4c610e4c-3dd2-4f08-ac46-956fe401f0f8
    Bridge "br0"
        fail_mode: secure
        Port "vethcd8eac3e"
            Interface "vethcd8eac3e"
        Port "veth2519fc46"
            Interface "veth2519fc46"
        Port "veth70b8cc96"
            Interface "veth70b8cc96"
        Port "veth9e0031e9"
            Interface "veth9e0031e9"
        Port "veth0c212d3e"
            Interface "veth0c212d3e"
        Port "vxlan0"
            Interface "vxlan0"
                type: vxlan
                options: {dst_port="4789", key=flow, remote_ip=flow}
        Port "veth765d118d"
            Interface "veth765d118d"
        Port "veth7ad73a21"
            Interface "veth7ad73a21"
        Port "veth079af6c0"
            Interface "veth079af6c0"
        Port "tun0"
            Interface "tun0"
                type: internal
        Port "veth1b479575"
            Interface "veth1b479575"
        Port "br0"
            Interface "br0"
                type: internal
    ovs_version: "2.7.0"
    
    
 ovs-ofctl -O OpenFlow13 dump-ports br0
OFPST_PORT reply (OF1.3) (xid=0x2): 12 ports
  port 11: rx pkts=2059, bytes=324426, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=2419, bytes=292455, drop=0, errs=0, coll=0
           duration=3394.562s
  port 10: rx pkts=4632, bytes=5610009, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=4813, bytes=571914, drop=0, errs=0, coll=0
           duration=3398.715s
  port  8: rx pkts=7373, bytes=26082708, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=7806, bytes=598614, drop=0, errs=0, coll=0
           duration=3404.297s
  port LOCAL: rx pkts=0, bytes=0, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=0, bytes=0, drop=0, errs=0, coll=0
           duration=3419.323s
  port  9: rx pkts=7480, bytes=26105746, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=7818, bytes=658812, drop=0, errs=0, coll=0
           duration=3404.009s
  port  5: rx pkts=3495, bytes=375831, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=4759, bytes=400533, drop=0, errs=0, coll=0
           duration=3418.688s
  port  1: rx pkts=56, bytes=63126, drop=?, errs=?, frame=?, over=?, crc=?
           tx pkts=106, bytes=24370, drop=?, errs=?, coll=?
           duration=3419.270s
  port  4: rx pkts=7360, bytes=26153148, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=7752, bytes=595224, drop=0, errs=0, coll=0
           duration=3418.842s
  port  6: rx pkts=18, bytes=1284, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=3, bytes=158, drop=0, errs=0, coll=0
           duration=3418.611s
  port  7: rx pkts=6284, bytes=2498120, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=8109, bytes=845490, drop=0, errs=0, coll=0
           duration=3418.518s
  port  2: rx pkts=37595, bytes=86345484, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=42549, bytes=3657606, drop=0, errs=0, coll=0
           duration=3419.243s
  port  3: rx pkts=243, bytes=12642, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=228, bytes=14696, drop=0, errs=0, coll=0
           duration=3418.921s
    
    
    
    
 ```
