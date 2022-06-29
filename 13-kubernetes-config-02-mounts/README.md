# 13.2 разделы и монтирование  

## Задание 1: подключить для тестового конфига общую папку  

Модифицируем манифест из предыдущего задания сделющим образом, добавив `volume`:  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: stage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - image: belas80/frontend
          imagePullPolicy: IfNotPresent
          name: frontend
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: "/static-f"
              name: my-vol
        - image: belas80/backend
          imagePullPolicy: IfNotPresent
          name: backend
          ports:
            - containerPort: 9000
          volumeMounts:
            - mountPath: "/static-b"
              name: my-vol
      volumes:
        - name: my-vol
          emptyDir: {}
      terminationGracePeriodSeconds: 30
```
Попробуем что нибудь записать в эту папку из контейнера `frontend` и прочитать это из `backend`:  
```shell
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c frontend -i -t -- ls /     
app   dev                   etc   lib64  opt   run   static-f  usr
bin   docker-entrypoint.d   home  media  proc  sbin  sys       var
boot  docker-entrypoint.sh  lib   mnt    root  srv   tmp
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c frontend -i -t -- ls -l /static-f
total 0
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c frontend -i -t -- sh -c 'echo "Hello from frontend" > /static-f/test.file'
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c frontend -i -t -- ls -l /static-f                                 
total 4
-rw-r--r-- 1 root root 20 Jun 29 17:53 test.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c backend -i -t -- ls -l /static-b 
total 4
-rw-r--r-- 1 root root 20 Jun 29 17:53 test.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c backend -i -t -- cat /static-b/test.file
Hello from frontend
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c backend -i -t -- rm /static-b/test.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec pods/frontend-74b45bb56f-xn4jf -c frontend -i -t -- ls -l /static-f       
total 0
```
Попробуем найти этот файл на ноде:  
```shell
# Определим ноду
a.belyaev@MacBook-Pro-2 manifests % kubectl get po frontend-74b45bb56f-xn4jf -o yaml | grep nodeName
  nodeName: node1

# Посмотрим на uid пода
a.belyaev@MacBook-Pro-2 manifests % kubectl get po frontend-74b45bb56f-xn4jf -o yaml | grep uid                     
    uid: 0edfda7d-5b0a-4ee8-9c28-400c77b18659
  uid: 583fb4c5-44b0-4002-9b99-6a1df7afa3e6

# Зайдем на ноду и поищем файл
yc-user@node1:~$ sudo find /var/lib/kubelet -name test.file
/var/lib/kubelet/pods/583fb4c5-44b0-4002-9b99-6a1df7afa3e6/volumes/kubernetes.io~empty-dir/my-vol/test.file
yc-user@node1:~$ sudo find /var/lib/kubelet -name test.file | sudo xargs cat
Hello from frontend

# Удалим под и проверим папку на ноде
a.belyaev@MacBook-Pro-2 manifests % kubectl delete po frontend-74b45bb56f-xn4jf 
pod "frontend-74b45bb56f-xn4jf" deleted

yc-user@node1:~$ sudo ls -l /var/lib/kubelet/pods/583fb4c5-44b0-4002-9b99-6a1df7afa3e6/volumes/kubernetes.io~empty-dir/my-vol/
ls: cannot access '/var/lib/kubelet/pods/583fb4c5-44b0-4002-9b99-6a1df7afa3e6/volumes/kubernetes.io~empty-dir/my-vol/': No such file or directory
yc-user@node1:~$ sudo ls -l /var/lib/kubelet/pods/
total 24
drwxr-x--- 5 root root 4096 Jun 24 18:01 24d17145-1d69-42d1-bf2d-ff5627ed7619
drwxr-x--- 5 root root 4096 Jun 24 18:03 3eeb5b98-8055-4f1e-a307-ae4223546b15
drwxr-x--- 5 root root 4096 Jun 26 13:57 5e2a6a5b-a5de-438e-b26d-422d9e6377db
drwxr-x--- 5 root root 4096 Jun 24 18:01 a881161b-5fc3-468b-97b6-7eec05fc57c0
drwxr-x--- 5 root root 4096 Jun 24 18:04 d672063c-38b9-4b47-b61e-9b938871eceb
drwxr-x--- 5 root root 4096 Jun 24 18:00 ecb9b6d73ad5d6e8e64bb19aabaa138f

# Нет пода вместе с volume :)
```

## Задание 2: подключить общую папку для прода  

Поработав на stage, доработки нужно отправить на прод. В продуктиве у нас контейнеры крутятся в разных подах, поэтому потребуется PV и связь через PVC. Сам PV должен быть связан с NFS сервером.  

С помощью helm установим NFS сервер:  
```shell
helm install nfs-server stable/nfs-server-provisioner
```
Появился `StorageClass`:  
```shell
a.belyaev@MacBook-Pro-2 manifests % kubectl get sc    
NAME   PROVISIONER                                       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs    cluster.local/nfs-server-nfs-server-provisioner   Delete          Immediate           true                   68m
```
Так же предварительно установим на рабочие ноды пакет `nfs-common`  
Создадим манифест [`pvc.yaml`](manifests/pvc.yaml) для `PersistentVolumeClaim`:  
```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-volume-claim
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
```
Применим его и проверим что создалось:  
```shell
a.belyaev@MacBook-Pro-2 manifests % kubectl apply -f pvc.yaml 
persistentvolumeclaim/my-volume-claim created

