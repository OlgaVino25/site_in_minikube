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
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

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

## Запуск PostgreSQL (для Kubernetes)

Перед развёртыванием Django в Kubernetes убедитесь, что запущен контейнер с PostgreSQL, к которому будет подключаться приложение. Выполните в терминале:

```bash
# powershell
docker run -d --name postgres-local -p 5432:5432 `
  -e POSTGRES_DB=test_k8s `
  -e POSTGRES_USER=test_k8s `
  -e POSTGRES_PASSWORD=OwOtBep9Frut `
  postgres:12.0-alpine

# Linux/macOS
docker run -d --name postgres-local -p 5432:5432 \
  -e POSTGRES_DB=test_k8s \
  -e POSTGRES_USER=test_k8s \
  -e POSTGRES_PASSWORD=OwOtBep9Frut \
  postgres:12.0-alpine
```

*Здесь используются тестовые учётные данные. При желании вы можете изменить их, но тогда не забудьте указать те же значения в секрете*

Если контейнер уже создан, но остановлен, запустите его:

```bash
docker start postgres-local
```

### Настройка секретов

Перед применением манифестов необходимо создать Secret с конфиденциальными данными. Это можно сделать двумя способами:

*IP Minikube можно узнать командой `minikube ip`*
*Для подключения к PostgreSQL, запущенному на хосте, используйте IP-адрес шлюза Minikube. Его можно узнать командой:*

```bash
minikube ssh "ip route show default | awk '{print $3}'"
```

Эта команда выведет IP шлюза (например, 192.168.49.1 для драйвера Docker или 192.168.59.1 для VirtualBox).
*Примечание для Windows*: если команда не сработает, вы можете зайти в Minikube вручную:

```bash
minikube ssh
```

Затем внутри виртуальной машины выполните:

```bash
ip route show default
```

Найдите строку, начинающуюся с default via, и скопируйте следующий за ней IP-адрес. Выйдите из ssh командой exit.

*В примерах замените <host-ip> на этот адрес (обычно 192.168.49.1 для драйвера Docker или 192.168.59.1 для VirtualBox).*

**Вариант 1 (через файл, не коммитить):**

1. Скопируйте `secrets.yaml.example` в `secrets.yaml` и заполните своими значениями.
2. Примените: `kubectl apply -f secrets.yaml`
3. Убедитесь, что `secrets.yaml` добавлен в `.gitignore`.

**Вариант 2 (через командную строку, без сохранения в файл):**

```bash
kubectl create secret generic django-secrets \
  --from-literal=SECRET_KEY="your-secret-key" \
  --from-literal=DATABASE_URL="postgres://test_k8s:OwOtBep9Frut@<host-ip>:5432/test_k8s" \
  --from-literal=ALLOWED_HOSTS="localhost,127.0.0.1,<minikube-ip>,star-burger.test"

# для powershell

kubectl create secret generic django-secrets `
  --from-literal=SECRET_KEY="your-secret-key" `
  --from-literal=DATABASE_URL="postgres://test_k8s:OwOtBep9Frut@<host-ip>:5432/test_k8s" `
  --from-literal=ALLOWED_HOSTS="localhost,127.0.0.1,<minikube-ip>,star-burger.test"
```

2. Примените манифесты:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

3. Для доступа к сайту в новом терминале:

```bash
minikube service django-service
```

## Развёртывание в Kubernetes с Ingress

**Предварительные требования:**

- Установленный гипервизор или контейнерный движок (например, Docker Desktop или VirtualBox) в зависимости от выбранного драйвера Minikube
- Minikube и kubectl
- Запущенный контейнер PostgreSQL (см. раздел «Запуск PostgreSQL»)

1. Запустите Minikube

```bash
minikube start
```

2. Включите Ingress-контроллер

```bash
minikube addons enable ingress
```

Дождитесь, пока поды ingress-nginx перейдут в состояние Running (`kubectl get pods -n ingress-nginx`).

3. Создайте секрет с конфиденциальными данными

Скопируйте файл-образец и отредактируйте его:

```bash
cp secrets.yaml.example secrets.yaml
# отредактируйте secrets.yaml, указав свои значения
```

Затем примените секрет:

```bash
kubectl apply -f secrets.yaml
```

*`secrets.yaml` – уже добавлен в `.gitignore`.*

4. Примените остальные манифесты

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

5. Примените миграции (если база новая)

```bash
kubectl exec -it deploy/django-app -- python manage.py migrate
```

6. Проверьте созданные ресурсы

```bash
kubectl get pods
kubectl get services    # django-service должен быть типа ClusterIP
kubectl get ingress     # должен быть указан адрес (например, 192.168.49.2)
```

7. Настройте локальный доступ к домену

Узнайте IP-адрес Minikube:

```bash
minikube ip
```

Далее действия зависят от используемого драйвера Minikube:

- **Для драйвера Docker** (по умолчанию на Windows):
Ingress-контроллер становится доступен через локальный туннель. Выполните в отдельном терминале (не закрывайте его):

```bash
minikube service ingress-nginx-controller -n ingress-nginx
```

Эта команда откроет браузер и покажет локальный адрес вида http://127.0.0.1:xxxxx. Запомните порт (например, 60743).

Отредактируйте файл `C:\Windows\System32\drivers\etc\hosts`.
Для быстрого открытия выполните в PowerShell от имени администратора:

```bash
notepad C:\Windows\System32\drivers\etc\hosts
```

Добавьте в конец файла строку:

```text
127.0.0.1   star-burger.test
```

*Для Linux/macOS: отредактируйте файл /etc/hosts с правами суперпользователя:*

```bash
sudo nano /etc/hosts
```

- **Для драйвера VirtualBox** (или других гипервизоров):
Ingress-контроллер доступен напрямую по IP Minikube (обычно 192.168.59.xxx).
В файл hosts добавьте строку в конец файла и замените <IP_Minikube> на значение из minikube ip:

```text
<IP_Minikube>   star-burger.test
```

*Если вы хотите использовать другой домен, отредактируйте поле host в файле ingress.yaml перед применением манифеста.*

После изменения hosts сбросьте кэш DNS:

```bash
ipconfig /flushdns
```

8. Откройте сайт

- **Для драйвера Docker:** перейдите по адресу `http://star-burger.test:xxxxx/admin/` (подставьте порт из шага 6).
- **Для драйвера VirtualBox:** перейдите по адресу `http://star-burger.test/admin/` (без порта).

Вы должны увидеть страницу входа в админку Django.

9. Создайте суперпользователя (если нужно)

```bash
kubectl exec -it deploy/django-app -- python manage.py createsuperuser
```

**Примечание:** При использовании драйвера Docker туннель нужно держать открытым всё время работы с сайтом. При каждом перезапуске Minikube порт может меняться – повторяйте шаг 6 для получения актуального адреса. Также при смене драйвера может измениться IP Minikube, и тогда нужно обновить hosts.

## Очистка устаревших сессий Django

Для автоматического удаления устаревших сессий используется CronJob, который запускает команду `python manage.py clearsessions` ежедневно в 02:00.

1. Примените манифест:

```bash
kubectl apply -f cronjob.yaml
```

2. Проверьте создание CronJob:

```bash
kubectl get cronjobs
```

3. Для немедленного запуска (например, для теста) создайте Job вручную:

```bash
kubectl create job --from=cronjob/django-clearsessions-cronjob django-clearsessions-manual
```

4. Убедитесь, что Job выполнилась успешно:

```bash
kubectl get jobs
kubectl logs <pod-name>
```

При желании можно изменить расписание, отредактировав поле `schedule` в `cronjob.yaml`.