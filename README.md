

Подключаемся к ВМ.

Проверяем статус firewalld и selinux
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
