# 12.4 Развертывание кластера на собственных серверах, лекция 2  

## Задание 1: Подготовить инвентарь kubespray  

Подготовим ноды для кластера в Yandex Cloud.  
```shell
% yc compute instance list
+----------------------+-----------+---------------+---------+----------------+---------------+
|          ID          |   NAME    |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP  |
+----------------------+-----------+---------------+---------+----------------+---------------+
| epd2ka8micogtedg6od7 | cp1       | ru-central1-b | RUNNING | 51.250.110.221 | 192.168.99.28 |
| epd6lvj2o85h95qme5os | node2     | ru-central1-b | RUNNING | 84.252.143.177 | 192.168.99.15 |
| epdasikuldnt8ajd3asv | node3     | ru-central1-b | RUNNING | 84.201.139.90  | 192.168.99.17 |
| epdqhhnc92kh0qodlas5 | node4     | ru-central1-b | RUNNING | 51.250.29.222  | 192.168.99.14 |
| epdui06mu6eqm6vnqu2k | node1     | ru-central1-b | RUNNING | 84.201.160.128 | 192.168.99.9  |
+----------------------+-----------+---------------+---------+----------------+---------------+
```
Склонируем себе репозиторий kubespray `git clone https://github.com/kubernetes-sigs/kubespray`, и далее по инсрукции.  
В итоге, подготовим файл `inventory/mycluster/hosts.yaml` следующего содержания:  
```yaml

```
Так же изменим файл `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml` следующими значениями:  
```yaml
supplementary_addresses_in_ssl_keys: [51.250.110.221] # Для удаленного доступа с рабочего компа, пропишем внешний IP контрол ноды.
container_manager: containerd                         # Выбор CRI. Был по умолчанию.
```
