https://github.com/openshift/origin/blob/2b41d8b0a674ffec10cf7d97e2e635803ca07351/pkg/network/node/node.go#L215

```golang
// Set node IP if required
func (c *OsdnNodeConfig) setNodeIP() error {
	if len(c.Hostname) == 0 {
		output, err := kexec.New().Command("uname", "-n").CombinedOutput()
		if err != nil {
			return err
		}
		c.Hostname = strings.TrimSpace(string(output))
		glog.Infof("Resolved hostname to %q", c.Hostname)
	}

	if len(c.SelfIP) == 0 {
		var err error
		c.SelfIP, err = netutils.GetNodeIP(c.Hostname)
		if err != nil {
			glog.V(5).Infof("Failed to determine node address from hostname %s; using default interface (%v)", c.Hostname, err)
			var defaultIP net.IP
			defaultIP, err = kubeutilnet.ChooseHostInterface()
			if err != nil {
				return err
			}
			c.SelfIP = defaultIP.String()
			glog.Infof("Resolved IP address to %q", c.SelfIP)
		}
	}

	if _, _, err := GetLinkDetails(c.SelfIP); err != nil {
		if err == ErrorNetworkInterfaceNotFound {
			err = fmt.Errorf("node IP %q is not a local/private address (hostname %q)", c.SelfIP, c.Hostname)
		}
		utilruntime.HandleError(fmt.Errorf("Unable to find network interface for node IP; some features will not work! (%v)", err))
	}

	return nil
}
```
```golang
//https://github.com/openshift/origin/blob/edddaaf84d87f9667d6833a328e262a5fb0d6c34/pkg/network/node/egressip.go#L151

func (eip *egressIPWatcher) assignEgressIP(egressIP, mark string) error {
	if egressIP == eip.localIP {
		return fmt.Errorf("desired egress IP %q is the node IP", egressIP)
	}

	if eip.testModeChan != nil {
		eip.testModeChan <- fmt.Sprintf("claim %s", egressIP)
		return nil
	}

	localEgressIPMaskLen, _ := eip.localEgressNet.Mask.Size()
	egressIPNet := fmt.Sprintf("%s/%d", egressIP, localEgressIPMaskLen)
	addr, err := netlink.ParseAddr(egressIPNet)
	if err != nil {
		return fmt.Errorf("could not parse egress IP %q: %v", egressIPNet, err)
	}
	if !eip.localEgressNet.Contains(addr.IP) {
		return fmt.Errorf("egress IP %q is not in local network %s of interface %s", egressIP, eip.localEgressNet.String(), eip.localEgressLink.Attrs().Name)
	}
	err = netlink.AddrAdd(eip.localEgressLink, addr)
	if err != nil {
		if err == syscall.EEXIST {
			glog.V(2).Infof("Egress IP %q already exists on %s", egressIPNet, eip.localEgressLink.Attrs().Name)
		} else {
			return fmt.Errorf("could not add egress IP %q to %s: %v", egressIPNet, eip.localEgressLink.Attrs().Name, err)
		}
	}
	// Use arping to try to update other hosts ARP caches, in case this IP was
	// previously active on another node. (Based on code from "ifup".)
	go func() {
		out, err := exec.Command("/sbin/arping", "-q", "-A", "-c", "1", "-I", eip.localEgressLink.Attrs().Name, egressIP).CombinedOutput()
		if err != nil {
			glog.Warningf("Failed to send ARP claim for egress IP %q: %v (%s)", egressIP, err, string(out))
			return
		}
		time.Sleep(2 * time.Second)
		_ = exec.Command("/sbin/arping", "-q", "-U", "-c", "1", "-I", eip.localEgressLink.Attrs().Name, egressIP).Run()
	}()

	if err := eip.iptables.AddEgressIPRules(egressIP, mark); err != nil {
		return fmt.Errorf("could not add egress IP iptables rule: %v", err)
	}

	return nil
}

```

eip.localEgressLink


localEgressLink netlink.Link

https://github.com/vishvananda/netlink



