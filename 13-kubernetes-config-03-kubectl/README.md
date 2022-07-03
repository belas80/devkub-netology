# 13.3 работа с kubectl  

## Задание 1: проверить работоспособность каждого компонента  

Манифесты из предыдущего задания [здесь](../13-kubernetes-config-02-mounts/manifests/prod)  
Проверка с использованием `exec` и пода `multitool`:  
```shell
# Frontend
% kubectl exec pods/multitool-55974d5464-cgtvf -- curl http://frontend:8000
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html lang="ru">
<head>
    <title>Список</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="/build/main.css" rel="stylesheet">
</head>
<body>
    <main class="b-page">
        <h1 class="b-page__title">Список</h1>
        <div class="b-page__content b-items js-list"></div>
    </main>
    <script src="/build/main.js"></script>
</body>
100   448  100   448    0     0  57502      0 --:--:-- --:--:-- --:--:-- 64000

# Backend
 % kubectl exec pods/multitool-55974d5464-cgtvf -- curl http://backend:9000 -v  
{"detail":"Not Found"}  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 10.233.14.171:9000...
* Connected to backend (10.233.14.171) port 9000 (#0)
> GET / HTTP/1.1
> Host: backend:9000
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< date: Sun, 03 Jul 2022 09:03:43 GMT
< server: uvicorn
< content-length: 22
< content-type: application/json
< 
{ [22 bytes data]
100    22  100    22    0     0   5592      0 --:--:-- --:--:-- --:--:--  7333
* Connection #0 to host backend left intact

# DB Postgres
% kubectl exec pods/multitool-55974d5464-cgtvf -it -- psql -U postgres -W -h db -c '\l'
Password: 
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 news      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
```
Проверка с использованием `port-forward` с локального компьютера:  
```shell
# Frontend
# Проверка перед запуском port-forward
a.belyaev@MacBook-Pro-2 devkub-netology % curl localhost:8888         
curl: (7) Failed to connect to localhost port 8888 after 27 ms: Connection refused
# Запуск port-forward
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl port-forward --address 0.0.0.0 deploy/frontend 8888:80
Forwarding from 0.0.0.0:8888 -> 80
# Снова проверка curl
a.belyaev@MacBook-Pro-2 devkub-netology % curl localhost:8888
<!DOCTYPE html>
<html lang="ru">
<head>
    <title>Список</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="/build/main.css" rel="stylesheet">
</head>
<body>
    <main class="b-page">
        <h1 class="b-page__title">Список</h1>
        <div class="b-page__content b-items js-list"></div>
    </main>
    <script src="/build/main.js"></script>
</body>
</html>%

# Backend
# Предварительная проверка порта
a.belyaev@MacBook-Pro-2 devkub-netology % curl localhost:9999
curl: (7) Failed to connect to localhost port 9999 after 11 ms: Connection refused
# Запуск port-forward
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl port-forward --address 0.0.0.0 deploy/backend 9999:9000
Forwarding from 0.0.0.0:9999 -> 9000
# Повторная проверка
a.belyaev@MacBook-Pro-2 devkub-netology % curl localhost:9999
{"detail":"Not Found"}%

# DB Postgres
# Предварительная проверка
a.belyaev@MacBook-Pro-2 devkub-netology % telnet 127.1 5432    
Trying 127.0.0.1...
telnet: connect to address 127.0.0.1: Connection refused
telnet: Unable to connect to remote host
# port-forward
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl port-forward --address 0.0.0.0 sts/db 5432            
Forwarding from 0.0.0.0:5432 -> 5432
# Проверка
a.belyaev@MacBook-Pro-2 devkub-netology % telnet 127.1 5432
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

## Задание 2: ручное масштабирование  

```shell
# Текущее состояние
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl get po -o wide 
NAME                                  READY   STATUS    RESTARTS      AGE     IP             NODE    NOMINATED NODE   READINESS GATES
backend-679695d447-k8t8w              1/1     Running   0             8m29s   10.233.90.60   node1   <none>           <none>
db-0                                  1/1     Running   1 (17m ago)   8h      10.233.96.62   node2   <none>           <none>
frontend-79d8b7bf95-khpkv             1/1     Running   1 (17m ago)   8h      10.233.90.59   node1   <none>           <none>
multitool-55974d5464-cgtvf            1/1     Running   3 (17m ago)   7d3h    10.233.90.57   node1   <none>           <none>
nfs-server-nfs-server-provisioner-0   1/1     Running   2 (17m ago)   3d23h   10.233.96.61   node2   <none>           <none>

# Увеличиваем реплики backend и frontend до 3-х
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl scale --replicas 3 deployment/frontend deployment/backend 
deployment.apps/frontend scaled
deployment.apps/backend scaled
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl get po -o wide                                           
NAME                                  READY   STATUS    RESTARTS      AGE     IP             NODE    NOMINATED NODE   READINESS GATES
backend-679695d447-c2dr4              1/1     Running   0             14s     10.233.96.66   node2   <none>           <none>
backend-679695d447-k8t8w              1/1     Running   0             11m     10.233.90.60   node1   <none>           <none>
backend-679695d447-nrtr4              1/1     Running   0             14s     10.233.90.61   node1   <none>           <none>
db-0                                  1/1     Running   1 (20m ago)   8h      10.233.96.62   node2   <none>           <none>
frontend-79d8b7bf95-766mw             1/1     Running   0             14s     10.233.96.65   node2   <none>           <none>
frontend-79d8b7bf95-b5t86             1/1     Running   0             14s     10.233.96.64   node2   <none>           <none>
frontend-79d8b7bf95-khpkv             1/1     Running   1 (20m ago)   8h      10.233.90.59   node1   <none>           <none>
multitool-55974d5464-cgtvf            1/1     Running   3 (20m ago)   7d3h    10.233.90.57   node1   <none>           <none>
nfs-server-nfs-server-provisioner-0   1/1     Running   2 (20m ago)   3d23h   10.233.96.61   node2   <none>           <none>
# По столбцу AGE (14s) видно, что дополнительные реплики frontend запустились на node2, а backend на node1 и node2

# Проверка с помощью describe
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl describe deploy/frontend | grep -i ^replicas
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl describe deploy/backend | grep -i ^replicas 
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable

# Уменьшим количество до 1
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl scale --replicas 1 deploy/frontend deployment/backend 
deployment.apps/frontend scaled
deployment.apps/backend scaled
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl describe deploy/frontend | grep -i ^replicas         
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl describe deploy/backend | grep -i ^replicas          
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
a.belyaev@MacBook-Pro-2 devkub-netology % kubectl get po -o wide 
NAME                                  READY   STATUS    RESTARTS      AGE     IP             NODE    NOMINATED NODE   READINESS GATES
backend-679695d447-c2dr4              1/1     Running   0             8m13s   10.233.96.66   node2   <none>           <none>
db-0                                  1/1     Running   1 (28m ago)   9h      10.233.96.62   node2   <none>           <none>
frontend-79d8b7bf95-khpkv             1/1     Running   1 (28m ago)   9h      10.233.90.59   node1   <none>           <none>
multitool-55974d5464-cgtvf            1/1     Running   3 (28m ago)   7d4h    10.233.90.57   node1   <none>           <none>
nfs-server-nfs-server-provisioner-0   1/1     Running   2 (28m ago)   3d23h   10.233.96.61   node2   <none>           <none>
```
