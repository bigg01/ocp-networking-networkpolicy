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
 
 # on node B add OVS FLOW
 ovs-ofctl -O OpenFlow13 dump-flows br0 table=100|cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101
 table=100, priority=100,ip,reg0=0xdf6553 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x1df6552->pkt_mark,goto_table:101
 table=100, priority=0 actions=goto_table:101


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
tcpdump -i any -e -nn|grep "10.0.0.12"

```
