# Next Gen Egress

We must order a Mobiltiy Range Subnet for each Netzone. We will assign on a DC1 and DC2 Egress nodes the mobility Range.
Failover will happen if egress node disapear automaticly. Egress IP will be assigned randomly. Egress NetworkPolicy will fillter egress Traffic.  Looks pritty cool! probelm Egress Policy can only be filtered with IPs and not with Pors.

|Zone |Range |EgressNode DC1 |EgressNode DC2   |   |   |
|---|---|---|---|---|---|
|  VN1 | 10.0.1.0/23  | {"egressCIDRs": ["10.0.1.0/23"]}  |  {"egressCIDRs": ["10.0.1.0/23"]} | failover happen when Node goes down  |   |
|  VN2 | 10.0.2.0/23  | {"egressCIDRs": ["10.0.2.0/23"]}  |  {"egressCIDRs": ["10.0.2.0/23"]} | failover happen when Node goes down  |   |
|---|---|---|---|---|
|   |   |   |   |   |
|   |   |   |   |   |

```
oc patch hostsubnet ocprouter01 --type=merge -p '{"egressCIDRs": ["10.0.0.0/24"]}'
oc patch hostsubnet ocprouter02 --type=merge -p '{"egressCIDRs": ["10.0.0.0/24"]}'
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.11"]}'
oc patch netnamespace egress-test -p '{"egressIPs": ["10.0.0.12"]}'

oc get netnamespace
egress-test             12824096   [10.0.0.12]
placeholder-egress      12824095   [10.0.0.11]

NAME          HOST          HOST IP    SUBNET          EGRESS CIDRS    EGRESS IPS
ocpmaster01   ocpmaster01   10.0.0.3   10.128.0.0/23   []              []
ocprouter01   ocprouter01   10.0.0.2   10.130.0.0/23   [10.0.0.0/24]   [10.0.0.11]
ocprouter02   ocprouter02   10.0.0.7   10.129.0.0/23   [10.0.0.0/24]   [10.0.0.12]
```
