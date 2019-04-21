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



tcpdump -i any -e -nn|grep "10.0.0.11"

ovs-ofctl -O OpenFlow13 dump-flows br0 table=100| cut -d',' -f3,6,7-
OFPST_FLOW reply (OF1.3) (xid=0x2):
 table=100, priority=300,udp,tp_dst=4789 actions=drop
 table=100, priority=200,tcp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=200,udp,nw_dst=10.0.0.2,tp_dst=53 actions=output:2
 table=100, priority=100,ip,reg0=0xdf6553 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x1df6552->pkt_mark,goto_table:101
 table=100, priority=100,ip,reg0=0x6f7936 actions=set_field:22:3b:a9:a4:8e:0c->eth_dst,set_field:0x6f7936->pkt_mark,goto_table:101
 table=100, priority=0 actions=goto_table:101



```
