# 12.5 Сетевые решения CNI  

## Задание 1: установить в кластер CNI плагин Calico  

Для установки плагина Calico с помощью Kubespray, нужно установить следующий параметр в `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml` (это значение по умолчанию):  
```yaml
kube_network_plugin: calico
```
Calico интерфейсы видны в маршрутах:  
```bash
yc-user@cp1:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.99.1    0.0.0.0         UG    100    0        0 eth0
10.233.90.0     10.233.90.0     255.255.255.0   UG    0      0        0 vxlan.calico
10.233.96.0     10.233.96.0     255.255.255.0   UG    0      0        0 vxlan.calico
10.233.110.0    0.0.0.0         255.255.255.0   U     0      0        0 *
10.233.110.1    0.0.0.0         255.255.255.255 UH    0      0        0 cali5f52b1ba3ff
10.233.110.2    0.0.0.0         255.255.255.255 UH    0      0        0 cali3baf2139dac
192.168.99.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
192.168.99.1    0.0.0.0         255.255.255.255 UH    100    0        0 eth0
```
Создадим три пода frontend, backend, cache с помощью деплоймента, с сервисом на 80-м порту. И организуем доступ из frontend в backend, из backend в cash.
Остальные вариации должны быть закрыты. Проверим доступ перед тем как начнем.  
```bash
a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-dr25v - 10.233.96.2
a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 cache
Praqma Network MultiTool (with NGINX) - cache-b4f65b647-8df5f - 10.233.90.2
a.belyaev@MacBook-Pro devkub-netology % 
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 cache
Praqma Network MultiTool (with NGINX) - cache-b4f65b647-8df5f - 10.233.90.2
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-kjchs - 10.233.90.1
```
Доступы есть. Закроем все входящие соединения следующим манифестом:  
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```
И проверим:  
```bash
a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 backend                
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 cache                
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 cache                  
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 frontend               
command terminated with exit code 28
```
Доступ пропал. Откроем порт 80 на входящий трафик для backend из frontend с помощью манифеста:  
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```
И проверим:  
```bash
a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 backend              
Praqma Network MultiTool (with NGINX) - backend-f785447b9-dr25v - 10.233.96.2

a.belyaev@MacBook-Pro devkub-netology % kubectl exec frontend-8645d9cb9c-kjchs -- curl -s -m 1 cache                  
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 cache                  
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 frontend                 
command terminated with exit code 28
```
Доступ из frontend к backend появился. Далее по аналогии, откроем доступ для cache из backend [cache.yaml](manifests/network-policy/30-cache.yaml), и проверим результат:  
```bash
a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 cache                
Praqma Network MultiTool (with NGINX) - cache-b4f65b647-8df5f - 10.233.90.2

a.belyaev@MacBook-Pro devkub-netology % kubectl exec backend-f785447b9-dr25v -- curl -s -m 1 frontend                 
command terminated with exit code 28
a.belyaev@MacBook-Pro devkub-netology % kubectl exec cache-b4f65b647-8df5f -- curl -s -m 1 frontend
command terminated with exit code 28

```
Остальной доступ по прежнему закрыт.

## Задание 2: изучить, что запущено по умолчанию  

