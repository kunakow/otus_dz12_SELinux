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

Запускаем стенд:
```
vagrant up
```
Подключаемся к клиентской машине:
```
vagrant ssh client
```
Актуализируем проблему командой:
```
nsupdate -k /etc/named.zonetransfer.key
server 192.168.50.10
zone ddns.lab 
update add www.ddns.lab. 60 A 192.168.50.15
send
```
Получаем ошибку:
```
update failed: SERVFAIL
```
На сервере определяем причину запрета доступа из лог-файла:
```
[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1614802421.411:1813): avc:  denied  { create } for  pid=4925 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1614802721.058:1814): avc:  denied  { create } for  pid=4925 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.


```
Сформируем и установим модуль:
```
sudo audit2allow -M otusselinux --debug < /var/log/audit/audit.log
sudo semodule -i otusselinux.pp
```
Посмотрим лог:
```
[root@ns01 vagrant]# cat /var/log/messages | grep ausearch
Mar  3 20:13:45 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
Разрешим isc-worker000 доступ к записи named.ddns.lab.view1.jnl:
```
ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000 | semodule -i my-iscworker0000.pp
```
Проверяем статус сервиса:
```
[root@ns01 vagrant]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-03-03 21:34:09 UTC; 2min 20s ago
  Process: 5069 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 5082 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 5080 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 5084 (named)
   CGroup: /system.slice/named.service
           └─5084 /usr/sbin/named -u named -c /etc/named.conf

Mar 03 21:34:09 ns01 named[5084]: network unreachable resolving './NS/IN': 2001:7fd::1#53
Mar 03 21:34:09 ns01 named[5084]: managed-keys-zone/view1: Key 20326 for zone . acceptance timer complete: key now trusted
Mar 03 21:34:09 ns01 named[5084]: resolver priming query complete
Mar 03 21:34:10 ns01 named[5084]: managed-keys-zone/default: Key 20326 for zone . acceptance timer complete: key now trusted
Mar 03 21:34:10 ns01 named[5084]: resolver priming query complete
Mar 03 21:34:21 ns01 named[5084]: client @0x7fb0b403c3e0 192.168.50.15#8450/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Mar 03 21:35:21 ns01 named[5084]: client @0x7fb0b403c3e0 192.168.50.15#24150/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Mar 03 21:35:21 ns01 named[5084]: client @0x7fb0b403c3e0 192.168.50.15#24150/key zonetransfer.key: view view1: updating zone 'ddns.la...168.50.15
Mar 03 21:35:21 ns01 named[5084]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
Mar 03 21:35:21 ns01 named[5084]: client @0x7fb0b403c3e0 192.168.50.15#24150/key zonetransfer.key: view view1: updating zone 'ddns.la...ted error
Hint: Some lines were ellipsized, use -l to show in full.
```
Обратим внимание на сообщение об ошибке permission denied

Удаляем файл /etc/named/dynamic/named.ddns.lab.view1.jnl, перезапускаем сервис и проверяем статус:
```
[root@ns01 vagrant]# systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2021-03-03 21:42:42 UTC; 8min ago
  Process: 5267 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 5280 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 5278 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 5282 (named)
   CGroup: /system.slice/named.service
           └─5282 /usr/sbin/named -u named -c /etc/named.conf

Mar 03 21:42:42 ns01 named[5282]: zone localhost/IN/view1: loaded serial 0
Mar 03 21:42:42 ns01 named[5282]: zone 0.in-addr.arpa/IN/default: loaded serial 0
Mar 03 21:42:42 ns01 named[5282]: zone newdns.lab/IN/default: loaded serial 2711201409
Mar 03 21:42:42 ns01 named[5282]: zone 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa/IN/default: loaded serial 0
Mar 03 21:42:42 ns01 named[5282]: zone 50.168.192.in-addr.arpa/IN/default: loaded serial 2711201408
Mar 03 21:42:42 ns01 named[5282]: zone dns.lab/IN/default: loaded serial 2711201408
Mar 03 21:42:42 ns01 named[5282]: zone ddns.lab/IN/default: loaded serial 2711201407
Mar 03 21:42:42 ns01 named[5282]: zone localhost.localdomain/IN/default: loaded serial 0
Mar 03 21:43:38 ns01 named[5282]: client @0x7fe0dc03c3e0 192.168.50.15#26254/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
Mar 03 21:43:38 ns01 named[5282]: client @0x7fe0dc03c3e0 192.168.50.15#26254/key zonetransfer.key: view view1: updating zone 'ddns.la...168.50.15
Hint: Some lines were ellipsized, use -l to show in full.
```
С клиента:
```
[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> 
```

Итог:
SELinux блокировал доступ к файлу динамического обновления DNS сервера. 

Ещё одно решение нашёл здесь:
[ссылка](https://www.linuxquestions.org/questions/linux-server-73/nsupdate-not-working-servfail-4175420637/?__cf_chl_jschl_tk__=3733fb6de4a8a9c8c55a473c9c4d9e41bf46c00b-1614803284-0-AespMeM2KDsxJwl9u8JZB04lnKfxxhghdHUe3vxXMgDOurkL-Wp0GTqy8sby4Q55o3GnKyXkV_JIICEN-ggur9jrDUdKXbG-5tRuunjmdpNzt4AKXHEH_-A1uAq57I7MWvhZZzZQqc3Bb-XzJip5F5yES1T_lmC1KorX6Le1ey2smc6VvfFaMUOR93bQCkeBih7GhHOTT210Xue_SenbynK5LAqA33orniyAAZAdFLKMxgsLVhwy4XGzYY2ieDrrtCHyvfqaYRh0V8LZoOf_wHUL-rtDjA08DPG0dg-THUHyXxsge8YVFV6au5Typ0PLpo4iStHhsKymIO2u4qP-d3sE9b-zOlR3doDFlSGTsLwX55ENwPiK0aY6plZA34joa-l_BWwZMQaaNGilZFrkGl_W3rKwfBWU9K3vOELN-Z_y-3fr3c6TWDxRTnJ-FDa39m41NuXwzqpZ1Cfj6oTncYo)




