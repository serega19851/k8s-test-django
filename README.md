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
Создайте фаил .env в корне репозитория.

```
POSTGRES_DB=test_k8s
POSTGRES_USER=test_k8s
POSTGRES_PASSWORD=OwOtBep9Frut

SECRET_KEY= ...
DEBUG=False
DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@db:5432/test_k8s
ALLOWED_HOSTS=127.0.0.1,localhos
```

## Kubernetes

Установим `minikube` и `kubectl`. Для этого перейдите по ссылкам и скачайте их:
[kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и [minikube](https://minikube.sigs.k8s.io/docs/start/).

Если вы работает на `Windows`, то вам необходимо прописать путь до папки с программами `minikube` и `kubectl` в переменной среде `Path`.

Запустим сам `minikube` (настройки `--cpus`, `--memory` и `--disk-size` опциональны):

```
minikube start --vm-driver=docker --cpus=3 --memory=4gb --disk-size=10gb
```

Настройте Ingress:

- Установите

```shell
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

- В файле `django-ingress.yml` Меняем `host` на свой
- Пропишите на своей машине в `/etc/hosts` (для linux) домен `star-burger.test`, сопоставить с IP виртуальной машины
  узнать ip c помощью команды:

```shell
- minikube ip
```

Заполните файл `configmap.yaml`.

```
SECRET_KEY: "django-insecure-0if40nf4nf93n4"
DEBUG: "False"
ALLOWED_HOSTS: "127.0.0.1,localhos,star-burger.test"
DATABASE_URL: "postgres://test_k8s:OwOtBep9Frut@Your local machine IP address:5432/test_k8s"
```

Теперь запустим все `yaml-файлы` по очереди:

```
kubectl apply -f configmap.yaml
kubectl apply -f django-deployment.yaml
kubectl apply -f django-ingress.yaml
django-clearsessions.yaml
django-migrate.yaml
```

Теперь переходите по адресу `star-burger.test`.

Подключения PostgreSQL внутри кластера.

Установить [Helm](https://helm.sh/)

```shell
sudo snap install helm --classic
```

Установить PostgreSQL (не забудьте вставить ваш пароль от админ-пользователя(OwOtBep9Frut с configmap.yaml ) БД в конце команды)

```shell
helm install test-db oci://registry-1.docker.io/bitnamicharts/postgresql --set auth.postgresPassword=<put-your-admin-password-here>
```

Сделать экспорт пароля от админ-пользователя БД в переменную окружения:

```shell
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default test-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
```

Создать `pod` с утилитой `psql` и выполнить в ней команду подключения к БД (после запуска команды дождитесь загрузки и появления `postgres=# `):

```shell
kubectl run test-db-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r7 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host test-db-postgresql -U postgres -d postgres -p 5432
```

Создать пользователя БД (и пароль) для подключения приложения:

```shell
CREATE ROLE <username> WITH LOGIN ENCRYPTED PASSWORD '<put-your-password-here>';
```

Создать базу для работы приложения, владельцем которой будет выбранный пользователь:

```shell
CREATE DATABASE <database-name> OWNER <username>;
```

В результате возможно сформировать `DATABASE_URL` следующего вида:

```yaml
DATABASE_URL: postgres://test_k8s:OwOtBep9Frut@test-db-postgresql:5432/test_k8s
```