a.belyaev@MacBook-Pro-2 manifests % kubectl get pvc,pv       
NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-volume-claim   Bound    pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac   100Mi      RWX            nfs            12s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
persistentvolume/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac   100Mi      RWX            Delete           Bound    prod/my-volume-claim   nfs                     12s
```
Создались `persistentvolumeclaim` и `persistentvolume`.  
Добавим `volume` и точку монтирования в деплоймент нашего [`frontend`](manifests/prod/frontend.yaml):  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: frontend
  name: frontend
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - image: belas80/frontend
          imagePullPolicy: IfNotPresent
          name: frontend
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: "/static"
              name: my-vol
      volumes:
        - name: my-vol
          persistentVolumeClaim:
            claimName: my-volume-claim
      terminationGracePeriodSeconds: 30
```
Аналогично для [`backend`](manifests/prod/backend.yaml).  
Применим и проверим что получилось:  
```shell
a.belyaev@MacBook-Pro-2 manifests % kubectl apply -f prod 
deployment.apps/backend created
service/backend created
statefulset.apps/db created
service/db created
deployment.apps/frontend created
service/frontend created

a.belyaev@MacBook-Pro-2 manifests % kubectl get po -o wide 
NAME                                  READY   STATUS    RESTARTS       AGE    IP             NODE    NOMINATED NODE   READINESS GATES
backend-67d575c954-r9lfh              1/1     Running   0              41s    10.233.90.49   node1   <none>           <none>
db-0                                  1/1     Running   0              41s    10.233.96.54   node2   <none>           <none>
frontend-79d8b7bf95-pp4z2             1/1     Running   0              41s    10.233.90.50   node1   <none>           <none>
multitool-55974d5464-cgtvf            1/1     Running   1 (146m ago)   3d5h   10.233.90.45   node1   <none>           <none>
nfs-server-nfs-server-provisioner-0   1/1     Running   0              78m    10.233.96.51   node2   <none>           <none>
```
Поды запустились, проверяем нашу шару:  
```shell
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/frontend-79d8b7bf95-pp4z2 -i -t -- ls -l /static
total 0
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/backend-67d575c954-r9lfh -i -t -- ls -l /static 
total 0
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/frontend-79d8b7bf95-pp4z2 -i -t -- sh -c 'echo "My test NFS" > /static/frontend.file'
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/backend-67d575c954-r9lfh -i -t -- ls -l /static                                      
total 4
-rw-r--r-- 1 root root 12 Jun 29 19:59 frontend.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/backend-67d575c954-r9lfh -i -t -- cat /static/frontend.file
My test NFS

# Поищем файл на NFS сервере
a.belyaev@MacBook-Pro-2 manifests % kubectl describe pv/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac | grep -i path
                     Path = /export/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac;
    Path:      /export/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac
    
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/nfs-server-nfs-server-provisioner-0 -i -t -- ls -l /export/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac
total 4
-rw-r--r-- 1 root root 12 Jun 29 19:59 frontend.file

# Удалим деплой и проверим этот файл на сервере NFS
a.belyaev@MacBook-Pro-2 manifests % kubectl delete -f prod 
deployment.apps "backend" deleted
service "backend" deleted
statefulset.apps "db" deleted
service "db" deleted
deployment.apps "frontend" deleted
service "frontend" deleted
a.belyaev@MacBook-Pro-2 manifests % kubectl get po
NAME                                  READY   STATUS    RESTARTS       AGE
multitool-55974d5464-cgtvf            1/1     Running   1 (157m ago)   3d6h
nfs-server-nfs-server-provisioner-0   1/1     Running   0              89m

# Проверяем
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/nfs-server-nfs-server-provisioner-0 -i -t -- cat /export/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac/frontend.file
My test NFS

# Файл на месте :)
# Снова задеплоим нашу прилагу
a.belyaev@MacBook-Pro-2 manifests % kubectl apply -f prod                                                                                                          
deployment.apps/backend created
service/backend created
statefulset.apps/db created
service/db created
deployment.apps/frontend created
service/frontend created
a.belyaev@MacBook-Pro-2 manifests % kubectl get po
NAME                                  READY   STATUS    RESTARTS       AGE
backend-67d575c954-d2p2r              1/1     Running   0              56s
db-0                                  1/1     Running   0              55s
frontend-79d8b7bf95-wrsj5             1/1     Running   0              55s
multitool-55974d5464-cgtvf            1/1     Running   1 (159m ago)   3d6h
nfs-server-nfs-server-provisioner-0   1/1     Running   0              91m

# И снова проверям
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/frontend-79d8b7bf95-wrsj5 -i -t -- ls -l /static
total 4
-rw-r--r-- 1 root root 12 Jun 29 19:59 frontend.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/backend-67d575c954-d2p2r -i -t -- cat /static/frontend.file
My test NFS

# И снова файл на месте. Попробуем что нибудь записать с backend
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/backend-67d575c954-d2p2r -i -t -- touch /static/backend.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/frontend-79d8b7bf95-wrsj5 -i -t -- ls -l /static            
total 4
-rw-r--r-- 1 root root  0 Jun 29 20:10 backend.file
-rw-r--r-- 1 root root 12 Jun 29 19:59 frontend.file
a.belyaev@MacBook-Pro-2 manifests % kubectl exec po/nfs-server-nfs-server-provisioner-0 -i -t -- ls -l /export/pvc-086732b0-bef5-48ec-870e-0b8c82dc65ac
total 4
-rw-r--r-- 1 root root  0 Jun 29 20:10 backend.file
-rw-r--r-- 1 root root 12 Jun 29 19:59 frontend.file
```
