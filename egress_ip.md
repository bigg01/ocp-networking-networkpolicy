https://github.com/openshift/origin/blob/2b41d8b0a674ffec10cf7d97e2e635803ca07351/pkg/network/node/node.go#L215

```
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

