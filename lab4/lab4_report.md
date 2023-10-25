# Сети связи в Minikube, CNI и CoreDNS

University: [ITMO University](https://itmo.ru/ru/)\
Faculty: [FICT](https://fict.itmo.ru)\
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)\
Year: 2023/2024\
Group: K4111c\
Author: Zhukov Pavel Yurievich\
Lab: Lab4\
Date of create: 25.10.2023\
Date of finished:

## 0. Введение

###  Цель работы
Познакомиться с CNI Calico и функцией `IPAM Plugin`, изучить особенности работы CNI и CoreDNS.

### Задание
- Запустить minikube c плагином `CNI=calico` и режимом работы `Multi-Node Clusters` одновременно, развернуть 2 ноды;
- Проверить работу CNI плагина Calico, количество нод;
- Указать `label` по признаку стойки или географического расположение (на свое усмотрение);
- Разработать манифест для Calico, который на основе указанных меток назначает IP адреса подам исходя из пулов IP адресов в манифесте.
- Создать деплоймент с двумя репликами контейнера [ifilyaninitmo/itdt-contained-frontend:master](https://hub.docker.com/repository/docker/ifilyaninitmo/itdt-contained-frontend) и передать переменные в эти реплики: `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME`;
- Создать сервис, через который будет доступ в поды;
- Запустить в `minikube` режим проброса портов и подключиться к контейнерам через веб браузер;
- Проверить переменные `Container name` и `Container IP`. Если меняются, объяснить почему;
- Зайти в любой под и попробовать попинговать поды используя FQDN соседнего пода.


## 1. Ход работы
## 1.1. Запуск миникуба
`minikube` в данной работе запускается с помощью следующей команды:
```bash
minikube start --network-plugin=cni --cni=calico --nodes 2
```
![](/lab4/sources/minikube-startup.png)

## 1.2 Проверка числа нод

После успешного запуска проверим, что создалось действительно две ноды с помощью команды
```bash
kubectl get nodes
```

Также не лишним будет проверить количество подов `calico`.
Их число должно совпадать с количеством нод.
```bash
kubectl get pods -l k8s-app=calico-node -A
```

![](/lab4/sources/nodes-listing.png)
![](/lab4/sources/pods-listing.png)

## 1.3 Пометка нод

Каждую ноду в соответствии с заданием пометим по признаку стойки. Так, кажется, будет логичнее.

Пометка делается с помощью команды
```bash
kubectl label nodes <node-name> rack=<rack-id>
```

![](/lab4/sources/nodes-labeling.png)

Можно убедиться, что ноды получили свои метки, например, с помощью команды
```bash
kubectl get nodes -l rack=0
```

![](/lab4/sources/labeled-node-list.png)

## 1.4 Создание Calico-манифеста

Описан [следующий манифест](/lab4/calico.yml).

Пробежимся по его полям:
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: rack-0-ippool
spec:
  cidr: 192.168.10.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "0"
```

В его спеке есть следующие поля:
- `cidr` позволяет определить количество адресов, доступных для конкретно этого IPPool. В нашем случае это `2^8 = 256` адресов.
- `ipipMode` позволяет настроить режим IP-туннелирования.
  Судя по документации, есть два режима:
  - Режим `Always` включает инкапсуляцию пакетов для всего трафика от Calico-хоста к другим Calico-контейнерам и всем VM, имеющим IP в заданном IPPool.
  - Режим `CrossSubnet` включает инкапсуляцию только для того трафика, который ходит между сетями.

  Calico рекомендует использовать режим `CrossSubnet` в случае `ipipMode`, так как это уменьшит накладные расходы на инкапсуляцию.
  Но так как у нас работа маленькая, то можно и использовать режим `Always`.
- `natOutgoing` позволяет разрешить подам ходить во внешнюю сеть с помощью `NAT`.
  В данной работе эта настройка не особо играет роли, так как нам не требуется ходить во внешний интернет и наши поды не развернуты на железках.

- `nodeSelector` позволяет определить, какие ноды должны получать адрес из этого пула. В данном конкретном примере все ноды, находящиеся в "нулевой стойке", будут получать IP из этого пула.


Перед созданием этих IPPools удалим стандартный через `calicoctl delete ippools default-ipv4-ippool`:

![](/lab4/sources/ip-pool-deletion.png)


Применим этот манифест:
```
./calicoctl create -f -< calico.yml
```

![](/lab4/sources/ip-pool-creation.png)

Проверим ippools:
![](/lab4/sources/ip-pool-listing.png)


## 1.5 Создание деплоймента

Манифест с сервисом, деплойментом и конфигмапой можно переиспользовать из третьей лабораторной работы (поменяем только тип сервиса на `LoadBalancer`, так мы сможем запустить тунель через `minikube tunnel` и попадать на разные контейнеры):

```bash
kubectl apply -f manifest.yaml
```

![](/lab4/sources/manifest-apply.png)

Можно теперь убедиться в том, что поды разместились каждый в разной ноде и при этом они получили IP из указанного нами пула.

![](/lab4/sources/pool-correctness.png)

## 1.6 Вход на поды

Запустим режим проброса портов с помощью

```bash
minikube tunnel
```

![](/lab4/sources/minikube-tunnel.png)


![](/lab4/sources/different-pods.png)

Видно, что разные контейнеры имеют разные ip, которые были назначены из соответствующих IPPool'ов.

### Почему меняются Container name и Container IP?
Потому что эти переменные задаются для каждого контейнера по отдельности, а контейнера у нас два, и на какой закинет -- решать LoadBalancer'у.

## 1.7 Ping соседа

Прежде, чем пинговать под, надо найти его FQDN.

Это делается через `nslookup`.
Зайдем в под `app-d7876fd9b-6x54p` и спросим его про имя соседнего пода:

```bash
kubectl exec app-d7876fd9b-6x54p -- nslookup 192.168.20.1
```

![](/lab4/sources/nslookup.png)

Отлично, по FQDN `192-168-20-1.app.default.svc.cluster.local` можем и пинговать:

```bash
kubectl exec app-d7876fd9b-6x54p -- ping 192-168-20-1.app.default.svc.cluster.local
```

![](/lab4/sources/ping.png)

Видно, что пакеты успешно долетают туда и приходят обратно.

## 2. Схема

![](/lab4/sources/scheme.png)