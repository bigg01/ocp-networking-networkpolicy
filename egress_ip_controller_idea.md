https://github.com/openshift/origin/blob/release-3.9/pkg/network/node/ovscontroller.go

run a deamonset byside of SDN Deamonset and add static Egress routes per VID and Projects.

Maybe over CR or watch filter. Egress Failover can be done over This controller.

-  App nodes OVS customer controller - VID tun target Egress Server
- Egress Nodes OVS customer controller - VID mapping and Iptables

https://github.com/openshift/origin/blob/release-3.11/pkg/network/node/ovscontroller.go

```go
func (oc *ovsController) ensureTunMAC() error {
	if oc.tunMAC != "" {
		return nil
	}

	val, err := oc.ovs.Get("Interface", Tun0, "mac_in_use")
	if err != nil {
		return fmt.Errorf("could not get %s MAC address: %v", Tun0, err)
	} else if len(val) != 19 || val[0] != '"' || val[18] != '"' {
		return fmt.Errorf("bad MAC address for %s: %q", Tun0, val)
	}
	oc.tunMAC = val[1:18]
	return nil
}

```


```go
func (oc *ovsController) SetNamespaceEgressViaEgressIP(vnid uint32, nodeIP, mark string) error {
	otx := oc.ovs.NewTransaction()
	otx.DeleteFlows("table=100, reg0=%d", vnid)
	if nodeIP == "" {
		// Namespace wants egress IP, but no node hosts it, so drop
		otx.AddFlow("table=100, priority=100, reg0=%d, actions=drop", vnid)
	} else if nodeIP == oc.localIP {
		if err := oc.ensureTunMAC(); err != nil {
			return err
		}
		// tkggo we add for any VID egress node
    otx.AddFlow("table=100, priority=100, reg0=%d, ip, actions=set_field:%s->eth_dst,set_field:%s->pkt_mark,goto_table:101", vnid, oc.tunMAC, mark)
    
	} else {
		otx.AddFlow("table=100, priority=100, reg0=%d, ip, actions=move:NXM_NX_REG0[]->NXM_NX_TUN_ID[0..31],set_field:%s->tun_dst,output:1", vnid, nodeIP)
	}
	return otx.Commit()
}


```
