# Домашнее задание
## Практика с SELinux
Цель: Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
#### 1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
К сдаче:
- README с описанием каждого решения (скриншоты и демонстрация приветствуются).

#### 2. Обеспечить работоспособность приложения при включенном selinux.
- Развернуть приложенный стенд
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
- Выяснить причину неработоспособности механизма обновления зоны (см. README);
- Предложить решение (или решения) для данной проблемы;
- Выбрать одно из решений для реализации, предварительно обосновав выбор;
- Реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.


#### 1. Запустить nginx на нестандартном порту 3-мя разными способами:

##### Переключатели setsebool:

Установить веб-сервер nginx:
```
sudo yum install -y nginx
```
В файле /etc/nginx/nginx.conf изменить порт на 26262, однако, на таком порту сервис не запустится:
```
...
 server {
        listen       26262 default_server;
        listen       [::]:26262 default_server;
       ...
```
Создать разрешающее правило:
```
sudo audit2why < /var/log/audit/audit.log
sudo setsebool -P nis_enabled 1
```
Запустить сервис командой:
```
sudo systemctl start nginx.service
```
Посмотреть на каком порту активен nginx:
```
[root@localhost vagrant]# netstat -tnlup
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:26262           0.0.0.0:*               LISTEN      5955/nginx: master
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      682/sshd
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      912/master
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      402/rpcbind
tcp6       0      0 :::26262                :::*                    LISTEN      5955/nginx: master
tcp6       0      0 :::22                   :::*                    LISTEN      682/sshd
tcp6       0      0 ::1:25                  :::*                    LISTEN      912/master
tcp6       0      0 :::111                  :::*                    LISTEN      402/rpcbind
udp        0      0 0.0.0.0:991             0.0.0.0:*                           402/rpcbind
udp        0      0 127.0.0.1:323           0.0.0.0:*                           384/chronyd
udp        0      0 0.0.0.0:68              0.0.0.0:*                           470/dhclient
udp        0      0 0.0.0.0:111             0.0.0.0:*                           402/rpcbind
udp6       0      0 :::991                  :::*                                402/rpcbind
udp6       0      0 ::1:323                 :::*                                384/chronyd
udp6       0      0 :::111                  :::*                                402/rpcbind

```

##### Добавление нестандартного порта в имеющийся тип:
Посмотреть какие порты могут использоваться по протоколу http командой:
```
[root@localhost vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

```
Как видно, порт 26262 отсутствует
Добавить порт можно командой:
```
sudo semanage port -a -t http_port_t -p tcp 26262

[root@localhost vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      26262, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Удалить командой:
```
sudo semanage port -d -t http_port_t -p tcp 12345
```

##### Формирование и установка модуля SELinux

Сформировать модуль посредством лог-файла:
```
sudo audit2allow -M httpd_add --debug < /var/log/audit/audit.log
```
```
[root@localhost vagrant]# sudo audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:
semodule -i httpd_add.pp
```
Установить сформированный модуль командой:
```
sudo semodule -i httpd_add.pp
```
Проверить, корректно ли установился модуль:
```
[root@localhost vagrant]# semodule -l | grep http
httpd_add       1.0
```

sudo semodule -e -v httpd_add    - включает модуль

sudo semodule -d -v httpd_add    - выключает модуль

sudo semodule -r httpd_add  - удаляет модуль

#### 2. Обеспечить работоспособность приложения при включенном selinux


