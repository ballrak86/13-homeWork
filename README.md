# Домашнее задание №1

## Описание файлов в директории
logFileFull.log - полный лог выполнения  
used_commands.txt - команды которые использовал  

screenshot - сделал несколько скришотов т.к. это требовалось. Но я все описал ниже, и мне кажется что они не нужны.  

## Описание проделанной работы
Поднял ВМ с установленным nginx. В конфигурационном файле сменил прослушиваемый порт (listen 5080).  
Перезапустил демон nginx
```
[root@nginx ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@nginx ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2021-09-30 12:18:42 UTC; 9s ago
  Process: 4493 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 4492 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 4096 (code=exited, status=0/SUCCESS)

Sep 30 12:18:42 nginx systemd[1]: Starting The nginx HTTP and reverse proxy server...
Sep 30 12:18:42 nginx nginx[4493]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Sep 30 12:18:42 nginx nginx[4493]: nginx: [emerg] bind() to 0.0.0.0:5080 failed (13: Permission denied)
Sep 30 12:18:42 nginx nginx[4493]: nginx: configuration file /etc/nginx/nginx.conf test failed
Sep 30 12:18:42 nginx systemd[1]: nginx.service: control process exited, code=exited status=1
Sep 30 12:18:42 nginx systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Sep 30 12:18:42 nginx systemd[1]: Unit nginx.service entered failed state.
Sep 30 12:18:42 nginx systemd[1]: nginx.service failed.
```
Перенес строки с ошибкой из файла /var/log/audit/audit.log в файл /tmp/1.log. И обработал его коммандой sealert
```
[root@nginx ~]# sealert -a /tmp/1.log
100% done
found 1 alerts in /tmp/1.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 5080.

*****  Модуль bind_ports предлагает (точность 92.2)  *************************

Если вы хотите разрешить /usr/sbin/nginx для привязки к сетевому порту $PORT_ЧИСЛО
То you need to modify the port type.
Сделать
# semanage port -a -t PORT_TYPE -p tcp 5080
    где PORT_TYPE может принимать значения: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Модуль catchall_boolean предлагает (точность 7.83)  *******************

Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

Сделать
setsebool -P nis_enabled 1

*****  Модуль catchall предлагает (точность 1.41)  ***************************

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 5080 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:httpd_t:s0
Целевой контекст              system_u:object_r:unreserved_port_t:s0
Целевые объекты               port 5080 [ tcp_socket ]
Источник                      nginx
Путь к источнику              /usr/sbin/nginx
Порт                          5080
Узел                          <Unknown>
Исходные пакеты RPM           nginx-1.20.1-2.el7.x86_64
Целевые пакеты RPM
Пакет регламента              selinux-policy-3.13.1-266.el7.noarch selinux-
                              policy-3.13.1-268.el7_9.2.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      nginx
Платформа                     Linux nginx 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           1
Впервые обнаружено            2021-09-30 12:38:10 UTC
В последний раз               2021-09-30 12:38:10 UTC
Локальный ID                  8473b7d8-14c0-4d00-9348-f93b672eefb5

Построчный вывод сообщений аудита
type=AVC msg=audit(1633005490.487:1327): avc:  denied  { name_bind } for  pid=5398 comm="nginx" src=5080 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1633005490.487:1327): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=562a1fd64b18 a2=10 a3=7ffde11166e0 items=0 ppid=1 pid=5398 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,unreserved_port_t,tcp_socket,name_bind
```
Выполним каждое решение попорядку
### 1. Добавление нестандартного порта в имеющийся тип
Определим тип на котором работает nginx и выполним semanage port, это нам позволит использовать нестандартный порт
```
[root@nginx ~]# which nginx
/sbin/nginx
[root@nginx ~]# ls -Z /sbin/nginx
-rwxr-xr-x. root root system_u:object_r:httpd_exec_t:s0 /sbin/nginx
[root@nginx ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@nginx ~]# semanage port -a -t http_port_t -p tcp 5080
```
Проверим что все у нас работает. После этого удалим добавленный порт и перезапустим nginx
```
[root@nginx ~]# systemctl start nginx
[root@nginx ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Чт 2021-09-30 12:54:18 UTC; 7s ago
  Process: 5622 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 5619 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 5618 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 5624 (nginx)
   CGroup: /system.slice/nginx.service
           ├─5624 nginx: master process /usr/sbin/nginx
           └─5625 nginx: worker process

сен 30 12:54:17 nginx systemd[1]: Starting The nginx HTTP and reverse proxy server...
сен 30 12:54:18 nginx nginx[5619]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
сен 30 12:54:18 nginx nginx[5619]: nginx: configuration file /etc/nginx/nginx.conf test is successful
сен 30 12:54:18 nginx systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@nginx ~]# semanage port -d -t http_port_t -p tcp 5080
[root@nginx ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
### 2. Переключатели setsebool
Включим nis_enabled. Запустим nginx и проверим его статус. После удачного запуска отключим nis_enabled и перейдем к следующему решению.
```
[root@nginx ~]# setsebool -P nis_enabled 1
[root@nginx ~]# systemctl start nginx
[root@nginx ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Чт 2021-09-30 12:55:14 UTC; 3s ago
  Process: 5674 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 5672 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 5671 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 5676 (nginx)
   CGroup: /system.slice/nginx.service
           ├─5676 nginx: master process /usr/sbin/nginx
           └─5677 nginx: worker process

сен 30 12:55:14 nginx systemd[1]: Starting The nginx HTTP and reverse proxy server...
сен 30 12:55:14 nginx nginx[5672]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
сен 30 12:55:14 nginx nginx[5672]: nginx: configuration file /etc/nginx/nginx.conf test is successful
сен 30 12:55:14 nginx systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@nginx ~]# setsebool -P nis_enabled 0
[root@nginx ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
### 3. Формирование и установка модуля SELinux
Создаем модуль через комманды ausearch и audit2allow. Через semodule подключаем созданый модуль, и запускаем nginx. После удачного запуска удаляем добавленный модуль.
```
[root@nginx ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@nginx ~]# semodule -i my-nginx.pp
[root@nginx ~]# systemctl start nginx
[root@nginx ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Чт 2021-09-30 13:04:10 UTC; 24s ago
  Process: 5749 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 5747 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 5745 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 5751 (nginx)
   CGroup: /system.slice/nginx.service
           ├─5751 nginx: master process /usr/sbin/nginx
           └─5752 nginx: worker process

сен 30 13:04:10 nginx systemd[1]: Starting The nginx HTTP and reverse proxy server...
сен 30 13:04:10 nginx nginx[5747]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
сен 30 13:04:10 nginx nginx[5747]: nginx: configuration file /etc/nginx/nginx.conf test is successful
сен 30 13:04:10 nginx systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@nginx ~]# semodule -r my-nginx
libsemanage.semanage_direct_remove_key: Removing last my-nginx module (no other my-nginx module exists at another priority).
[root@nginx ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

# Домашнее задание №2
## Описание файлов в директории
logFileFull.log - полный лог выполнения на ns01  
used_commands.txt - команды которые использовал  

screenshot - сделал несколько скришотов т.к. это требовалось. Но я все описал ниже, и мне кажется что они не нужны.  

## Описание проделанной работы
Чтоыб понять на каком из серверов проблема подключился по ssh к клиенту и серверу и выполнил комманду tail чтобы понять на какой ВМ проблема
```
tail -n0 -f /var/log/audit/audit.log
```
Выполнил на клиенте нужные комманды и получил ошибку
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
В этот момент на ns01 tail выдал ошибки. Скопировал их в файл.
```
[root@ns01 ~]# tail -n0 -f /var/log/audit/audit.log
type=AVC msg=audit(1633085762.808:1926): avc:  denied  { create } for  pid=5203 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
type=SYSCALL msg=audit(1633085762.808:1926): arch=c000003e syscall=2 success=no exit=-13 a0=7f6f209d7050 a1=241 a2=1b6 a3=24 items=0 ppid=1 pid=5203 auid=4294967295 uid=25 gid=25 euid=25 suid=25 fsuid=25 egid=25 sgid=25 fsgid=25 tty=(none) ses=4294967295 comm="isc-worker0000" exe="/usr/sbin/named" subj=system_u:system_r:named_t:s0 key=(null)
type=PROCTITLE msg=audit(1633085762.808:1926): proctitle=2F7573722F7362696E2F6E616D6564002D75006E616D6564002D63002F6574632F6E616D65642E636F6E66
^C
[root@ns01 ~]# vi /tmp/1.log
```
По выводу выше, проблема с созданием файла, не совпадают тип и домен (named_t|etc_t). Предположил что в названии типа тоже должно присутствовать named :)  
Далее передал строки с ошибкой комманде sealert, чтобы посмотреть предложенные варианты
```
[root@ns01 ~]# sealert -a /tmp/1.log
100% done
found 1 alerts in /tmp/1.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/named доступ create к файл named.ddns.lab.view1.jnl.

*****  Модуль catchall_labels предлагает (точность 83.8)  ********************

Если вы хотите разрешить named иметь create доступ к named.ddns.lab.view1.jnl $TARGET_УЧЕБНЫЙ КЛАСС
То необходимо изменить метку на named.ddns.lab.view1.jnl
Сделать
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
где FILE_TYPE может принимать значения: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
Затем выполните:
restorecon -v 'named.ddns.lab.view1.jnl'


*****  Модуль catchall предлагает (точность 17.1)  ***************************

Если вы считаете, что named должно быть разрешено create доступ к named.ddns.lab.view1.jnl file по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
# semodule -i my-iscworker0000.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:named_t:s0
Целевой контекст              system_u:object_r:etc_t:s0
Целевые объекты               named.ddns.lab.view1.jnl [ file ]
Источник                      isc-worker0000
Путь к источнику              /usr/sbin/named
Порт                          <Unknown>
Узел                          <Unknown>
Исходные пакеты RPM           bind-9.11.4-26.P2.el7_9.7.x86_64
Целевые пакеты RPM
Пакет регламента              selinux-policy-3.13.1-266.el7.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      ns01
Платформа                     Linux ns01 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           1
Впервые обнаружено            2021-10-01 10:56:02 UTC
В последний раз               2021-10-01 10:56:02 UTC
Локальный ID                  4c2d631c-c642-4ba0-94df-34e7106ac85f

Построчный вывод сообщений аудита
type=AVC msg=audit(1633085762.808:1926): avc:  denied  { create } for  pid=5203 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1633085762.808:1926): arch=x86_64 syscall=open success=no exit=EACCES a0=7f6f209d7050 a1=241 a2=1b6 a3=24 items=0 ppid=1 pid=5203 auid=4294967295 uid=25 gid=25 euid=25 suid=25 fsuid=25 egid=25 sgid=25 fsgid=25 tty=(none) ses=4294967295 comm=isc-worker0000 exe=/usr/sbin/named subj=system_u:system_r:named_t:s0 key=(null)

