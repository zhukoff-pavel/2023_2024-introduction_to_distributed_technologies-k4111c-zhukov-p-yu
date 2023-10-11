# Развертывание веб-сервиса в Minikube, доступ к веб-интерфейсу сервиса. Мониторинг сервиса

University: [ITMO University](https://itmo.ru/ru/)\
Faculty: [FICT](https://fict.itmo.ru)\
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)\
Year: 2023/2024\
Group: K4111c\
Author: Zhukov Pavel Yurievich\
Lab: Lab2\
Date of create: 11.10.2023\
Date of finished:

## 0. Введение

###  Цель работы
Ознакомиться с типами "контроллеров" развертывания контейнеров, ознакомиться с сетевыми сервисами и развернуть свое веб-приложение

### Задание
- Необходимо создать `deployment` с 2 репликами контейнера [ifilyaninitmo/itdt-contained-frontend:master](https://hub.docker.com/repository/docker/ifilyaninitmo/itdt-contained-frontend) и передать переменные в эти реплики: `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME`.

- Создать сервис через который у вас будет доступ на эти "поды". Выбор типа сервиса остается на ваше усмотрение.

- Запустить в `minikube` режим проброса портов и подключитесь к вашим контейнерам через веб браузер.

- Проверить на странице в веб браузере переменные `REACT_APP_USERNAME`, `REACT_APP_COMPANY_NAME` и `Container name`. Изменяются ли они? Если да, то почему?

- Проверить логи контейнеров, приложить логи в отчёт.

## 1. Ход работы
### 1.1. Создание деплоймента
Опишем деплоймент:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: itdt-contained-frontend
        image: ifilyaninitmo/itdt-contained-frontend:master
        ports:
        - containerPort: 3000
          name: http
        env:
          - name: REACT_APP_USERNAME
            value: 'zhukoff-pavel'
          - name: REACT_APP_COMPANY_NAME
            value: 'FICT ITMO'
```
Кратко пробежимся по назначению полям в `spec`:
* `replicas` отвечает за количество реплик контейнеров, которые необходимо создать
* `selector` указывает `ReplicaSet` (которая создается деплойментом), какими подами она сможет управлять. По большому счету, это {k, v}-мапа, и для выбора конкретного пода нужно совпадение всех ключей с метками.
* `template` описывает шаблон подов, которые необходимо создать. Внутри него все идентичном заданию из первой лабы, но здесь мы добавили еще одно поле, о нём ниже.
* `env` позволяет выставить переменные окружения внутри контейнеров.

### 1.2. Создание сервиса
Создать сервис можно, конечно, и командой
```yaml
kubectl expose deployment.apps app --type=NodePort --port=3000
```

Но это неинтересно, потому что так мы уже делали в прошлой работе.
Опишем сервис в том же манифесте:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  type: NodePort
  ports:
    - port: 3000
      protocol: TCP
      name: http
  selector:
    app: app
```

Такое описание повторяет действие команды, но является более интересным с точки зрения повышенной кастомизации, что может быть (и будет) в случае более масштабных задач.

### 1.3. Запуск

Всю эту красоту, что мы написали в двух предыдущих пунктах, запустим командой
```
kubectl apply -f manifest.yaml
```

![](/lab2/sources/applying_manifest.png)

Статус `unchanged` означает, что с момента предыдущего запуска конфигурация соответствующей сущности не изменилась.

Убедимся в том, что все создалось.
```
kubectl get deployments
kubectl get rs
kubectl get pods
```
![](/lab2/sources/deployments.png)
![](/lab2/sources/replicas.png)
![](/lab2/sources/pods.png)

Все существует и запущено, отлично.

### 1.4. Подключение к контейнерам

Тут все стандартно - запускаем проброс портов и в браузер:

```
kubectl port-forward service/app 3000:3000
```
![](/lab2/sources/forwarding.png)

![](/lab2/sources/browser.png)
![](/lab2/sources/browser2.png)

Тут можно увидеть те значения переменных, что мы проставляли в манифесте. Они не меняются от контейнера к контейнеру.

### 1.5. Логи
Логи можно посмотреть несколькими способами.
```
kubectl logs <тип/название сущности>
```

Посмотрим лог пода.
```
kubectl logs app-d666448d6-m9l2k
```

![](/lab2/sources/logs.png)

Для другого пода лог идентичен

## 2. Схема

![](/lab2/sources/scheme.png)