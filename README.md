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