Hash: isc-worker0000,named_t,etc_t,file,create
```
Первый вариант был примлем, но хотелось понять для начала где создается файл. Посмотрел контекст исполняемого файла и статус демона named
```
[root@ns01 ~]# ls -Z /usr/sbin/named
-rwxr-xr-x. root root system_u:object_r:named_exec_t:s0 /usr/sbin/named
[root@ns01 ~]# systemctl status named.service
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Пт 2021-10-01 09:06:51 UTC; 2h 6min ago
  Process: 5201 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 5199 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 5203 (named)
   CGroup: /system.slice/named.service
           └─5203 /usr/sbin/named -u named -c /etc/named.conf

окт 01 09:06:51 ns01 named[5203]: network unreachable resolving './NS/IN': 2001:500:2d::d#53
окт 01 09:06:51 ns01 named[5203]: network unreachable resolving './DNSKEY/IN': 2001:503:c27::2:30#53
окт 01 09:06:51 ns01 named[5203]: managed-keys-zone/view1: Key 20326 for zone . acceptance timer complete: key now trusted
окт 01 09:06:51 ns01 named[5203]: managed-keys-zone/default: Key 20326 for zone . acceptance timer complete: key now trusted
окт 01 09:06:51 ns01 named[5203]: resolver priming query complete
окт 01 09:06:52 ns01 named[5203]: resolver priming query complete
окт 01 10:56:02 ns01 named[5203]: client @0x7f6f1403c3e0 192.168.50.15#39567/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
окт 01 10:56:02 ns01 named[5203]: client @0x7f6f1403c3e0 192.168.50.15#39567/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15
окт 01 10:56:02 ns01 named[5203]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
окт 01 10:56:02 ns01 named[5203]: client @0x7f6f1403c3e0 192.168.50.15#39567/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': error: journal open failed: unexpected error
```
И systemd выдал мне нужную информацию. Выпишу ее повторно.
```
окт 01 10:56:02 ns01 named[5203]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
```
Мы узнали где создается файл. Нужно посмотреть контекст директории и файлов в ней.
```
[root@ns01 ~]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
[root@ns01 ~]# ls -Z /etc/named/
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
По названию директории предположил что если к named будут подключаться еще клиенты, то будут созданы дополнительные файлы. А предложенный вариант sealert распространяется только на один файл. И при подключении нового клиента прийдется проделать всю процедуру повторно. Значит нужно дать разрешение на всю директорию dynamic.  
В выводе sealert были указаны нужные нам типы. Повторю их для наглядности.
```
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
где FILE_TYPE может принимать значения: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
```
Любой из представленных типов скорее всего подошел. Но хотелось бы понять какой из них лучше подойдет.  
Нашел каталог /var/named/dynamic/. И посмотрел его контекст и тип, named_cache_t. И в предложенном варианте sealert он есть.
```
[root@ns01 ~]# ls -Z /var/named/dynamic/
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 default.mkeys.jnl
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 view1.mkeys.jnl
[root@ns01 ~]# ls -Z /var/named/
drwxrwx---. named named system_u:object_r:named_cache_t:s0 data
drwxrwx---. named named system_u:object_r:named_cache_t:s0 dynamic
-rw-r-----. root  named system_u:object_r:named_conf_t:s0 named.ca
-rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.empty
-rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.localhost
-rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.loopback
drwxrwx---. named named system_u:object_r:named_cache_t:s0 slaves
```
Выполнил комманду semanage с нужными параметрами и указал шаблон директории dynamic. Далее восстановил контекст директории. Повторно проверил 
```
[root@ns01 ~]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic(/.*)?'
[root@ns01 ~]# restorecon -v -R /etc/named/dynamic/
restorecon reset /etc/named/dynamic context unconfined_u:object_r:etc_t:s0->unconfined_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
[root@ns01 ~]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:named_cache_t:s0 named.ddns.lab
-rw-rw----. named named system_u:object_r:named_cache_t:s0 named.ddns.lab.view1
```
На клиенте повторно выполнил нужные комманды. Ошибок не последовало.
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
>
```
Далее на ns01 проверил что нужный файл был создан
```
[root@ns01 ~]# ll /etc/named/dynamic/named.ddns.lab.view1.jnl
-rw-r--r--. 1 named named 700 окт  1 11:42 /etc/named/dynamic/named.ddns.lab.view1.jnl
```
Готово.  

## Рассмотрим воторой способ решения задачи
Проверяем второй способ, предложенный sealert, создание модуля named.
Удаляем ранее созданный шаблон
```
[root@ns01 ~]# semanage fcontext -d -t named_cache_t '/etc/named/dynamic(/.*)?'
[root@ns01 ~]# restorecon -v -R /etc/named/dynamic/
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_cache_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_cache_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_cache_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_cache_t:s0->system_u:object_r:etc_t:s0
[root@ns01 ~]# ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
```
И создаем наш модуль. Посмотрим исходник этого модуля.
```
[root@ns01 ~]# audit2allow -M named_add --debug < /tmp/1.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i named_add.pp

