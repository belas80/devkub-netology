# 12.1 Компоненты Kubernetes  

## Задача 1: Установить Minikube  

Версия minikube:  
```bash
~ % minikube version
minikube version: v1.25.2
commit: 362d5fdc0a3dbee389b3d3f1034e8023e72bd3a7
```
Запуск:  
```bash
~ % minikube start --driver=docker
😄  minikube v1.25.2 на Darwin 12.4 (arm64)
✨  Используется драйвер docker на основе конфига пользователя
👍  Запускается control plane узел minikube в кластере minikube
🚜  Скачивается базовый образ ...
🔥  Creating docker container (CPUs=2, Memory=3885MB) ...
🐳  Подготавливается Kubernetes v1.23.3 на Docker 20.10.12 ...
    ▪ kubelet.housekeeping-interval=5m
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Компоненты Kubernetes проверяются ...
    ▪ Используется образ gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Включенные дополнения: storage-provisioner, default-storageclass
🏄  Готово! kubectl настроен для использования кластера "minikube" и "default" пространства имён по умолчанию
```
Статус:  
```bash
~ % minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
Запущенные служебные компоненты:  
```bash
~ % kubectl get pods --namespace=kube-system 
NAME                               READY   STATUS    RESTARTS        AGE
coredns-64897985d-6lx2v            1/1     Running   0               6m2s
etcd-minikube                      1/1     Running   0               6m15s
kube-apiserver-minikube            1/1     Running   0               6m17s
kube-controller-manager-minikube   1/1     Running   0               6m15s
kube-proxy-6669g                   1/1     Running   0               6m2s
kube-scheduler-minikube            1/1     Running   0               6m17s
storage-provisioner                1/1     Running   1 (5m32s ago)   6m14s
```

