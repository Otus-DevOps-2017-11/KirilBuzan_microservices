# Homework#17 Buzan Kirill
#### 1. Network driver
Запустим контейнеры с различными драйверами:
```bash
docker run --network none --rm -d --name net_test joffotron/docker-net-tools -c "sleep 100
docker run --network host --rm -d --name net_test joffotron/docker-net-tools -c "sleep 100"
```
Сравним вывод команд:
```bash
docker exec -ti net_test ifconfig
docker-machine ssh docker-host ifconfig
```
Вывод команд одинаковый, так как контейнер запущен в сетевом пространстве хоста. Если запустить docker контейнер в сетевом пространстве, например reddit, то вывод команд будт разным. Контейнеру будут доступны только интерфейсы определенные для reddit - eth0 и lo.

Попробууем запустить конейтнер несколько раз (2-4 раза):
```bash
docker run --network host -d nginx
```
nginx использует порт 80. Поэтому первый запуск контейнера успешен, последующие запуски контейнера "падают", так как порт 80 уже занят. 

#### 2. Docker networks
Запустим контейнеры из пункта 1 с различными драйверами и посомтрим какие создаются namespace
Если контейнер запускаем с драйвером host, то новый namespace не созлается, используется namespace default
```bash
gcp_buzan@docker-host:/$ sudo ip netns
default
```

Если используется драйвер none (или другой отличный от host), то создается новый namespace.
```bash
gcp_buzan@docker-host:/$ sudo ip netns
75b4d1bb285c
default
```

#### 3. Bridge network driver
запустим проект в 2-х bridge сетях. Так , чтобы сервис ui не имел доступа к базе данных.
Создадим docker-сети
```bash
docker network create back_net —subnet=10.0.2.0/24
docker network create front_net --subnet=10.0.1.0/24
```
Запустим docker-контенеры в созданных сетях:
```bash
# Сервис UI
docker run -d --network=front_net -p 9292:9292 dockerbuzankirill/ui:1.0
# Сервис comment
docker run -d --network=back_net --name comment dockerbuzankirill/comment:1.0
rill/ui:1.0
# Сервис post-py
docker run -d --network=back_net --name post dockerbuzankirill/post:1.0
# Mongo
docker run -d --network=back_net --name mongo_db --network-alias=post_db --network-alias=comment_db mongo:latest
```
Для работы приложения, необходимо чтобы контейнеры comment и post находились помимо сети back_net еще и в post_end:
```bash
docker network connect front_net post
docker network connect front_net comment
```
После включения контейнеров во вторую подсеть, приложение работает штатно.

осмотрим как выглядит сетевой стек Linux в текущий момент.
Для этого проивзодим установку на host:
```bash
sudo apt-get update && sudo apt-get install bridge-utils
```
Найдем все bridge-интерфейсы для каждой из сетей:
```bash
docker-user@docker-host:~$ ifconfig | grep br
br-5502d6e485a8 Link encap:Ethernet  HWaddr 02:42:2c:5f:61:7a  
br-8f2629ea7321 Link encap:Ethernet  HWaddr 02:42:f1:68:5b:6f  
br-cf24cfaf9175 Link encap:Ethernet  HWaddr 02:42:a5:37:44:fe 
```
Посмотрим отображаемые veth-интерфейсы:
```bash
brctl show br-5502d6e485a8
1) back_net interface
br-5502d6e485a8 Link encap:Ethernet  HWaddr 02:42:2c:5f:61:7a  
          inet addr:10.0.2.1  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::42:2cff:fe5f:617a/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:74 errors:0 dropped:0 overruns:0 frame:0
          TX packets:243 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:4772 (4.7 KB)  TX bytes:23450 (23.4 KB)

brctl show br-8f2629ea7321
2) front_net interface
br-8f2629ea7321 Link encap:Ethernet  HWaddr 02:42:f1:68:5b:6f  
          inet addr:10.0.1.1  Bcast:10.0.1.255  Mask:255.255.255.0
          inet6 addr: fe80::42:f1ff:fe68:5b6f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:79 errors:0 dropped:0 overruns:0 frame:0
          TX packets:256 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:20246 (20.2 KB)  TX bytes:26585 (26.5 KB)

brctl show br-cf24cfaf9175
br-cf24cfaf9175 Link encap:Ethernet  HWaddr 02:42:a5:37:44:fe  
          inet addr:172.18.0.1  Bcast:172.18.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:a5ff:fe37:44fe/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:113 errors:0 dropped:0 overruns:0 frame:0
          TX packets:159 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:39751 (39.7 KB)  TX bytes:33296 (33.2 KB
```

