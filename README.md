# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up -d
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web python ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web python ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8080/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


### Как запустить приложение через k8s

1. [Установить minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)
2. В директории `k8s` создать файл `secrets.yaml`. В нём будут храниться конфиденциальные данные, 
   которые предварительно надо закодировать в [Base64](http://base64.ru/). Структура файла должна быть следующей:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
       name: app-secrets
   type: Opaque
   data:
       # Здесь как раз и указываются все конфиденциальные данные в Base64
       DB_DATABASE: YXBwX2Ri
       DB_USER: YWRtaW4= 
       DB_PASSWORD: emFxMTIzZWRjeHN3Mg==
       SECRET_KEY: c2VjcmV0c2VjcmV0c2VjcmV0c2VjcmV0c2VjcmV0 
   ```
3. Запустить minikube и после выполнить следующие команды:
   ```shell
   minikube start
   minikube status # Проверка успешного запуска
   minikube tunnel # Для вывода во внешний мир minikubu, который запущен в Docker
   
   # Следующие команды вводятся в новом оне терминала
   minikube addons enable ingress # Установка компонентов для входящего трафика 
   minikube dashboard # Запуск веб-интерфейса для работы с minikubu  
   ```
4. [Установить kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
   ```shell
   kubectl cluster-info # Проверка установки
   ```
5. [В файл `host` добавить строку](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit):
   ```text
   127.0.0.1 star-burger.test
   ```
6. Запустить приложение с помощью команд:
   ```shell
   kubectl apply -f k8s/secrets.yaml
   kubectl apply -f k8s/config.yaml
   kubectl apply -f k8s/postgres.yaml
   kubectl apply -f k8s/jobs.yaml
   kubectl apply -f k8s/django.yaml
   kubectl apply -f k8s/ingress.yaml
   ```
   Проверка запуска:
   ```shell
   kubectl get configmap -o wide
   kubectl get secrets -o wide
   kubectl get deployment -o wide
   kubectl get pods -o wide
   kubectl get svc -o wide
   kubectl get pvc -o wide
   kubectl get ingress -o wide
   kubectl describe ingress  
   ```
   Набор команды, если вдруг надо полностью удалить приложение в minikube:
   ```shell
   kubectl delete deploy django-deployment
   kubectl delete deploy postgres-deployment
   kubectl delete cronjobs django-job-clearsessions
   kubectl delete jobs django-job-migrations
   kubectl delete svc django-cluster-ip-service
   kubectl delete svc postgres-cluster-ip-service
   kubectl delete ingress ingress-service
   kubectl delete configmap app-config
   kubectl delete pvc postgres-persistent-volume-claim
   kubectl delete secrets app-secrets
   ```
7. Создать администратора Django:
   ```shell
   kubectl get pods -o wide 
   # Из списка подов выбрать любой django-deployment-x...-x...
   kubectl exec django-deployment-5bf545fd5f-6ssvw -it -- bash # Входим в терминал пода
   # Внутри пода ddtcnb команду
   python manage.py createsuperuser
   ```
8. Приложение развёрнуто и доступно по адресу http://star-burger.test