# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Создать докер-образ Django на основе Nginx Unit
```shell
cd backend_main_django/
```

```shell
docker build -t django_app .
```

Запустить minikube (на драйвере VirtualBox)
```shell
minikube start --driver=virtualbox
```

Передать образ в minikube
```shell
minikube image load django_app
```

Настроить виртуальную сеть в VirtualBox с двумя сетевыми интерфейсами: Nat + Host-only [по инструкции](https://blogs.oracle.com/scoter/post/oracle-vm-virtualbox-networking-options-and-how-to-manage-them)

Установить [Helm](https://helm.sh/)
```shell
sudo snap install helm --classic
```

Установить PostgreSQL (не забудьте вставить ваш пароль от админ-пользователя БД в конце команды)
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
DATABASE_URL: postgres://<username>:<your-password>@test-db-postgresql:5432/<database-name>
```

Создать .env файл с двумя переменными окружения:
```dotenv
SECRET_KEY=put-your-secret-key-here
DATABASE_URL=postgres://...
```

Создать `Secret` c именем `django-secrets` используя файл `.env` в корневой директории проекта
```shell
kubectl create secret generic django-secrets --from-env-file=./.env
```

Перейти в директорию `kubernetes`
```shell
cd kubernetes/
```

Создать конфиг-файл `django-config.yaml` и в блоке `data` прописать переменные окружения `ALLOWED_HOSTS` и `DEBUG`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  labels:
    app.kubernetes.io/name: django
    app.kubernetes.io/instance: django-test
    app.kubernetes.io/component: web
data:
  ALLOWED_HOSTS: "*"
  DEBUG: "False"
```

Перейти обратно в корневую директорию проекта
```shell
cd ..
```

Запустить манифесты для `configmaps`, `deployment`, `ingress` и `cronjob`
```shell
kubectl apply -f kubernetes/
```

Проверить работу всех компонентов:
```shell
kubectl get deployments.apps,pods,service,configmaps,ingress,cronjobs.batch
```

Для изменения настроек (после внесения изменений в `ConfigMap`)
```shell
kubectl apply -f kubernetes/ && kubectl rollout restart deployment django-deploy
```

Для запуска процесса миграций в БД
```shell
kubectl apply -f django-migrate-job.yaml
```

Для проверки прошедших миграций
```shell
kubectl logs <name of django-migrate-job pod>
```

Список подов можно вывести так
```shell
kubectl get pods
```

Создать супер-пользователя Django (выбрать можно любой из рабочих подов `deploy`)
```shell
kubectl exec -it <django-deploy-pod-name> -- python manage.py createsuperuser
```

Обновить файл /etc/hosts для маршрутизации запросов от [star-burger.test](star-burger.test)
```shell
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения в `django-config.yaml` и в `secrets` (внутри Kubernetes):

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `True` или `False`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,star-burger.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