#### 4. Docker-compose
Установим docker-compose. 
```bash
pip install docker-compose
```
Создадим файл docker-compose.yml
В презентации ссылка не вернка. Правильная ссылка:
https://github.com/express42/otus-snippets/blob/master/hw-17/docker-compose.yml

Зададим переменную  окружения:
export USERNAME=dockerbuzankirill

Произведем запуск docker-compose:
```bash
docker-compose up -d
```
Полчил ошибку:
```bash
Creating network "redditmicroservices_front_net" with the default driver
ERROR: cannot create network 38aa9df8ad74f5a031e5e9ebb178d2479c01bf88e58dbe4d491e619256fcf33d (br-38aa9df8ad74): conflicts with network 8f2629ea73219c84634b6c6e5d48232881518cc73ed39af43e857164dcc439a3 (br-8f2629ea7321): networks have overlapping IPv4
```
Посмотрим созданные сети:
```bash
docker network ls
NETWORK ID          NAME                         DRIVER              SCOPE
5502d6e485a8        back_net                     bridge              local
a9a0732c9e37        bridge                       bridge              local
8f2629ea7321        front_net                    bridge              local
a621d14fafdc        host                         host                local
f5726317cea5        none                         null                local
cf24cfaf9175        reddit                       bridge              local
```
Удалим конфликтные сети, ориентируясь на ID:
```bash
docker network rm back_net
docker network rm front_net
```
Теперь команла отработает корректно.
Посмотрми текущий список:
```bash
docker network ls
NETWORK ID          NAME                            DRIVER              SCOPE
a9a0732c9e37        bridge                          bridge              local
a621d14fafdc        host                            host                local
f5726317cea5        none                            null                local
cf24cfaf9175        reddit                          bridge              local
9c078820374a        redditmicroservices_back_net    bridge              local
e461abd3ef3c        redditmicroservices_front_net   bridge              local
```
Как видим появились две новые сети:
redditmicroservices_back_net
redditmicroservices_front_net 

После запуска docker-compose приложение работает штатно.

Изменим файл docker-compose.yml в сотвветствии с заданием:
1) Изменить docker-compose под кейс с множеством сетей, сетевых алиасов.
2) Параметризуйте с помощью переменных окружений:
• порт публикации сервиса ui
• версии сервисов
• возможно что-либо еще на ваше усмотрение

1) В файл добавил блок:
```yml
...
networks:
  back_net:
    ipam:
      config:
      # default network back_net 10.0.2.0/24
      - subnet: ${BACK_NET_IP}
  front_net:
    ipam:
      config:
      # default network front_net 10.0.1.0/24
      - subnet: ${FRONT_NET_IP}
```
добавил соответствующие сети для сервисов.

2) Параметезировал:
Порт публикации UI
```yml
- ${UI_PORT}:${UI_PORT}/tcp
```
Версии сервисов:
```yml
...
image: mongo:${MONGO_V}
...
image: ${USERNAME}/ui:${UI_V}
...
image: ${USERNAME}/post:${POST_V}
...
image: ${USERNAME}/comment:${COMMENT_V}
...
```
в файле комментариями указаны default значения
Дополнительно сделал параметризацию сетей
```yml
...
- subnet: ${BACK_NET_IP}
...
- subnet: ${FRONT_NET_IP}
```

3) Параметризованные параметры запишите в отдельный файл c расширением .env
4) Без использования команд source и export
docker-compose должен подхватить переменные из этого файла. Проверьте.

Ссылка на документацию docker:
https://docs.docker.com/compose/compose-file/#env_file
Файл должен иметь имя и расширение:
.env
в таком случае docker-compose его подхватит автоматически.
содержимое файла .env
```text
USERNAME=dockerbuzankirill
MONGO_V=3.2
UI_V=1.0
UI_PORT=9292
POST_V=1.0
COMMENT_V=1.0
BACK_NET_IP=10.0.2.0/24
FRONT_NET_IP=10.0.1.0/24
```

Как образуется базовое имя проекта. Можно ли его задать? Если можно то как?
Описание в документации docker:
https://docs.docker.com/compose/reference/envvars/
Можно задать с помощью тега -p или с помощью переменной окружения: COMPOSE_PROJECT_NAME

