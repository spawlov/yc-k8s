# Запуск в кластере Yandex Cloud

### Сборка приложения

Для сборки и отправки в репозитарий [hub.docker.com](hub.docker.com) необходимо запустить, в папке `backend_main_django` следующий скрипт (заменив `namespace_in_dockerhub` на свой логин в `dockerhub`).

Создание образа:
```shell
docker build -t <namespace_in_dockerhub>/k8s-web:latest .
```
Загрузка в dockerhub:
```shell
docker pull <namespace_in_dockerhub>/k8s-web:latest
```
### Установка secrets - переменных окружения и SSL сертификата

Проверьте, возможно уже есть `Secret` с сертификатом для подключения к базе данных, в этом случае можно пропустить шаг с установкой сертификата и созданием Secret:

```shell
kubectl get secret
kubectl get secret pg-root-cert -o yaml
```

#### Установка рутового SSL сертификата для подключения к Postgres
Для верификации сервера при шифровании соединения необходимо поместить в secrets рутовый сертификат Яндекса. 
Для этого его необоходимо сначала [скачать](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect?utm_referrer=https%3A%2F%2Fgithub.com%2Fspawlov%2Fyc-k8s%2Ftree%2Fmain%2Fyc-sirius%2Fedu-goofy-allen#get-ssl-cert) со страницы справки. 
Затем поместить в secrets:
```shell
kubectl create secret generic pg-root-cert --from-file=<path_to_cert>root.crt
```

#### Переменные окружения
В манифест `django-secrets.yaml`, надо прописать следующие данные:
 - необходимо генерировать случайную строку для переменной `SECRET_KEY` и задать `DEBUG` режим.
 - указать в списке `ALLOWED_HOSTS` ваш домен.

### Запуск приложения

Перед запуском приложения нужно применить миграции. Для применения миграций к базе при обновлении приложения используется следующая команда:
```shell
kubectl apply -f migrate
```

Для запуска приложения нужно создать объект Deployment:
```shell
kubectl apply -f .
```

сайт будет доступен по адресу: [edu-goofy-allen.sirius-k8s.dvmn.org](https://edu-goofy-allen.sirius-k8s.dvmn.org/)