https://github.com/vishvananda/netlink/blob/028453c77ce572d3554b3873c654663283ac42a3/addr_linux.go#L17
```golang

// AddrAdd will add an IP address to a link device.
// Equivalent to: `ip addr add $addr dev $link`
func AddrAdd(link Link, addr *Addr) error {
	return pkgHandle.AddrAdd(link, addr)
}
```
```golang
package main

import (
    "github.com/vishvananda/netlink"
)

func main() {
    lo, _ := netlink.LinkByName("lo")
    addr, _ := netlink.ParseAddr("169.254.169.254/32")
    netlink.AddrAdd(lo, addr)
}
```



```sh
# hacking egress route
oc patch hostsubnet ocprouter01 -p '{"egressIPs": ["10.0.0.11","10.0.0.13"]}'
oc patch netnamespace egress-test2 -p '{"egressIPs": ["10.0.0.13"]}'
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'


c get netnamespace
NAME                    NETID      EGRESS IPS
dbtest                  8935821    []
default                 0          []
egress-test             7305526    [10.0.0.11]
egress-test2            14640467   [10.0.0.13]
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


# node A
 docker exec -it k8s_openvswitch_ovs-xrtfv_openshift-sdn_d038855b-1ce4-11e9-be6e-9a2f895abae0_5 bash
[root@ocpmaster01 origin]# ovs-ofctl -O OpenFlow13 dump-flows br0 table=100 --color
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=8201.224s, table=100, n_packets=0, n_bytes=0, priority=300,udp,tp_dst=4789 actions=drop
 cookie=0x0, duration=8201.224s, table=100, n_packets=0, n_bytes=0, priority=200,tcp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 cookie=0x0, duration=8201.224s, table=100, n_packets=468, n_bytes=43458, priority=200,udp,nw_dst=10.0.0.3,tp_dst=53 actions=output:2
 cookie=0x0, duration=505.625s, table=100, n_packets=0, n_bytes=0, priority=100,ip,reg0=0xdf6553 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 cookie=0x0, duration=96.343s, table=100, n_packets=0, n_bytes=0, priority=100,ip,reg0=0x6f7936 actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:10.0.0.2->tun_dst,output:1
 cookie=0x0, duration=8201.224s, table=100, n_packets=18065, n_bytes=1789972, priority=0 actions=goto_table:101

# get vid
echo $((0xdf6553))
14640467 = egress-test2

 echo $((0x6f7936))
7305526 = egress-test


# node B Egress

# convert hex to ip
printf '%02X' 10 0 0 11 ; echo
0A00000B
[root@ocprouter01 origin]# printf '%d.%d.%d.%d\n' `echo 0A00000B | sed -r 's/(..)/0x\1 /g'`
10.0.0.11

tcpdump -i any -e -nn|grep "10.0.0.11"

ovs-ofctl -O OpenFlow13 dump-flows br0 table=100| cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0xdf6553 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x1df6552->pkt_mark,goto_table:101
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101
 table=100, priority=0 actions=goto_table:101
 
 
 ovs-ofctl add-flow br0 "table=100,priority=123,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101" -O OpenFlow13

ovs-ofctl del-flows br0 "table=100,ip,reg0=0x6f7936" -O OpenFlow13






 ip addr add 10.0.0.12/24  dev ens18
[root@ocprouter01 origin]#  /sbin/arping -q -A -c 1 -I  ens18 10.0.0.12
[root@ocprouter01 origin]# ip neigh
10.0.0.12 dev ens18  FAILED
10.128.1.224 dev tun0 lladdr 0a:58:0a:80:01:e0 STALE
10.128.1.206 dev tun0 lladdr 0a:58:0a:80:01:ce STALE
10.0.0.202 dev ens18 lladdr 38:c9:86:28:94:89 REACHABLE
10.0.0.13 dev ens18  FAILED
10.0.0.3 dev ens18 lladdr 9a:2f:89:5a:ba:e0 REACHABLE
10.128.1.226 dev tun0 lladdr 0a:58:0a:80:01:e2 STALE
10.0.0.155 dev ens18  FAILED
10.0.0.1 dev ens18  FAILED
10.128.0.1 dev tun0 lladdr 22:e9:5c:34:a4:83 STALE
10.128.1.222 dev tun0 lladdr 0a:58:0a:80:01:de STALE
10.0.0.4 dev ens18 lladdr 84:2b:2b:b1:7c:04 STALE
10.130.0.123 dev tun0 lladdr 0a:58:0a:82:00:7b STALE
10.0.0.138 dev ens18 lladdr 7c:b7:33:03:48:3f STALE
10.0.0.156 dev ens18 lladdr 00:11:32:4e:6f:09 STALE
2a02:120b:c3d8:30a0:7eb7:33ff:fe03:483f dev ens18 lladdr 7c:b7:33:03:48:3f router STALE
fe80::7eb7:33ff:fe03:483f dev ens18 lladdr 7c:b7:33:03:48:3f router STALE
[root@ocprouter01 origin]#  ip addr del 10.0.0.12/24  dev ens18
[root@ocprouter01 origin]#  /sbin/arping -q -A -c 1 -I  ens18 10.0.0.12
bind: Cannot assign requested address
[root@ocprouter01 origin]#



ovs-ofctl -O OpenFlow13 dump-flows br0 | cut -d',' -f3|grep table| uniq
 table=0
 table=10
 table=20
 table=21
 table=25
 table=30
 table=40
 table=50
 table=60
 table=70
 table=80
 table=90
 table=100
 table=101
 table=110
 table=111
 table=120
 table=253


# vid maps over vxlan to vid and iptable makes SNAT
iptables -t nat --list -n|grep '10.0.0.11'
SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.11
[root@ocprouter01 origin]# iptables -t nat --list -n|grep '10.0.0.12'
SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x1df6552 to:10.0.0.12
[root@ocprouter01 origin]# iptables -t nat --list -n|grep '0x6f7936'
SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.11
[root@ocprouter01 origin]# iptables -t nat --list -n|grep '10.0.0.11'^C
[root@ocprouter01 origin]# ovs-ofctl -O OpenFlow13 dump-flows br0 table=100|cut -d',' -f3,6,7- |grep 0x6f7936
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101




 iptables -t nat -I OPENSHIFT-MASQUERADE -s 10.128.0.0/14 -m mark --mark 0x6f7936 -j SNAT --to-source 10.0.0.12
[root@ocprouter01 origin]# iptables -t nat  --list OPENSHIFT-MASQUERADE -n --line-numbers
Chain OPENSHIFT-MASQUERADE (1 references)
num  target     prot opt source               destination
1    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.12
2    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x1df6552 to:10.0.0.12
3    SNAT       all  --  10.128.0.0/14        0.0.0.0/0            mark match 0x6f7936 to:10.0.0.11
4    OPENSHIFT-MASQUERADE-2  all  --  10.128.0.0/14        0.0.0.0/0            /* masquerade pod-to-external traffic */
[root@ocprouter01 origin]# iptables -t nat -D OPENSHIFT-MASQUERADE  4


```


#egress by hand
```sh
oc new-project egress-test3
oc get netnamespace|grep egress-test3
egress-test3            3064846    []


printf '%02X' 3064846 ; echo
2EC40E

# tun0 interface
## NODE A
root@ocpmaster01 origin]# ip a show|grep "tun" -A +2
10: tun0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 22:e9:5c:34:a4:83 brd ff:ff:ff:ff:ff:ff
    inet 10.128.0.1/23 brd 10.128.1.255 scope global tun0
       valid_lft forever preferred_lft forever
11: veth80959efb@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group defaultff

NODE_B_TUN_MAC="22:e9:5c:34:a4:83"

## NODE B
ip a show|grep "tun" -A +2
10: tun0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 22:3b:a9:a4:8e:0c brd ff:ff:ff:ff:ff:ff
    inet 10.130.0.1/23 brd 10.130.1.255 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::203b:a9ff:fea4:8e0c/64 scope link
    
NODE_B_TUN_MAC="22:3b:a9:a4:8e:0c"


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
