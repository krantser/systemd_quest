
## **Инициализация системы. Systemd.** 

### 1. Написать сервис мониторинга лог-файла.

Напишем сервис, который будет мониторить определённый лог-файл раз в
30 секунд. Сперва создадим файл конфигурации для сервиса:

```sh
sudo vi /etc/sysconfig/watchlog
```

Со следующим содержимым:

```
# Conf for watchlog service
WORD="yum"
LOG=/var/log/messages
```

Далее создадим шелл-скрипт:

```sh
sudo vi /opt/watchlog.sh
```

Со следующим содержимым:

```sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
    echo "$DATE: Word was found..." >> /var/log/watch.log
else
    exit 0
fi
```

Теперь создадим юнит сервиса:

```sh
sudo vi /usr/lib/systemd/system/watchlog.service
```

С содержимым:

```
[Unit]
Description=WatchLog Service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создадим юнит для таймера:

```sh
sudo vi /usr/lib/systemd/system/watchlog.timer
```

С содержимым:

```
[Unit]
Description=Run WatchLog every 30 second

[Timer]
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

Теперь запустим команду yum для того, что бы в нужном нам логе появились записи
о работе пакетного менеджера, на имя которого и будет реагировать наш сервис:

```sh
sudo yum update
```

Далее запустим наш сервис:

```sh
sudo systemctl start watchlog.timer
```

Затем проверяем лог-файл нашего сервиса:

```sh
tail /var/log/watch.log
```

По содержимому лог-файла видим, что сервис рабоает:

```
Tue Sep 29 16:22:35 UTC 2020: Word was found...
Tue Sep 29 16:23:58 UTC 2020: Word was found...
```

### 2. Переисать spawn-fcgi init-скрипт на unit-файл.

Из epel установим spawn-fcgi и переишем инит-скрипт на юнит. Где имя сервиса
будет называться аналогично.

Для начала установим необходимые пакеты:

```sh
sudo yum install epel-release -y && sudo yum install spawn-fcgi php php-cli \
mod_fcgid httpd -y
```
Просмотрим файл, который будем переписывать:

```sh
less /etc/rc.d/init.d/spawn-fcgi
```

Далее отредактируем файл настроек:

```sh
sudo vi /etc/sysconfig/spawn-fcgi
```

И приведём его к следующему виду:

```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"

```

Создадим юнит-файл:

```sh
sudo vi /etc/systemd/system/spawn-fcgi.service
```

Со следующим содержимым:

```
[Unit]
Description=Spawn-fcgi Startup Service
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Затем выполним команду, перезагрузив настройки systemd:

```sh
sudo systemctl daemon-reload
```

И просмотрим статус нашего сервиса:

```sh
sudo systemctl status spawn-fcgi
```

Вывод команды:
```
● spawn-fcgi.service - Spawn-fcgi Startup Service
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
```

Запустим сервис:

```sh
sudo systemctl start spawn-fcgi
```

И снова проверим статус:

```sh
sudo systemctl status spawn-fcgi
```

Как видим из краткого вывода команды, запуск прошёл успешно:

```
● spawn-fcgi.service - Spawn-fcgi Startup Service
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-09-29 20:02:14 UTC; 2s ago
 Main PID: 13831 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─13831 /usr/bin/php-cgi
           ├─13832 /usr/bin/php-cgi
           ├─13833 /usr/bin/php-cgi
```

### 3. Запуск нескольких экземпляров сервера Apach.

Дополним юнит-файл сервиса httpd возможностью запускать несколько экземпляров
сервера с разными конфигурациями.

Для начала отредактируем файл веб-сервера:

```sh
sudo vi /usr/lib/systemd/system/httpd.service
```

И приведём его к виду:

```
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%i
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Далее создаём файлы окружения, где указываем пути к конфигурациям экземпляров
веб-сервера:

```sh
sudo vi /etc/sysconfig/httpd-first
sudo vi /etc/sysconfig/httpd-second
```

Содержимое файлов:

```
OPTIONS=-f conf/first.conf
```

```
OPTIONS=-f conf/second.conf
```

Далее скопируем стандартную рабочую конфигурацию Apach:

```sh
sudo cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf
sudo cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf
```

И отредактируем получившиеся конфигурации добавив в first.conf следующие строки:

```
Listen 8080
ServerName localhost
PidFile /var/run/httpd-first.pid
```

А в файл second.conf соответственно:

```
Listen 8081
ServerName localhost
PidFile /var/run/httpd-second.pid
```

Теперь переименуем юнит-файл веб-сервера, что бы можно было запускать экземпляры:

```sh
sudo mv /usr/lib/systemd/system/httpd.service /usr/lib/systemd/system/httpd-@.service
```

Перезагрузим настройки systemd:

```sh
sudo systemctl daemon-reload 
```

Далее запустим экземпляры веб-сервера:

```sh
sudo systemctl start httpd-@first.service
sudo systemctl start httpd-@second.service
```

И проверим состояние первого демона:

```sh
sudo systemctl status httpd-@first.service
```

Вывод команды:

```
● httpd-@first.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd-@.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-10-07 12:43:42 UTC; 31s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 4035 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd\x2d.slice/httpd-@first.service
           ├─4035 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4036 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4037 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4038 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4039 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─4040 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
```

Проверим состояние второго демона:

```sh
sudo systemctl status httpd-@second.service
```

Вывод команды:

```
● httpd-@second.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd-@.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-10-07 12:43:58 UTC; 23s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 4049 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd\x2d.slice/httpd-@second.service
           ├─4049 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4050 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4051 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4052 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4053 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─4054 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
```

Как видно, экземпляры сервера работают. На данном этапе работа завершена.
