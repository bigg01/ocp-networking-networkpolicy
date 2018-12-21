# 

### we want to access TCP thru ingress OCP Router
 - option 1: Ingress CIDR External IP -- external loadbalancer
 
   can't work because external Ip works only on Node Level
   
 - option 2: build uniq ingress Router for TCP - port by annotation
 
   PoC
   
 - option 3: using nodeports on the Router nodes forwarded with a service
   could work
