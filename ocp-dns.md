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

## golang client for coredns
