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

## Запуск в кластере minikube

Должен быть установлен [VirtualBox](https://www.virtualbox.org/), а также [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/), [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) и [Helm](https://helm.sh/)

Создайте кластер на виртуальной машине:
```shell-session
$ minikube start --vm-driver=virtualbox
```
затем разверните в кластере PostgreSQL:
```shell-session
$ helm install <имя базы> oci://registry-1.docker.io/bitnamicharts/postgresql
```
подключитесь к базе через `psql` (будет создан еще один под, который автоматически будет удален по окончанию работы по настройке):
```shell-session
$ export POSTGRES_PASSWORD=$(kubectl get secret --namespace default <имя базы>-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
$ kubectl run <имя базы>-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.4.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host dev-postgresql -U postgres -d postgres -p 5432
```
для создания новой базы для приложения и пользователя, выполните команды:
```shell-session
postgres=# CREATE DATABASE <db_name>;
postgres=# CREATE USER <db_user> WITH PASSWORD '<db_password>';
postgres=# ALTER ROLE <db_user> SET client_encoding TO 'utf8';
postgres=# ALTER ROLE <db_user> SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE <db_user> SET timezone TO 'UTC';
postgres=# ALTER DATABASE <db_name> OWNER TO <db_user>;
postgres=# GRANT ALL PRIVILEGES ON DATABASE <db_name> TO <db_user>;
postgres=# \q
```
отредактируйте файлы `django-config.yaml` и `django-service.yaml`

_IP адрес_ и _host_ можно получить, выполнив соответсвующие команды:
```shell-session
$ minikube ip
$ kubectl get svc
```
В файле `hosts` поставьте в соответсвие с вашим `IP` домен `star-burger.test`

Запустите деплой проекта комендой:
```sell-session
$ kubectl apply -f .kube/
```
Затем выполните миграции:
```shell-session
$ kubectl apply -f .kube/migrate/
```
Проект будет доступен по адресу [http://star-burger.test](http://star-burger.test)

