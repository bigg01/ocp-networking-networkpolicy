# 
Pods need to know internal service address to use correct external service.


https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/

<POD> --> SVC.NAMESPACE.EGRESS.svc.cluser.local-> database1.corp.net

Because fot SSL and other problems we need to use realname not internal service name.

We need to translate the SVC IP to exteranl IP.
146.0.0.2.3 database1.corp.net
172.168.0.2 database1.corp.net

```sh
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
...
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...


```

### Options
- user CoreDNS as split DNS. verry tricky because its must mit splitted per Namespace
- Host alis should work.
  - reading svc ip and update hosts file
  - write an operator?
  - move from http/https egress haproxy mode to plane TCP. supporting ranges
