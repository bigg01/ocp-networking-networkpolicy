he idea is to run a coredns on openshift and add it to dnsmasq. while we are speaking to internal system we can set CNAMEs on SVC address to provide better NAT solution and Certs are matching.

## install helm on OCP
```sh
export TILLER_NAMESPACE=mytiller

oc new-project $TILLER_NAMESPACE
oc process -f https://raw.githubusercontent.com/bigg01/helm-s2i-centos7/master/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.12.3 | oc create -f -

oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
```

## coredns helm chart
https://github.com/helm/charts/tree/master/stable/coredns
```sh
 helm install --name coredns --namespace=six-coredns stable/coredns
```

```sh
oc  run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

dnstools# dig www.example.org  @172.17.0.16 -p 9053

dig   @coredns-coredns.six-coredns.svc -p 1053 lampl.hugo
```

## golang client for coredns




# setup egress specific dns for PODS.

## rewrite setup
$ sudo ./coredns -conf corefile -dns.port 53
```
# corefile
# https://coredns.io/2017/05/08/custom-dns-entries-for-kubernetes/
.:53 {
    log
    reload 3s
    erratic
    debug
    errors
    health
    cache 30
    # import redirect for DNS
    import all-redirects
    # forward all requests to local DNS on OCP Node
    forward . 10.0.0.3:8053
    #file example.db ovhome.local
}
```

```
#all-redirects

rewrite stop {
        name regex (.*)\.oliverg.ch {1}.egress-test.svc.cluster.local
        answer name (.*)\.egress-test\.svc\.cluster\.local {1}.oliverg.ch
    }

rewrite stop {
        name regex (.*)\.oliverg.ch {1}.egress-test2.svc.cluster.local
        answer name (.*)\.egress-test2\.svc\.cluster\.local {1}.oliverg.ch
    }
```

## get nodename on pod
```
- name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

```
### POD specs

```yaml
    spec:
      containers:
        - command:
            - tail
        ..............
      dnsConfig:
        nameservers:
          - 10.0.0.208
        options:
          - name: ndots
            value: '2'
          - name: edns0
        searches:
          - oliverg.ch
      dnsPolicy: None
```
### POD specs



######POD dns comes from host resolv
```
dnstools# cat /etc/resolv.conf
nameserver 10.0.0.208
search oliverg.ch
options ndots:2 edns0
dnstools# dig httpd-example.oliverg.ch

; <<>> DiG 9.11.3 <<>> httpd-example.oliverg.ch
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19650
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 0335883fe91cbaea (echoed)
;; QUESTION SECTION:
;httpd-example.oliverg.ch.      IN      A

;; ANSWER SECTION:
httpd-example.oliverg.ch. 30    IN      A       172.30.35.55

;; Query time: 99 msec
;; SERVER: 10.0.0.208#53(10.0.0.208)
;; WHEN: Sun Aug 25 12:30:48 UTC 2019
;; MSG SIZE  rcvd: 105
```

# before 3.10/3.11
<aside class="error">

Reason: Pod "dnstools" is invalid: [spec.dnsConfig: Forbidden: DNSConfig: custom pod DNS is disabled by feature gate, spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` or `spec.tolerations` (only additions to existing tolerations)

</aside>








### rewrite setup
sudo ./coredns -conf corefile -dns.port 53


```
# coredns
# https://coredns.io/2017/05/08/custom-dns-entries-for-kubernetes/
#ovhome.local
.:53 {
    log
    reload 3s
    erratic
    debug
    errors
    health
    cache 30
    rewrite name oliverg.svc.cluster.local lamp.ovhome.local
    rewrite name hugo.svc.cluster.local oliverg.ch
    forward . 10.0.0.138
    file example.db ovhome.local
}


#file example.db ovhome.local
```

```
# example.db
$ORIGIN ovhome.local.
$TTL 86400
ovhome.local. IN SOA ns.ovhome.local. mail.ovhome.local. (
	33
	43200
	180
	1209600
	10800
)
ocprouter02.ovhome.local.	30	A	10.0.0.7
ocprouter01.ovhome.local.	3600	A	10.0.0.2
console.ocp.ovhome.local.	3600	A	10.0.0.2
ocpmaster01.ovhome.local.	3600	A	10.0.0.3
*.app.ocp.ovhome.local.	3600	A	10.0.0.2
modem.ovhome.local.	86400	A	10.0.0.138
lamp.ovhome.local.	86400	A	10.0.0.4
nas.ovhome.local.	86400	A	10.0.0.156
ovhome.local.	NS	ns.ovhome.local.
ns.ovhome.local.	A	10.0.0.156

```
### test 
```bash
dig @localhost -p 53 oliverg.svc.cluster.local  +noall +answer

; <<>> DiG 9.10.6 <<>> @localhost -p 53 oliverg.svc.cluster.local +noall +answer
; (2 servers found)
;; global options: +cmd
oliverg.svc.cluster.local. 18	IN	A	10.0.0.4
 guo  ~   (Detached)  $  ssh oliverg.svc.cluster.local
guo@oliverg.svc.cluster.local's password:

```



## start single coredns server / hostfile setup
$ sudo ./coredns -conf corefile -dns.port 53

corefile
```

.:53 {
    reload 3s
    erratic
    debug
    #log stdout
    log . stdout "{remote} {proto} Request: {name} {type} {when}" {
        class all
    }
    hosts example.hosts oliverg.net oliverg.ch six-group.net {
        10.0.0.1 example.org hugo.oliverg.net hugo.six-group.net
        10.0.0.4 oli.six-group.net
        fallthrough
        
    }

    forward . 10.0.0.138
}
```
