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

Затем выполните миграции:
```shell-session
$ kubectl apply -f .kube/migrate/
```
Запустите деплой проекта командой:
```sell-session
$ kubectl apply -f .kube/
```
Проект будет доступен по адресу [http://star-burger.test](http://star-burger.test)