Пример:
До:
```bash
docker ps
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                    NAMES
e5087b6ca082        dockerbuzankirill/comment:1.0   "puma"                   40 minutes ago      Up 30 minutes                                redditmicroservices_comment_1docker ps
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                    NAMES
f4460f733725        dockerbuzankirill/ui:1.0        "puma"                   8 seconds ago       Up 5 seconds        0.0.0.0:9292->9292/tcp   otusms_ui_1
95982156bf72        dockerbuzankirill/comment:1.0   "puma"                   8 seconds ago       Up 6 seconds                                 otusms_comment_1
d01264309f23        mongo:3.2                       "docker-entrypoint.s…"   8 seconds ago       Up 6 seconds        27017/tcp                otusms_post_db_1
e8de7d48e10a        dockerbuzankirill/post:1.0      "python3 post_app.py"    8 seconds ago       Up 6 seconds                                 otusms_post_1
[kirill@localhost reddit-microservices]$ 

9a8c7f258d56        dockerbuzankirill/ui:1.0        "puma"                   40 minutes ago      Up 30 minutes       0.0.0.0:9292->9292/tcp   redditmicroservices_ui_1
e1dadd55710c        dockerbuzankirill/post:1.0      "python3 post_app.py"    40 minutes ago      Up 30 minutes                                redditmicroservices_post_1
ca140225cc48        mongo:3.2                       "docker-entrypoint.s…"   40 minutes ago      Up 30 minutes       27017/tcp                redditmicroservices_post_db_1
```
Выполним переопределение имени:
```bash
docker-compose -p OtusMS up -d
```
Проверим:
```bash
docker ps
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                    NAMES
f4460f733725        dockerbuzankirill/ui:1.0        "puma"                   8 seconds ago       Up 5 seconds        0.0.0.0:9292->9292/tcp   otusms_ui_1
95982156bf72        dockerbuzankirill/comment:1.0   "puma"                   8 seconds ago       Up 6 seconds                                 otusms_comment_1
d01264309f23        mongo:3.2                       "docker-entrypoint.s…"   8 seconds ago       Up 6 seconds        27017/tcp                otusms_post_db_1
e8de7d48e10a        dockerbuzankirill/post:1.0      "python3 post_app.py"    8 seconds ago       Up 6 seconds                                 otusms_post_1
```
#### 5. Docker-compose override
Создан файл: docker-compose.override.yml
Для возможности изменения кода каждого из приложений, не выполняя сборку образа, необходимо использовать volumes.

Для проверки, был удален host и создан заново. 
Далее выполнил команду, что использовать docker-compose.yml и docker-compose.override.yml использовал тег -f для явного указания файлов:
docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d

# Homework#16 Buzan Kirill
#### 1. Новая структура приложения
1) Все файлы предыдущих домашних заданий (14 и 15) перенесены в каталог monolith
2) Домашнее задание 16 находится в каталоге reddit-microservices. Приложение разбито на 3 каталога:

      - post-py - сервис отвечающий за написание постов
      - comment - сервис отвечающий за написание комментариев
      - ui - веб-интерфейс, работающий с другими сервисами
      - Отдельно создан контейнер с БД на базе MongoDB
3) Установить linter не удалось на CentOs 7, поэтому воспользовался сайтом: https://www.fromlatest.io
#### 2. Dockerfile сервисов
1) Сервис post-py
```Docker
FROM python:3.6.0-alpine
RUN pip install flask pymongo

WORKDIR /app
COPY . /app

RUN pip install -r /app/requirements.txt

ENV POST_DATABASE_HOST=post_db \
    POST_DATABASE=posts

CMD ["python3", "post_app.py"]
```
Заменено использование ADD на COPY. ENV записано в одну строку. Больше файл изменений не притерпел.
Онлайн сервис linter предложений по улучшению не показал.

