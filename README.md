

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

