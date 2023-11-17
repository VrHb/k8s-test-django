# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как развернуть сайт с помощью kubernetes

### Установите minikube, см. [доку](https://minikube.sigs.k8s.io/docs/start/)

### Запустите контейнер с бд
- в `docker-compose` поменяйте публичный IP для доступа к бд

```sh 
docker-compose up -d db 
```

### Создайте config-файл с переменными окружения, [документация](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

### Подгрузите в кластер переменные окружения

```sh 
kubectl apply -f config.yml
```

#### Чтобы обновить configmap 

```sh 
kubectl apply -f config.yml
```

```sh 
kubectl rollout restart deployment
```

### Разверните деплоимент

```sh 
kubectl apply -f django-deployment.yml
```

### Запустите сервис

```sh 
minikube service django-service
```

### Запустите cronjob для удаления истекших сессий django 

```sh 
kubectl apply -f django-cronjon.yml
```

- Можно запустить принудительно


```sh 
kubectl create job clearsession --from=cronjob/django-clearsession-cronjob 
```

#### Настройка ingress

1. активируйте ingress в minikube

```sh 
minikbue addons enable ingress
```

2. измените домен в `/etc/hosts` и добавте его в ALLOWED_HOSTS в файле `config.yml`

3. добавьте его в `ingress-django.yml`

```yaml 
spec:
  rules:
  - host: <you_domain>
```

4. запустите ingress в класстере

```sh 
kubectk apply -f ingress-django.yml
```
