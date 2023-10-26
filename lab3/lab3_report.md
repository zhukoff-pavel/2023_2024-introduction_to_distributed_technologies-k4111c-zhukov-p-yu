# Сертификаты и "секреты" в Minikube, безопасное хранение данных

University: [ITMO University](https://itmo.ru/ru/)\
Faculty: [FICT](https://fict.itmo.ru)\
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)\
Year: 2023/2024\
Group: K4111c\
Author: Zhukov Pavel Yurievich\
Lab: Lab3\
Date of create: 22.10.2023\
Date of finished: 26.10.2023

## 0. Введение

###  Цель работы
Познакомиться с сертификатами и "секретами" в Minikube, правилами безопасного хранения данных в Minikube.

### Задание
- Необходимо сделать `configMap` с переменными: `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME`.
- Создать `replicaSet` с двумя репликами контейнера [ifilyaninitmo/itdt-contained-frontend:master](https://hub.docker.com/repository/docker/ifilyaninitmo/itdt-contained-frontend) и передать переменные в эти реплики: `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME` через этот `configMap`.
- Включить `minikube addons enable ingress` и сгенерировать TLS сертификат, импортировать сертификат в minikube
- Создать ingress в minikube, где указать ранее импортированиный сертификат, FQDN, по которому будем заходить, а также имя ранее созданного сервиса
- В `hosts` прописать FQDN и IP адрес ingress, попробовать перейти в браузере по FQDN
- Зайти в приложение через FQDN по HTTPS и проверить наличие сертификата.

## 1. Ход работы
### 1.1. Что нужно

Для выполнения работы нам необходимо описать несколько сущностей:
* `ConfigMap`, через который мы будем передавать нужные переменные,
* `Deployment`, который мы будем использовать вместо ReplicaSet,
* `Service`
* `Ingress` -- сущность, которая обрабатывает внешние запросы к кластеру. В частности, мы можем указать виды запросов, которые должны обрабатываться (например, какие имена, пути, порты, etc.)

Также нам необходимо создать сертификат.
Его можно описать в манифесте, что не очень хорошо для безопасности в продовых приложениях (легко засветить сертификат), но можно и добавить через cli.
Конечно, возможны более модные и безопасные способы, но эта работа не про них.

### 1.2 Описание манифеста

Опишем `configMap`:
```yaml
apiVersion: v1
data:
  REACT_APP_USERNAME: "zhukoff-pavel"
  REACT_APP_COMPANY_NAME: "ITMO"
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: config
```

В целом, деплоймент и сервис можно переиспользовать из второй ЛР, они почти не поменяются.
Единственное, что изменится, так это в сервисе секция `env` заменится на `envFrom` с указанием имени конфигмапы:

<table class="iksweb">
<tbody>
<tr>
<td> Было </td>
<td> Стало </td>
</tr>
<tr>
<td>

```yaml
env:
    - name: REACT_APP_USERNAME
    value: 'zhukoff-pavel'
    - name: REACT_APP_COMPANY_NAME
    value: 'FICT ITMO'
```

</td>
<td>

```yaml
envFrom:
    - configMapRef:
        name: config
```

</td>
</tr>
	</tbody>
</table>


Опишем `Ingress`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  tls:
    - hosts:
        - test-k8s-lab.app
      secretName: app-tls
  rules:
    - host: test-k8s-lab.app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  name: http
```

Что здесь написано:
* `tls` указывает, серты откуда (поле `secretName`, берется из отдельно созданного секрета) надо использоваться для хостов `hosts`
* `rules` указывает правила обработки тех или иных запросов.
  * `PathType: Prefix` вместе с `path: /` означает, что под это правило подпадают _все_ пути, по которым мы можем обращаться к данному приложению, например, `test-k8s-lab.app/foo/bar` или просто `test-k8s-lab.app`
  * Поле `backend` позволяет указать, на какой конкретно сервис надо направить такой запрос.


### 1.3 Запуск minikube

`Minikube` запускается стандартно, только теперь нам необходимо добавить addon `ingress`.

![](/lab3/sources/ingress-enable.png)

### 1.4 Создание сертификата
Теперь создадим себе сертификат.
Это делается средствами `openssl`, а именно с помощью команды:
```bash
$ openssl req -new -newkey rsa:4096 -x509 -sha256 -days 36
5 -nodes -out selfsigned.crt -keyout selfsigned.key
```

Что делает эта команда?
* `-new` указывает, что нам нужен новый серт.
* `-newkey rsa:4096` мы указываем, что у нас будет ключ длиной 4096 бит.
* `-x509` указывает, что мы создаем сертификат по станларту X.509
* `-sha256` сгенерирует сертификат с использованием `sha256` суммы
* `-days 365` указывает, что срок валидности этого сертификата -- 365 дней
* `-nodes` позволит не зашифровывать приватный ключ парольной фразой
* `-out file.crt` описывает, куда будет помещен полученный сертификат
* `-keyout file.key` описывает, куда будет помещен полученный ключ

![](/lab3/sources/cert-creation.png)

Добавим этот сертификат в наш minikube:

![](/lab3/sources/add-secret.png)

### 1.5 Подготовка кластера
Теперь можно применять конфигурации из манифеста:

![](/lab3/sources/manifest-applying.png)

Допишем FQDN нашего хоста в `/etc/hosts`:
![](/lab3/sources/etc-hosts-exp.png)


### 1.6 В браузер!

Запустим `minikube tunnel`:
![](/lab3/sources/tunnel-start.png)


И можно заходить в браузер:
![](/lab3/sources/page-with-data.png)

> Некоторые браузеры не позволяют зайти на сайт с самоподписанным сертификатом и не предлагают кнопки аля "да, я все понимаю, пустите меня".
> В таком случае можно написать фразу `thisisunsafe` прямо на странице с информацией о неверном сертификате
>
> _Способ работает для Chromium-based браузеров_

Посмотрим наш сертификат:
![](/lab3/sources/browser-with-cert-info.png)

Видно, что это именно тот самый сертификат, который мы выписали двумя пунктами ранее.

### 1.7 Схема

![](/lab3/sources/scheme.png)
