### OS Ubuntu
```bash
# netstat -tulnp | grep :::
# ip addr | grep inet6
Выключаем интерфейс
```
```bash
На ssh
# nano /etc/ssh/sshd_config или nano /etc/ssh/ssh_config
Добавить ListenAddress 0.0.0.0
service sshd restart
```
Если есть, то редактируем
```bash
# nano /etc/exim4/exim4.conf.template
disable_ipv6 = true
# service exim4 restart
```
В dhclient для отключения ipv6 в конфиге убираем все параметры в запросе request, начинающиеся с dhcp6. Должно получиться вот так:
```bash
# nano /etc/dhcp/dhclient.conf
request subnet-mask, broadcast-address, time-offset, routers,
        domain-name, domain-name-servers, domain-search, host-name,
        netbios-name-servers, netbios-scope, interface-mtu,
        rfc3442-classless-static-routes, ntp-servers;
# service networking restart
```
Если есть, то редактируем
```bash
nano /etc/netconfig
#udp6       tpi_clts      v     inet6    udp     -       -
#tcp6       tpi_cots_ord  v     inet6    tcp     -       -
```
Перезапускаем службу rpcbind и nfs-common, которая от него зависит:
```bash
# service rpcbind restart
# service nfs-common restart
```
Проверяем
```bash
# netstat -tulnp | grep :::
```
dhclient почему-то остался висеть на ipv6 порту, но ладно, это не страшно, запрашивать по ipv6 он все равно ничего не будет. Теперь полностью отключаем ipv6 в Debian
```bash
# nano /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.ens33.disable_ipv6 = 1
# sysctl -p
```
Проверяем 
```bash
# sysctl net.ipv6.conf.all.disable_ipv6
```
Редактируем
```bash
# nano /etc/modprobe.d/blacklist-ipv6.conf
добавляем blacklist ipv6
# update-initramfs -u
# reboot
```
Проверяем
```bash
# netstat -tulnp | grep :::
```
Смотрим что осталось
```bash
# netstat -tulnp | grep :::   или так # ss -lnptu | sort
udp6       0      0 :::33600                :::*                                2435/avahi-daemon: 
udp6       0      0 :::5353                 :::*                                2435/avahi-daemon:

nano /etc/default/avahi-daemon
use-ipv6=no
publish-a-on-ipv6=no

service avahi-daemon stop
service avahi-daemon start
```