2) Сервис comment
```Docker
FROM ruby:2.2
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends build-essential &&\
    rm -rf /var/lib/apt/lists/*

ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

COPY Gemfile* $APP_HOME/
RUN bundle install
COPY . $APP_HOME

ENV COMMENT_DATABASE_HOST=comment_db
ENV COMMENT_DATABASE=comments

CMD ["puma"]
```
Заменено использование ADD на COPY. Онлайн linter предложил удалять кеш rm -rf /var/lib/apt/lists/* и использовать конструкцию --no-install-recommends.
Эти изменения повелкли к изменению размера образа на 10 МБ.
Без рекомендаций linter:
```bash
[kirill@localhost reddit-microservices]$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
dockerbuzankirill/comment   1.0                 ae9e46b04522        16 minutes ago      771MB
```
После применения рекомендация linter:
```bash
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
dockerbuzankirill/comment   1.0                 bd415ce462db        9 minutes ago       761MB
```
3) Сервис UI
```Docker
FROM ruby:2.2
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*

ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME
COPY Gemfile* $APP_HOME/

RUN bundle install
COPY . $APP_HOME
ENV POST_SERVICE_HOST=post
ENV POST_SERVICE_PORT=5000
ENV COMMENT_SERVICE_HOST=comment
ENV COMMENT_SERVICE_PORT=9292
CMD ["puma"]
```
Заменено использование ADD на COPY. Онлайн linter предложил удалять кеш rm -rf /var/lib/apt/lists/* и использовать конструкцию --no-install-recommends.
Эти изменения повелкли к изменению размера образа на 11 МБ.
Без рекомендаций linter:
```bash
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
dockerbuzankirill/ui        1.0                 c4e566f81f5a        15 minutes ago      779MB
```
После применения рекомендация linter:
```bash
REPOSITORY                  TAG                 IMAGE ID            CREATED              SIZE
dockerbuzankirill/ui        1.0                 93c6eb4456a2        About a minute ago   768MB
```
#### 3. Сборка приложений
Команды для сборки приложений:
```bash
# Mongo
docker pull mongo:latest
# Сервис post-py
docker build -t dockerbuzankirill/post:1.0 ./post-py
# Сервис comment
docker build -t dockerbuzankirill/comment:1.0 ./comment
# Сервис UI
docker build -t dockerbuzankirill/ui:1.0 ./ui
```
Сборка приложения UI началалось не с первого пункта, так как docker уже успел закешировать несколько слоев при выполнении comment.

#### 4. Запуск приложений
1) Создадим специальную сеть для приложения
```bash
docker network create reddit
```
Создали bridge-сеть для контейнеров, так как сетевые алиасы не работают в сети по умолчанию.

2) Запустим контейнеры в созданные сети с алиасами к контейнерам. Команды для запуска контейнеров:
```bash
# Mongo
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
# Сервис post-py
docker run -d --network=reddit --network-alias=post dockerbuzankirill/post:1.0
# Сервис comment
docker run -d --network=reddit --network-alias=comment dockerbuzankirill/comment:1.0
# Сервис UI
docker run -d --network=reddit -p 9292:9292 dockerbuzankirill/ui:1.0
```
#### 5. Задание со звездочкой
Конмады для запусков контейнеров с новыми сетевыми алиасами. Переменные окружения задаются в команде:
```bash
# Mongo
docker run -d --network=reddit --network-alias=post_db_mydocker --network-alias=comment_db_mydocker mongo:latest
# Сервис post-py
docker run -d --network=reddit --network-alias=post_mydocker -e POST_DATABASE_HOST=post_db_mydocker dockerbuzankirill/post:1.0
# Сервис commen
docker run -d --network=reddit --network-alias=comment_mydocker -e COMMENT_DATABASE_HOST=comment_db_mydocker dockerbuzankirill/comment:1.0
# Сервис UI
docker run -d --network=reddit -p 9292:9292 -e POST_SERVICE_HOST=post_mydocker -e COMMENT_SERVICE_HOST=comment_mydocker dockerbuzankirill/ui:1.0
```
#### 6. Улучшение образа UI
Собран образ на базе Ubuntu - версия 2. Так же применены все рекомендации linter. Образ на основе ruby переименрован в Dockerfile_ruby. 
Результат:
```bush
[kirill@localhost reddit-microservices]$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
dockerbuzankirill/ui        2.0                 6e4896f7e84d        21 seconds ago      394MB
dockerbuzankirill/ui        1.0                 93c6eb4456a2        45 minutes ago      768MB
```
Образ уменьшился почти в 2 раза - 48,7%
#### 7. Задание со звездочкой. Образ на основе Alpine Linux
Собран образ на базе Apline Linix - версия 3. Так же применены все рекомендации linter. Образ на основе Ubuntu переименрован в Dockerfile_ubuntu. 
Результат:
```bush
REPOSITORY                  TAG                 IMAGE ID            CREATED              SIZE
dockerbuzankirill/ui        3.0                 7b1aa2e62c77        About a minute ago   207MB
dockerbuzankirill/ui        2.0                 6e4896f7e84d        2 hours ago          394MB
dockerbuzankirill/ui        1.0                 93c6eb4456a2        2 hours ago          768MB
```
Образ уменьшился по сравнение с версией 1 в 3.5 раза - 73%. По сравнение с версией 2 почти в 2 раза - 47,5%
Применение рекомендация linter ведет к снижению замнаиемого образа, так как чистится кеш. 

Можно сделать вывод, что основополагающим способом по уменьшению образа является использование минимального базового образа. Так же необходимо устанавливать в контейнер только необходимые пакеты и утилиты. 
Рекомендации от linter позволяют сделать вывод, что удаление временных файлов, кешей и другого мусора так же позволяют уменьшить образ.
Уменьшение количества слоев так же ведет к меньшению образа.

#### 8. Docker Volume
1) Создадим docker volume
```bas
docker volume create reddit_db
```
2) Запустим контейнеры, используя volume
```bash
# Mongo
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
# Сервис post-py
docker run -d --network=reddit --network-alias=post dockerbuzankirill/post:1.0
# Сервис comment
docker run -d --network=reddit --network-alias=comment dockerbuzankirill/comment:1.0
# Сервис UI
docker run -d --network=reddit -p 9292:9292 dockerbuzankirill/ui:2.0
```
Запуск контейнеров с volume позволило удалять и запускать контейнер без потери данных в базе данных. 

# Homework#15 Buzan Kirill
#### 1. Установка docker-machine
```bash
curl -L https://github.com/docker/machine/releases/download/v0.13.0/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine && \
> sudo install /tmp/docker-machine /usr/local/bin/docker-machine
```
Проверяем установленную версию:
```bash
docker-machine version

docker-machine version 0.13.0, build 9ba6da9
```

#### 2. Новый проект Docker в GCE
Создано новый проект Docker в GCE. Произведена настройка и подключения gcloud к проекту Docker

#### 3. Создаем интанс в GCE с помощью docker-machine
```bash
docker-machine create --driver google \
--google-project docker-myid \
--google-zone europe-west1-b \
--google-machine-type g1-small \
--google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
docker-host
```
В результате будет создан инстанс в GCE.
Проверим, что он создан успешно:
```bash
docker-machine ls

NAME          ACTIVE   DRIVER   STATE     URL                         SWARM   DOCKER        ERRORS
docker-host   -        google   Running   tcp://35.189.195.131:2376           v18.02.0-ce  
```
Мы видим что инстанс запущен, но в колонке active, мы видим - , это значит что переменные окружения не настроены. 
Для настройки переменных окружений в автоматическом режиме выполним команду:
```bash
#Посмотрим переменные окружения, необходимые для работы с docker хостом:
[kirill@localhost 15_docker_2]$ docker-machine env docker-host
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://35.189.195.131:2376"
export DOCKER_CERT_PATH="/home/kirill/.docker/machine/machines/docker-host"
export DOCKER_MACHINE_NAME="docker-host"
# Run this command to configure your shell: 
# eval $(docker-machine env docker-host)

# Выполним команду для применения переменных, укаазнных выше, в текущем окрудении клиента
[kirill@localhost 15_docker_2]$ eval $(docker-machine env docker-host)

```
Повторим команду docker-machine ls:
```bash
[kirill@localhost 15_docker_2]$ docker-machine ls
NAME          ACTIVE   DRIVER   STATE     URL                         SWARM   DOCKER        ERRORS
docker-host   *        google   Running   tcp://35.189.195.131:2376           v18.02.0-ce   
```
В столбце active появилась * это значит, данный dcoker host в настоящий момент активен и при запуске утидиты dcoker все ее команды будут выполняться на этом хосте.

#### 4. Namespaces
Запуск контейнера (tehbilly/htop) используя изоляцию - свое пространство имен. Видны только процессы внутри контейнера. Главный процесс PID=1.
```bash
docker run --rm -ti tehbilly/htop
```
Запуск контейнера (tehbilly/htop). При это контейнер делит пространство имен порцессов на хосте, где он запущен. Таким образом позволяя процессам внутри контейнер видеть процессы в системе. 
```bash
docker run --rm --pid host -ti tehbilly/htop
```

#### 5. Создание образа с приложением reddit на основе ubuntu 16.04
1. Созданы файлы, необходимые для работы приложения и конфигурации mongo.
db_config - задается переменная окружения DATABASE_URL
mongod.conf - конфигурационный файл для Mongo
start.sh - скрипт для запуска приложения reddit
Dockerfile - файл для сборки docker образа
2. Сборка образа. 
```bash
docker build -t reddit:latest .

Successfully built 760f90752dcf
Successfully tagged reddit:latest
```
Указываем тег при создании образа. Теги полезны, так как не маркированные тегами образы будут попадать в список подвешенных образов. Это слои, которые не имеют связей с другими маркированными образами.
Узнать список подвешенных образов можно командой:
```bash
docker images -f dangling=true
```
3. Вывод списка доступных образов
```bash
docker images -a

[kirill@localhost 15_docker_2]$ docker images -a
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
reddit              latest              760f90752dcf        9 minutes ago       682MB
<none>              <none>              fa12b1150ff7        9 minutes ago       682MB
...
...
ubuntu              16.04               0458a4468cbc        3 weeks ago         112MB
tehbilly/htop       latest              692db5793206        5 weeks ago         6.85MB
```
#### 6. Запуск docker контейнера
1. Запуск произведем командой:
```bash
docker run --name reddit -d --network=host reddit:latest

4074399af177afd2a1fe8a049ad6890a82a43800a86889248258fc4809e4fe1d
```
*--name reddit* задает имя контейнера. Можно выполнять работа с контейнером используя его имя, а не container_id. Если имя не задать, docker сам сгенерирует время
*-d* запуск контейнера в автономном режиме (открепит терминал)
*--network=host* контейнер будет использовать сетевой стек хоста. 
*reddit:latest* в конце задано имя образа и тег на осонове которого будет запущен контейнер.
2. Проверяем, что конейнер запущен
```bash
docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
4074399af177        reddit:latest       "/start.sh"         About a minute ago   Up About a minute   
```

#### 7. Настройка firewall
Без разрешения входящего TCP-трафика на порт 9292 нет возможности использовать web-приложение.
Разрешим входящий TCP-трафик.
```bash
gcloud compute firewall-rules create reddit-app \
--allow tcp:9292 --priority=65534 \
--target-tags=docker-machine \
--description="Allow TCP connections" \
--direction=INGRESS

Creating firewall...done.                                                                                                                   
NAME        NETWORK  DIRECTION  PRIORITY  ALLOW     DENY
reddit-app  default  INGRESS    65534     tcp:9292
```
В консоли GCE в правилах брандмауэра добавилось новое правиль reddit-app

После создания правила, web-приложение успешно заработало.

#### 8. Docker Hub
1. Произвел регистрацию на docker hub
2. Осуществил подклювчение к docker hub в консоли:
```bash
docker login

Login Succeeded
```
3. Производим загрузку образа на docker hub:
```bash
docker tag reddit:latest dockerbuzankirill/otus-reddit:1.0
docker push dockerbuzankirill/otus-reddit:1.0

The push refers to repository [docker.io/dockerbuzankirill/otus-reddit]
c81510f35bf3: Pushed 
f16c607d4b69: Pushed 
cc7ec20b9272: Pushed 
9ea6e05198e2: Pushed 
9cbee2a4dce8: Pushed 
1ad4dbbc42ab: Pushed 
256a0e97f2da: Pushed 
34b2d7663eeb: Pushed 
b770752480de: Pushed 
6f4ce6b88849: Mounted from library/ubuntu 
92914665e7f6: Mounted from library/ubuntu 
c98ef191df4b: Mounted from library/ubuntu 
9c7183e0ea88: Mounted from library/ubuntu 
ff986b10a018: Mounted from library/ubuntu 
1.0: digest: sha256:3ef2444594c99d1ef6b649200bd2bb648b51f68876306fd5c0851bbd604f2e88 size: 3241
```
Образ успешно загружен на docker hub:
https://hub.docker.com/r/dockerbuzankirill/otus-reddit/tags/
Во вкладке tags прописано имя тега 1.0, то что мы указали после названия репозитория.


# Homework#14 Buzan Kirill
#### 1. Установка Docker. Centos 7
Ссылка на инструкцию:
https://docs.docker.com/install/linux/docker-ce/centos/

1. Install required packages
```bash
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```
2. Set up the stable repository
```bash
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
3. Install the latest version of Docker CE
```bash
sudo yum install docker-ce
```
4. Start Docker
```bash
sudo systemctl start docker
```
5. Status Docker
```bash
sudo systemctl status docker

● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2018-02-18 20:52:28 +07; 15h ago
     Docs: https://docs.docker.com
 Main PID: 4570 (dockerd)
   Memory: 44.3M
   CGroup: /system.slice/docker.service
           ├─4570 /usr/bin/dockerd
           ├─4579 docker-containerd --config /var/run/docker/containerd/conta...
           └─5731 docker-containerd-shim -namespace moby -workdir /var/lib/do...
```

#### 2. Проверка устанвленного Docker
1. Версия Docker сервера и клиента
```bash
sudo docker version

Client:
 Version:	17.12.0-ce
 API version:	1.35
 Go version:	go1.9.2
 Git commit:	c97c6d6
 Built:	Wed Dec 27 20:10:14 2017
 OS/Arch:	linux/amd64

Server:
 Engine:
  Version:	17.12.0-ce
  API version:	1.35 (minimum version 1.12)
  Go version:	go1.9.2
  Git commit:	c97c6d6
  Built:	Wed Dec 27 20:12:46 2017
  OS/Arch:	linux/amd64
  Experimental:	false
```
2. Состояние docker daemon
```bash
sudo docker info

Containers: 2
 Running: 1
 Paused: 0
 Stopped: 1
Images: 4
Server Version: 17.12.0-ce
...
```
3. Запуск проверочного контейнера
```bash
sudo docker run hello-world
```
Произошло скачивание образа hello-world с Docker Hub. Далее на основе образа был создан контейнер и на экран выведен поток stdout. В котором говорится об успешноЙ работе контейнера. 
Hello from Docker!
This message shows that your installation appears to be working correctly.

#### 4. Список контейнеров
1. Список всех запущенных контейнеров
```bash
sudo docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
044ed99ec125        ubuntu:16.04        "/bin/bash"         16 hours ago        Up 16 hours                             focused_mestorf
 ```
 2. Список всех контейнеров
 ```bash
 sudo docker ps -a
 
 CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                           PORTS               NAMES
631bd42d65af        hello-world         "/hello"                 3 minutes ago       Exited (0) 3 minutes ago                             heuristic_williams
5c84de9ec43b        nginx:latest        "nginx -g 'daemon of…"   16 hours ago        Exited (137) About an hour ago                       elastic_jennings
044ed99ec125        ubuntu:16.04        "/bin/bash"              16 hours ago        Up 16 hours                                          focused_mestorf
 ```
 
 #### 5. Список сохранненых образов
 ```bash
 sudo docker images
 
 REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
buzankirill/ubuntu-tmp-file   latest              04635fd88b98        16 hours ago        112MB
nginx                         latest              9e988ed19567        45 hours ago        109MB
ubuntu                        16.04               0458a4468cbc        3 weeks ago         112MB
hello-world                   latest              f2a91732366c        3 months ago        1.85kB
 ```
 
 #### 6. Docker RUN. 
 Команда docker run создает и запускает каждый раз новый контейнер из образа. 
 ```bash
[kirill@localhost 14_docker_1]$ sudo docker run -it ubuntu:16.04 bash
root@5ef269e9563b:/# echo 'Hello world!' > /tmp/file
root@5ef269e9563b:/# exit
exit
[kirill@localhost 14_docker_1]$ sudo docker run -it ubuntu:16.04 bash
root@a12f8c53d358:/# cat /tmp/file
cat: /tmp/file: No such file or directory
root@a12f8c53d358:/# 
```
Убедились что созданный в 1м контейнере файл, отсутствует в контейнере 2.
команда -it
-i запускает контейнер в foreground режиме (docker attach)
-t создает TTY

#### 7. Docker ps
``` bash
# Команда, позволяет использовать форматирование при выводе инфмаорции на экран. 
[kirill@localhost 14_docker_1]$ sudo docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.CreatedAt}}\t{{.Names}}"
CONTAINER ID        IMAGE               CREATED AT                      NAMES
a12f8c53d358        ubuntu:16.04        2018-02-19 13:00:31 +0700 +07   peaceful_clarke
5ef269e9563b        ubuntu:16.04        2018-02-19 12:59:55 +0700 +07   infallible_edison
631bd42d65af        hello-world         2018-02-19 12:52:12 +0700 +07   heuristic_williams
5c84de9ec43b        nginx:latest        2018-02-18 21:05:20 +0700 +07   elastic_jennings
044ed99ec125        ubuntu:16.04        2018-02-18 20:57:31 +0700 +07   focused_mestorf
```
#### 8. Docker start && run
Docker run - это выполнение сразу 3х команд:  create, start и attach
Docjer start запускает уже созданный остановленный контейнер. run создает, запускает и присоединяется к контейнеру.
Конмада exit, которую мы выолнили в пунтке 6, остановила контейнер. Теперь нам нужно его запустить.
Узнаем container_id из пункта 7. и вызываем start:
```bash
[kirill@localhost 14_docker_1]$ sudo docker start 5ef269e9563b
5ef269e9563b
```
Автоматического подключения к контейнеру не произошо. 

#### 9. Docker attach
Команда необходима для подключения к запущенному репозиторию.
```bash
[kirill@localhost 14_docker_1]$ sudo docker attach 5ef269e9563b
root@5ef269e9563b:/# 
root@5ef269e9563b:/# 
``` 
После введения команды, необходимо нажать ENTER
Чтобы выйти из терминала и оставить контейнер в рабочем статусе, нажимаем ctrl+p, ctrl+q

#### 10. Docker exec
Позволяет запускать процессы внутри контейнера
Запустим bash внутри контейнера
```bash
[kirill@localhost 14_docker_1]$ sudo docker exec -it 5ef269e9563b bash
root@5ef269e9563b:/# ps axf
  PID TTY      STAT   TIME COMMAND
   10 pts/1    Ss     0:00 bash
   18 pts/1    R+     0:00  \_ ps axf
    1 pts/0    Ss+    0:00 bash
root@5ef269e9563b:/# 
```
#### 11. Docker commit
Позволяет создать образ из контейнера
```bash
[kirill@localhost 14_docker_1]$ sudo docker commit 044ed99ec125 buzankirill/ubuntu-tmp-file 
sha256:9ad3d9dfc6aba0bc976ed1778ac783a371daf764809bc36fe56a3138c6e19f17
```
Вывод конмады sudo docker images находит в файле docker-1.log

### 12. Задание со звездочкой. 
Файл docker-1.log
#### 13. Docker kill
Позволяет безусловно завершить процесс контейнера
```bash
[kirill@localhost 14_docker_1]$ sudo docker ps -q
5ef269e9563b
044ed99ec125

[kirill@localhost 14_docker_1]$ sudo docker kill $(sudo docker ps -q)
5ef269e9563b
044ed99ec12

[kirill@localhost 14_docker_1]$ sudo docker ps -q
[kirill@localhost 14_docker_1]$ 
```
Все запущенные контейнеры были остановлены.

#### 14. Docker system df
1. Отображает сколько дискового пространства занято образами, контейнерами и volume’ами.
2. Отображает сколько из них не используется и возможно удалить
```bash
[kirill@localhost 14_docker_1]$ sudo docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              5                   3                   220.2MB             111.7MB (50%)
Containers          5                   0                   151B                151B (100%)
Local Volumes       0                   0                   0B                  0B
Build Cache 
```
#### 15. Docker rm и rmi
Docker rm - позволяет удалить контейнер. Если указать флаг -f, то позволит удалить и запущенный контейнер, предварительно остановив контейнер.
Удалим все незапущенные контейнеры:
```bash
[kirill@localhost 14_docker_1]$ sudo docker rm $(sudo docker ps -a -q)
a12f8c53d358
5ef269e9563b
631bd42d65af
5c84de9ec43b
044ed99ec125
# Проверяем наличие контейнеров
[kirill@localhost 14_docker_1]$ ps -a
  PID TTY          TIME CMD
 7561 pts/0    00:00:00 ps
```

Docker rmi позволяет удалить образы.
Удалим все образы:
```bash
# выводим спиок всех образов до удаления
[kirill@localhost 14_docker_1]$ sudo docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
buzankirill/ubuntu-tmp-file   latest              9ad3d9dfc6ab        12 minutes ago      112MB
<none>                        <none>              04635fd88b98        16 hours ago        112MB
nginx                         latest              9e988ed19567        46 hours ago        109MB
ubuntu                        16.04               0458a4468cbc        3 weeks ago         112MB
hello-world                   latest              f2a91732366c        3 months ago        1.85kB

# Удаляем все образы
[kirill@localhost 14_docker_1]$ sudo docker rmi $(sudo docker images -q)
Untagged: buzankirill/ubuntu-tmp-file:latest
Deleted: sha256:9ad3d9dfc6aba0bc976ed1778ac783a371daf764809bc36fe56a3138c6e19f17
Deleted: sha256:04635fd88b9858cc55e5988ecb6e3237b2e5d8d095ffe922811e4ee95ad45952
Deleted: sha256:a891fc083d3d2da1af222e245b10d21f5107ae94af81e77c670d0c9637861cf1
Untagged: nginx:latest
Untagged: nginx@sha256:0ffc09487404ea43807a1fd9e33d9e924d2c8b48a7b7897e4d1231a396052ff9
Deleted: sha256:9e988ed19567262bd07d62a8e51b3c0db8bda5d7afd63268860aabfe04d46168
Deleted: sha256:5f9755b3f59c5674da3f1e95d21253288c8984012e06449373e334d254710d94
Deleted: sha256:d6569cc491707f1b0312811dfe4ca03dd12f250c5078ebaa1ec31a0d69fa8fa0
Deleted: sha256:014cf8bfcb2d50b7b519c4714ac716cda5b660eae34189593ad880dc72ba4526
Untagged: ubuntu:16.04
Untagged: ubuntu@sha256:e27e9d7f7f28d67aa9e2d7540bdc2b33254b452ee8e60f388875e5b7d9b2b696
Deleted: sha256:0458a4468cbceea0c304de953305b059803f67693bad463dcbe7cce2c91ba670
Deleted: sha256:77e6ddba346d8ad1e436256f6373dede5af4002006981b7d4116c561c759cefa
Deleted: sha256:8db758ab2fdb54da0aec53aeac876934337e6170f5a8c8872b3d4171e3d465b7
Deleted: sha256:a7fc6b405fe8ef71edfa6163d1dc9f1cb1df426049eefaa7d388e9df21a061ad
Deleted: sha256:5a3e35538f7f2e2727c8ac92f08c30002b9e8a77737de0dab91244344d59f69b
Deleted: sha256:ff986b10a018b48074e6d3a68b39aad8ccc002cdad912d4148c0f92b3729323e
Untagged: hello-world:latest
Untagged: hello-world@sha256:083de497cff944f969d8499ab94f07134c50bcf5e6b9559b27182d3fa80ce3f7
Deleted: sha256:f2a91732366c0332ccd7afd2a5c4ff2b9af81f549370f7a19acd460f87686bc7
Deleted: sha256:f999ae22f308fea973e5a25b57699b5daf6b0f1150ac2a5c2ea9d7fecee50fdf

# Выводим список образов
[kirill@localhost 14_docker_1]$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
[kirill@localhost 14_docker_1]$ 
```
