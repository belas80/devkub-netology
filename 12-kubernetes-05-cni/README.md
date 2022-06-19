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

