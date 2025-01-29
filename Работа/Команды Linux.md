

посмотреть DHCP сервер:
```bash
grep dhcp-server-identifier /var/lib/dhcp/dhclient/state/dhclient.leases
```
Посмотреть размер диска:
```bash
fdisk -l /dev/vda
```
Посмотреть свободное место в разделе:
```bash
parted /dev/sdb/ print free
```
Посмотреть активные сервисы:
```bash
systemctl --type=service --state=active
systemctl --type=service --state=running
```
Расширить раздел:
```bash
growpart /dev/sdb 2
```
Расширить диск:
```bash
resize2fs /dev/vda
```
Проверить права на папки:
```bash
mount
mount |grep /tmp
```
Удалить большое количество файлов:
```bash
find /opt/_Tomcat/dockarchive-api-28080/work/Catalina/localhost/api/ -name "*.tmp" -type f -print0 | xargs -0 /bin/rm -f
```
Удалить файлы старше чем:
```bash
find ~/Downloads -type f -mtime +35 -delete
с указанием расширения:
find ~/Downloads -name "*.zip" -type f -mtime +35 -delete
```
Посмотреть количество файлов  в папке:
```bash
Выдаст и файлы и каталоги:
ls | wc -l
Только файлы:
find . -maxdepth 1 -type f | wc -l
```
Добавить exec для папки:
```bash
mount -o remount,exec /tmp
```
Поменять права на папку:
```bash
chown myuser:myuser /var/run/mydaemon
```
Посмотреть информацию о версии ОС:
```bash
cat /etc/os-release
```
Команда для проверки кто сидит на порте:
```bash
lsof -i :80
```
Проверка доступности порта
```bash
nc -zv ip port

nc -vz -u 10.1.0.100 53 #UDP PORT

nc -vz 192.168.31.247 1-1000 2>&1 | grep succeeded
```
Прослушивание порта:
```bash
nc -nlv 8080
```
Копирование с удаленного сервера на локальную тачку
```bash
scp user@172.18.32.199:/opt/_Tomcat/dockarchive-8080/webapps/app.war .\TST1-REP-APPCONSOLE1\
```
Копирование на удаленный сервер
```bash
scp .\TST1-REP-APPCONSOLE1\app.war user@172.18.32.199:/opt/_Tomcat/dockarchive-8080/webapps/app.war
```
DNS Zones:
```bash
sudo vim /etc/bind/zones/*
```
grep port
```bash
ss -tulpn |grep 53
```
Посмотреть какой процесс занимает порт
```bash
fuser port/tcp
```
Варианты команды ls:
```bash
ls -lhaF | grep ^l # list links

ls -lhaF | grep ^d # list directories

ls -lhaF | grep ^- # list files
```
Посмотреть свободные ip
```bash
nmap -v -sP 192.168.1.3/24
```
Посмотреть срок сертификата:
```bash
openssl x509 -enddate -noout -in <filename>.<type>
```
задать размер экрана VNC
```bash
vncserver -geometry 1900x1000
```
посмотреть символьные ссылки в каталоге
```bash
sudo find /var/www/ -type l
```
Посмотреть исходящий ip:
```bash
dig TXT +short o-o.myaddr.l.google.com @ns1.google.com | sed 's/"//g'

ЛИБО:

curl ipinfo.io
```
вывод запущенных сервисов:
```bash
systemctl list-units --type=service --state=running
```
проверка конфига haproxy
```bash
haproxy -f /etc/haproxy/haproxy.cfg -c
```
обновление сертификатов в Альте:
```bash
cp /home/user/qca2020.crt /etc/pki/ca-trust/source/anchors/ && update-ca-trust
```
сетевая нагрузка на интерфейс(Нужно поставить утилиту nload)
```bash
nload eth0
```
Чтобы найти самые большие файлы в определенном месте, просто добавьте путь к команде **find**:
```bash
find /home/fox -type f -exec du -Sh {} + | sort -rh | head -n 5
```
или
```bash
find /home/fox -type f -printf "%s %p\n" | sort -rn | head -n 5
```
Чтобы отобразить самые большие папки или файлы, включая подкаталоги, выполните следующую команду:
```bash
du -Sh | sort -rh | head -5
```
## journalctl

Флаг `-b` — одна из самых простых опций, которой вы часто будете пользоваться. С его помощью вы сможете вывести для просмотра все записи журнала, собранные с момента последней перезагрузки.
```bash
journalctl -b
```
Если вы получили отчеты о перебоях в работе службы, которые начались в 9:00 и закончились час назад, вы можете ввести следующую команду:
```bash
journalctl --since 09:00 --until "1 hour ago"
```
Например, чтобы посмотреть все журналы единицы Nginx в нашей системе, мы можем ввести команду:
```bash
journalctl -u nginx.service
```
Обычно при этом также используется фильтрация по времени, чтобы вывести строки, которые вас интересуют. Например, чтобы проверить, как служба работает сегодня, можно ввести команду:
```bash
journalctl -u nginx.service --since today
```
Например, если ваш процесс Nginx использует единицу PHP-FPM для обработки динамического контента, вы можете объединить их данные в хронологическом порядке, указав обе единицы:
```bash
journalctl -u nginx.service -u php-fpm.service --since today
```


#linux #commands #everyday