[root@ns01 ~]# ll
total 24
-rw-------. 1 root root 5570 апр 30  2020 anaconda-ks.cfg
-rw-r--r--. 1 root root  925 окт  1 11:49 named_add.pp
-rw-r--r--. 1 root root  196 окт  1 11:49 named_add.te
-rw-------. 1 root root 5300 апр 30  2020 original-ks.cfg
[root@ns01 ~]# cat named_add.te

module named_add 1.0;

require {
        type etc_t;
        type named_t;
        class file create;
}

#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
[root@ns01 ~]# semodule -i named_add.pp
```
Если я правильно понял то мы дадим доступ named создавать файлы если на директории есть тип etc_t, и тут в модуле нас предупреждают что это базовый тип.  
Если мы посмотрим каталог /etc (указал усеченый вывод), то мы убедимся что большая часть файлов и папок с типом etc_t.
```
[root@ns01 ~]# ls -Z /etc/
-rw-r--r--. root root   system_u:object_r:adjtime_t:s0   adjtime
-rw-r--r--. root root   system_u:object_r:etc_aliases_t:s0 aliases
-rw-r--r--. root root   system_u:object_r:etc_aliases_t:s0 aliases.db
drwxr-xr-x. root root   system_u:object_r:etc_t:s0       alternatives
-rw-------. root root   system_u:object_r:etc_t:s0       anacrontab
drwxr-x---. root root   system_u:object_r:etc_t:s0       audisp
drwxr-x---. root root   system_u:object_r:auditd_etc_t:s0 audit
drwxr-xr-x. root root   system_u:object_r:etc_t:s0       bash_completion.d
-rw-r--r--. root root   system_u:object_r:etc_t:s0       bashrc
drwxr-xr-x. root root   system_u:object_r:etc_t:s0       binfmt.d
-rw-r--r--. root root   system_u:object_r:etc_t:s0       centos-release
-rw-r--r--. root root   system_u:object_r:etc_t:s0       centos-release-upstream
drwxr-xr-x. root root   system_u:object_r:etc_t:s0       chkconfig.d
-rw-r--r--. root root   system_u:object_r:etc_t:s0       chrony.conf
-rw-r-----. root chrony system_u:object_r:chronyd_keys_t:s0 chrony.keys
drwxr-xr-x. root root   system_u:object_r:etc_t:s0       cifs-utils
```
Если named взломают то могут получить полный контроль над сервером. Создание модуля, не наш вариант.
А вот установка правильного контекста на директорию, лучший вариант.

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)