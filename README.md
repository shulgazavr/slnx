# Практика с `SELinux`.
### Описание домашнего задания.
1. Запустить `nginx` на нестандартном порту 3-мя разными способами:
- переключатели `setsebool`;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля `SELinux`.

2. Обеспечить работоспособность приложения при включенном `SELinux`.
- развернуть приложенный [стенд](https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems);
- выяснить причину неработоспособности механизма обновления зоны (см. [README](https://github.com/mbfx/otus-linux-adm/blob/master/selinux_dns_problems/README.md));
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

### Особенности [Vagrantfile](https://github.com/shulgazavr/slnx/blob/main/Vagrantfile):
- Описание параметров создаваемого сервера:
```
servers = [
    { :hostname => 'web', :ips => '192.168.31.202', :ram => 2048, :cpuz => 2 },
]
```

- Настройка параметров сети:
```
            node.vm.network :public_network, ip: machine[:ips], bridge: "wlp58s0"
```

> Примечание: ip-адреса и имя интерфейса выбраны исходя из особенности окружения.

### Добавление нестандартного порта в имеющийся тип и формирование и установка модуля SELinux:
1. Установка виртуальной машины `Vagrant`:
```
vagrant up
```
2. Создание каталогов и файлов сайтов: 
```
# tree /srv/www/vhosts/
/srv/www/vhosts/
|-- nginx1
|   `-- index.html
`-- nginx2
    `-- index.html
```
> Примечание: Для доступа к этим файла по http, необходимо изменить им тип:
> ```
> chcon -v -t httpd_sys_content_t index.html
> ```
3. Добавление в конфигурационный файл `/etc/nginx/nginx.conf` нестандартных портов:
```
...
    server {
        listen       12345;
        listen       [::]:12345;
        root         /srv/www/vhosts/nginx1/;
        index index.html;
    }
    server {
        listen       12346;
        listen       [::]:12346;
        root         /srv/www/vhosts/nginx2;
        index index.html;
    }
...
```
4. Проверка стандартных портов в имеющимся типе:
```
# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988

```
5. Добавление нестандартного порта, проверка:
```
semanage port -a -t http_port_t -p tcp 12345
```
```
# semanage port -l | grep http_port_t
http_port_t                    tcp      12345, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
6. Обнуление лог-файла:
```
# cat /dev/null > /var/log/audit/audit.log
```
7. Запуск `nginx`, получение ошибки:
```
# systemctl start nginx.service 
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
8. Формирование модуля на основе сообщения об ошибке:
```
# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
9. Применение получившегося модуля:
```
# semodule -i nginx.pp
```
10. После выполнения вышеуказанных действий и запуска `nginx`, можно проверять порты:

![image](https://user-images.githubusercontent.com/105816449/221911887-73407dc8-7619-445f-b125-71ccdafacf25.png)

11. Удаление модуля и удаление нестандартного порта:
```
# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```
```
# semanage port -d -t http_port_t -p tcp 12345
```
> Примечание: можно удостовериться, что сервис `nginx` не сможет успешно перезагрузиться:
>```
># systemctl restart nginx.service 
>Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
>```

### Разрешение в `SELinux` работу `nginx` на нестандартных портах 12345 и 12346 при помощи переключателей `setsebool`.
1. Обнуление лог-файла:
```
# cat /dev/null > /var/log/audit/audit.log
```
2. Рестарт `nginx`, получение ошибки:
```
# systemctl restart nginx.service 
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
3. Просмотр лог-файла и, по найденному времени ошибки, просмотр информации о запрете:
```
# cat /var/log/audit/audit.log
type=AVC msg=audit(1677601548.669:1599): avc:  denied  { name_bind } for  pid=26808 comm="nginx" src=12345 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1677601548.669:1599): arch=c000003e syscall=49 success=no exit=-13 a0=8 a1=55b9ec900ab0 a2=10 a3=7ffe401a8bb0 items=0 ppid=1 pid=26808 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=PROCTITLE msg=audit(1677601548.669:1599): proctitle=2F7573722F7362696E2F6E67696E78002D74
type=SERVICE_START msg=audit(1677601548.678:1600): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
```
# grep 1677601548.669:1599 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1677601548.669:1599): avc:  denied  { name_bind } for  pid=26808 comm="nginx" src=12345 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
4. Замена значения параметра, рестарт `nginx`:
```
# setsebool -P nis_enabled 1
```
```
# systemctl restart nginx.service
```
5. После выполнения вышеуказанных действий и запуска `nginx`, можно проверять порты:
 
![image](https://user-images.githubusercontent.com/105816449/221917320-3f7ec874-d939-432f-afc5-30f3d456dfd5.png)

6. Возврат параметра в исходное значение:
```
setsebool -P nis_enabled off
```
> Примечание: доступ по этим портам будет открыт до перезапуска сервиса `nginx`.

### Обеспечить работоспособность приложения при включенном `SELinux`.
После загрузки репозитория и запуска виртуальных машин была проверена возможность внесения изменений в зону:
```
$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab 
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
При просмотре логов аудита на сервере, на котором предпринималась попытка внести изменения в зону, была обноружена ошибка в контексте безопасности:
```
# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1677616500.550:1759): avc:  denied  { create } for  pid=4692 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
> Примечание: в данном выводе видно, что контекст источника `scontext=system_u:system_r:named_t:s0` не совпадает с контекстом цели `tcontext=system_u:object_r:etc_t:s0`.
> Конкретней, ошибка в несоответствии типов источника `named_t` и цели `etc_t`. Исходя из определения типа - `атрибут объекта, который определяет, кто может
получить к нему доступ`, можно предположить, что изменение типа цели на подходящий (т.е. дающий возможность совершать какие-либо действия) для источника, решит проблему.

Проверка пути правильного расположения файлов, а главное определение правильного типа контекста этих файлов:
```
# semanage fcontext -l | grep named
...
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
...
```
Результат команды позволяет сделать вывод, что необходимо изменить тип в контексте конфигурационных файлов зон с `etc_t` на `named_zone_t`, а содержимое `/etc/named.conf` указывает, что необходимые файлы находятся в каталоге `/etc/named/`:
```
chcon -R -t named_zone_t /etc/named
```

После этого на ВМ `client` можно проверить внесение изменений:
```
$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
```
$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57171
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Feb 28 21:18:33 UTC 2023
;; MSG SIZE  rcvd: 96

```

PS. Помимо способа, предлагаемого в методическом пособии, возникла идея привести текущее расположение конфигурационных файлов в соответствии со стандартами `SELinux`. Т.е. выполнить ряд действий:
- перенести файлы из `/etc/named/` в `/var/named/`;
```
# cp -r /etc/named/* /var/named/
```
- изменить пути к файлам на актуальные в `/etc/named.conf`;
```
# sed -i "s:/etc/named/:/var/named/:g" /etc/named.conf
```
- поправить права доступа для перенесённых файлов:
```
# chown root:named /var/named/named.50.168.192.rev
# chown root:named /var/named/named.dns.lab       
# chown root:named /var/named/named.dns.lab.view1
# chown root:named /var/named/named.newdns.lab
chown -R named:named /var/named/dynamic/*
```
- зарестартить named:
```
systemctl restart named.service
```
После чего можно снова проверить внесение изменений:
```
$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
```
```
$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30313
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Feb 28 21:38:20 UTC 2023
;; MSG SIZE  rcvd: 80
```
