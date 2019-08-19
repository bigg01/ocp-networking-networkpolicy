# the idea is to run a coredns on openshift and add it to dnsmasq. while we are speaking to internal system we can set CNAMEs on SVC address to provide better NAT solution and Certs are matching.

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


##POD dns comes from host resolv
$ cat /etc/resolv.conf
nameserver 172.17.84.9
search splunktest.svc.cluster.local svc.cluster.local cluster.local reddog.microsoft.com
options ndots:5
$

##  /etc/resolv.conf comes from dnsmaqsk
