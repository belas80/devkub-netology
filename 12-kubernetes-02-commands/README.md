# 12.2 Команды для работы с Kubernetes  

## Задание 1: Запуск пода из образа в деплойменте  

Запуск 2-х реплик nginx через деплой в неймспейсе app-namespace:  
```bash
~$ kubectl create namespace app-namespace
namespace/app-namespace created

~$ kubectl create deployment nginx --namespace app-namespace --image nginx:latest --replicas 2
deployment.apps/nginx created

~$ kubectl get deployments.apps -n app-namespace 
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           25s

~$ kubectl get pods -n app-namespace 
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7c658794b9-b2f2b   1/1     Running   0          51s
nginx-7c658794b9-pwgmw   1/1     Running   0          51s
```

## Задание 2: Просмотр логов для разработки  

Cоздание пользователя и выдача ему доступа на чтение конфигурации и логов подов в app-namespace.  
Созданим сертификат и ключ для нашего пользователя следующими командами:  
```bash
openssl genrsa -out user1.key 2048
openssl req -new -key user1.key -out user1.csr -subj "/CN=user1"
openssl x509 -req -in user1.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out user1.crt -days 500
```
Далее отредактиктируем `~/.kube/config`. Можно вручную, а можно командами ниже:  
```bash
kubectl config set-credentials user1 --client-certificate ~/certs/user1.crt --client-key ~/certs/user1.key
kubectl config set-context user1-context --cluster minikube --user=user1 --namespace app-namespace
```
В итоге оплучим следующий конфиг:  
```bash
~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/ubuntu/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Tue, 07 Jun 2022 17:39:46 UTC
        provider: minikube.sigs.k8s.io
        version: v1.25.2
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Tue, 07 Jun 2022 17:39:46 UTC
        provider: minikube.sigs.k8s.io
        version: v1.25.2
      name: context_info
    namespace: default
    user: minikube
  name: minikube
- context:
    cluster: minikube
    namespace: app-namespace
    user: user1
  name: user1-context
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/ubuntu/.minikube/profiles/minikube/client.crt
    client-key: /home/ubuntu/.minikube/profiles/minikube/client.key
- name: user1
  user:
    client-certificate: /home/ubuntu/certs/user1.crt
    client-key: /home/ubuntu/certs/user1.key
```
Далее созданим RoleBinding для нашего пользователя на просмотр с помощью файла `roleuser1.yaml` следующего содержания:  
```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: user1
  namespace: app-namespace
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```
Применим его:  
```bash
~$ kubectl apply -f roleuser1.yaml
rolebinding.rbac.authorization.k8s.io/user1 created
```
Проверим нашего пользователя:  
```bash
~$ kubectl get pods --as user1
Error from server (Forbidden): pods is forbidden: User "user1" cannot list resource "pods" in API group "" in the namespace "default"

~$ kubectl get pods -n app-namespace --as user1
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7c658794b9-b2f2b   1/1     Running   0          28m
nginx-7c658794b9-pwgmw   1/1     Running   0          28m
```
Пользователь `user1` может просматривать ресурсы только в неймспейсе `app-namespace`.  
Ну и еще несколько проверок на чтение логов и созадние ресурсов в конетексте этого пользователя:  
```bash
~$ kubectl config use-context user1-context 
Switched to context "user1-context".

~$ kubectl get deployments.apps,po
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   2/2     2            2           32m

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-7c658794b9-b2f2b   1/1     Running   0          32m
pod/nginx-7c658794b9-pwgmw   1/1     Running   0          32m

~$ kubectl logs pods/nginx-7c658794b9-b2f2b --tail 5
2022/06/07 17:56:24 [notice] 1#1: start worker processes
2022/06/07 17:56:24 [notice] 1#1: start worker process 31
2022/06/07 17:56:24 [notice] 1#1: start worker process 32
2022/06/07 17:56:24 [notice] 1#1: start worker process 33
2022/06/07 17:56:24 [notice] 1#1: start worker process 34

~$ kubectl describe pods/nginx-7c658794b9-pwgmw 
Name:         nginx-7c658794b9-pwgmw
Namespace:    app-namespace
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Tue, 07 Jun 2022 17:56:22 +0000
Labels:       app=nginx
              pod-template-hash=7c658794b9
Annotations:  <none>
Status:       Running
IP:           172.17.0.5
...
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  36m   default-scheduler  Successfully assigned app-namespace/nginx-7c658794b9-pwgmw to minikube
  Normal  Pulling    36m   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     36m   kubelet            Successfully pulled image "nginx:latest" in 3.077978224s
  Normal  Created    36m   kubelet            Created container nginx
  Normal  Started    36m   kubelet            Started container nginx

~$ kubectl create deployment nginx2 --image nginx:latest
error: failed to create deployment: deployments.apps is forbidden: User "user1" cannot create resource "deployments" in API group "apps" in the namespace "app-namespace"

~$ kubectl edit deployments.apps nginx 
error: deployments.apps "nginx" could not be patched: deployments.apps "nginx" is forbidden: User "user1" cannot patch resource "deployments" in API group "apps" in the namespace "app-namespace"
You can run `kubectl replace -f /tmp/kubectl-edit-821343259.yaml` to try this update again.
```

## Задание 3: Изменение количества реплик  

Необходимо изменить запущенный deployment, увеличив количество реплик до 5.  
```bash
~$ kubectl scale -n app-namespace deployment nginx --replicas 5
deployment.apps/nginx scaled

~$ kubectl get -n app-namespace deployment,po
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   5/5     5            5           113m

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-b979bf8bf-dbwg8   1/1     Running   0          19m
pod/nginx-b979bf8bf-dcjpg   1/1     Running   0          18s
pod/nginx-b979bf8bf-k2v9x   1/1     Running   0          18s
pod/nginx-b979bf8bf-pj86x   1/1     Running   0          19m
pod/nginx-b979bf8bf-szb7v   1/1     Running   0          18s
```
