# 13.3 работа с kubectl  

## Задание 1: проверить работоспособность каждого компонента  

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
Проверка с использованием `port-forward`:  
```shell

```