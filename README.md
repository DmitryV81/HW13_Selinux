Домашняя работа №15 "Практика Selinux"

1. Запустить nginx на нестандартном порту 3-мя разными способами:

    переключатели setsebool;
    
    добавление нестандартного порта в имеющийся тип;
    
    формирование и установка модуля SELinux.
    
    К сдаче:
    
    README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.

    развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
   
    выяснить причину неработоспособности механизма обновления зоны (см. README);
    
    предложить решение (или решения) для данной проблемы;
    
    выбрать одно из решений для реализации, предварительно обосновав выбор;
    
    реализовать выбранное решение и продемонстрировать его работоспособность.
    
    К сдаче:
    
    README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
    
    исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

Ход работы.

Первая часть ДЗ.

Подключаемся к ВМ.

Проверяем статус firewalld и selinux

Способ 1.

```
hunter@Hunter:~/HW13$ vagrant ssh
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@selinux ~]# getenforce
Enforcing
```
В файле лога /var/log/audit/audit.log ищем запись о блокировке порта 4881
```
[root@selinux ~]# less /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1675416948.001:850): avc:  denied  { name_bind } for  pid=2939 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Устанавливаем пакет policycoreutils-python
```
yum install policycoreutils-python
```
С помощью утилиты audit2why по метке времени находим информацию о причине запрета
```
[root@selinux ~]# grep 1675416948.001:850 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1675416948.001:850): avc:  denied  { name_bind } for  pid=2939 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
[root@selinux ~]# 
```
В последней строке вывода утилиты audit2why видим рекомендацию по устранению проблемы: setsebool -P nis_enabled 1

Включаем параметр nis_enabled, перезапускаем nginx и проверяем статус сервиса:
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor prese                                                                                                 t: disabled)
   Active: active (running) since Fri 2023-02-03 10:17:17 UTC; 15s ago
  Process: 3313 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3310 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3309 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=                                                                                                 0/SUCCESS)
 Main PID: 3315 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3315 nginx: master process /usr/sbin/nginx
           └─3317 nginx: worker process

Feb 03 10:17:17 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Feb 03 10:17:17 selinux nginx[3310]: nginx: the configuration file /etc/ngi...ok
Feb 03 10:17:17 selinux nginx[3310]: nginx: configuration file /etc/nginx/n...ul
Feb 03 10:17:17 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```

Можно проверить, что nginx работает на порту 4881:
```
hunter@Hunter:~/HW13$ curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
..............
```

Способ 2.

В выводе комнад semanage ищем разрешённые порты для http:
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Порт 4881 отсутствует в списке. Добавим его
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Перезапустим nginx и проверим работу:
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-02-03 10:50:08 UTC; 7s ago
  Process: 22253 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22250 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22249 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22255 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22255 nginx: master process /usr/sbin/nginx
           └─22257 nginx: worker process

Feb 03 10:50:07 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Feb 03 10:50:08 selinux nginx[22250]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Feb 03 10:50:08 selinux nginx[22250]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Feb 03 10:50:08 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

```

Способ 3.

С помощью утилиты audit2allow на основе анализа логов nginx формируем модуль, разрешающий работу nginx на порту 4881:
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Устанавливаем модуль, перезапускаем nginx и проверяем:
```
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# semodule -l | grep nginx
nginx   1.0
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-02-03 10:59:14 UTC; 7s ago
  Process: 22315 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22313 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22312 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22317 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22317 nginx: master process /usr/sbin/nginx
           └─22319 nginx: worker process

Feb 03 10:59:14 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Feb 03 10:59:14 selinux nginx[22313]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Feb 03 10:59:14 selinux nginx[22313]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Feb 03 10:59:14 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Вторая часть ДЗ.

Клонируем стенд

Подключаемся к ВМ client и пробуем внести изменения в зону

```
hunter@Hunter:~/HW13/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
[vagrant@client ~]$ dig @192.168.50.10 ns01.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.50.10 ns01.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16786
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ns01.dns.lab.                  IN      A

;; ANSWER SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Fri Feb 03 11:26:04 UTC 2023
;; MSG SIZE  rcvd: 71

[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Внести изменения не получилось.

Используя утилиту audit2why анализируем логи:
```
[root@client ~]# tail /var/log/audit/audit.log | audit2why
Nothing to do
```
Ошибок нет.

Подключаемся к ns01 и с помощью утилиты audit2why ищем ошибки в логах:
```
hunter@Hunter:~/HW13/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Fri Feb  3 11:20:35 2023 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1675423683.364:1939): avc:  denied  { create } for  pid=5238 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]#

```

На сервере появилась ошибка в контексте безопасности, вместо типа named_t используется etc_t
В каталоге /etc/named находим подтверждение проблемы - конфигурационные файлы расположены в неправильном каталоге.

```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Посмотрим, в каком каталоге должны находиться конфигурационные файлы, чтобы на них распространялись политики Selinux:

```
[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
[root@ns01 ~]#
```
Изменим тип контекста безопасности для каталога /etc/named
```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
[root@ns01 ~]#
```

Для проверки подключаемся на ВМ client, и повторяем команду:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3658
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Fri Feb 03 12:30:03 UTC 2023
;; MSG SIZE  rcvd: 96

[vagrant@client ~]$
```
Изменения применились.