Получить список нод, ipPool и profile с помощью `calicoctl`  
```bash
$ calicoctl get node -o wide
NAME    ASN       IPV4               IPV6   
cp1     (64512)   192.168.99.34/24          
node1   (64512)   192.168.99.3/24           
node2   (64512)   192.168.99.33/24

$ calicoctl get ipPool -o wide
NAME           CIDR             NAT    IPIPMODE   VXLANMODE   DISABLED   DISABLEBGPEXPORT   SELECTOR   
default-pool   10.233.64.0/18   true   Never      Always      false      false              all()

$ calicoctl get profile -o wide
NAME                                                 LABELS                                                                                         
projectcalico-default-allow                                                                                                                         
kns.default                                          pcns.kubernetes.io/metadata.name=default,pcns.projectcalico.org/name=default                   
kns.kube-node-lease                                  pcns.kubernetes.io/metadata.name=kube-node-lease,pcns.projectcalico.org/name=kube-node-lease   
kns.kube-public                                      pcns.kubernetes.io/metadata.name=kube-public,pcns.projectcalico.org/name=kube-public           
kns.kube-system                                      pcns.kubernetes.io/metadata.name=kube-system,pcns.projectcalico.org/name=kube-system           
ksa.default.default                                  pcsa.projectcalico.org/name=default                                                            
ksa.kube-node-lease.default                          pcsa.projectcalico.org/name=default                                                            
ksa.kube-public.default                              pcsa.projectcalico.org/name=default                                                            
ksa.kube-system.attachdetach-controller              pcsa.projectcalico.org/name=attachdetach-controller                                            
ksa.kube-system.bootstrap-signer                     pcsa.projectcalico.org/name=bootstrap-signer                                                   
ksa.kube-system.calico-kube-controllers              pcsa.projectcalico.org/name=calico-kube-controllers                                            
ksa.kube-system.calico-node                          pcsa.projectcalico.org/name=calico-node                                                        
ksa.kube-system.certificate-controller               pcsa.projectcalico.org/name=certificate-controller                                             
ksa.kube-system.clusterrole-aggregation-controller   pcsa.projectcalico.org/name=clusterrole-aggregation-controller                                 
ksa.kube-system.coredns                              pcsa.addonmanager.kubernetes.io/mode=Reconcile,pcsa.projectcalico.org/name=coredns             
ksa.kube-system.cronjob-controller                   pcsa.projectcalico.org/name=cronjob-controller                                                 
ksa.kube-system.daemon-set-controller                pcsa.projectcalico.org/name=daemon-set-controller                                              
ksa.kube-system.default                              pcsa.projectcalico.org/name=default                                                            
ksa.kube-system.deployment-controller                pcsa.projectcalico.org/name=deployment-controller                                              
ksa.kube-system.disruption-controller                pcsa.projectcalico.org/name=disruption-controller                                              
ksa.kube-system.dns-autoscaler                       pcsa.addonmanager.kubernetes.io/mode=Reconcile,pcsa.projectcalico.org/name=dns-autoscaler      
ksa.kube-system.endpoint-controller                  pcsa.projectcalico.org/name=endpoint-controller                                                
ksa.kube-system.endpointslice-controller             pcsa.projectcalico.org/name=endpointslice-controller                                           
ksa.kube-system.endpointslicemirroring-controller    pcsa.projectcalico.org/name=endpointslicemirroring-controller                                  
ksa.kube-system.ephemeral-volume-controller          pcsa.projectcalico.org/name=ephemeral-volume-controller                                        
ksa.kube-system.expand-controller                    pcsa.projectcalico.org/name=expand-controller                                                  
ksa.kube-system.generic-garbage-collector            pcsa.projectcalico.org/name=generic-garbage-collector                                          
ksa.kube-system.horizontal-pod-autoscaler            pcsa.projectcalico.org/name=horizontal-pod-autoscaler                                          
ksa.kube-system.job-controller                       pcsa.projectcalico.org/name=job-controller                                                     
ksa.kube-system.kube-proxy                           pcsa.projectcalico.org/name=kube-proxy                                                         
ksa.kube-system.namespace-controller                 pcsa.projectcalico.org/name=namespace-controller                                               
ksa.kube-system.node-controller                      pcsa.projectcalico.org/name=node-controller                                                    
ksa.kube-system.nodelocaldns                         pcsa.addonmanager.kubernetes.io/mode=Reconcile,pcsa.projectcalico.org/name=nodelocaldns        
ksa.kube-system.persistent-volume-binder             pcsa.projectcalico.org/name=persistent-volume-binder                                           
ksa.kube-system.pod-garbage-collector                pcsa.projectcalico.org/name=pod-garbage-collector                                              
ksa.kube-system.pv-protection-controller             pcsa.projectcalico.org/name=pv-protection-controller                                           
ksa.kube-system.pvc-protection-controller            pcsa.projectcalico.org/name=pvc-protection-controller                                          
ksa.kube-system.replicaset-controller                pcsa.projectcalico.org/name=replicaset-controller                                              
ksa.kube-system.replication-controller               pcsa.projectcalico.org/name=replication-controller                                             
ksa.kube-system.resourcequota-controller             pcsa.projectcalico.org/name=resourcequota-controller                                           
ksa.kube-system.root-ca-cert-publisher               pcsa.projectcalico.org/name=root-ca-cert-publisher                                             
ksa.kube-system.service-account-controller           pcsa.projectcalico.org/name=service-account-controller                                         
ksa.kube-system.service-controller                   pcsa.projectcalico.org/name=service-controller                                                 
ksa.kube-system.statefulset-controller               pcsa.projectcalico.org/name=statefulset-controller                                             
ksa.kube-system.token-cleaner                        pcsa.projectcalico.org/name=token-cleaner                                                      
ksa.kube-system.ttl-after-finished-controller        pcsa.projectcalico.org/name=ttl-after-finished-controller                                      
ksa.kube-system.ttl-controller                       pcsa.projectcalico.org/name=ttl-controller
```
Посмотрим продобности про ноду `cp1`:  
```bash
$ calicoctl get node cp1 -o yaml
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"cp1","kubernetes.io/os":"linux","node-role.kubernetes.io/control-plane":"","node-role.kubernetes.io/master":"","node.kubernetes.io/exclude-from-external-load-balancers":""}'
  creationTimestamp: "2022-06-19T18:10:33Z"
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: cp1
    kubernetes.io/os: linux
    node-role.kubernetes.io/control-plane: ""
    node-role.kubernetes.io/master: ""
    node.kubernetes.io/exclude-from-external-load-balancers: ""
  name: cp1
  resourceVersion: "20731"
  uid: c599fe94-9854-4215-b5da-c45499efd4cb
spec:
  addresses:
  - address: 192.168.99.34/24
    type: CalicoNodeIP
  - address: 192.168.99.34
    type: InternalIP
  bgp:
    ipv4Address: 192.168.99.34/24
  ipv4VXLANTunnelAddr: 10.233.110.0
  orchRefs:
  - nodeName: cp1
    orchestrator: k8s
status:
  podCIDRs:
  - 10.233.64.0/24
```
