# zimbra-installation-guide
Zimbra installation guide on Ubuntu 18.04 with dnsmasq (local domain)
<h1>Настройка Zimbra Open Source</h1>
<h2>1. Настраиваем время и имя</h2>

Устанавливаем корректный часовой пояс:

* timedatectl set-timezone Europe/Moscow


Устанавливаем имя системы:

* sudo hostnamectl set-hostname zimbra.mail

<h2>2. Открываем порты</h2>

Для нормальной работы Zimbra нужно открыть много портов:

    25 — основной порт для обмена почтой по протоколу SMTP.
    80 — веб-интерфейс для чтения почты (http).
    110 — POP3 для загрузки почты.
    143 — IMAP для работы с почтовым ящиком с помощью клиента.
    443 — SSL веб-интерфейс для чтения почты (https).
    465 — безопасный SMTP для отправки почты с почтового клиента.
    587 — SMTP для отправки почты с почтового клиента (submission).
    993 — SSL IMAP для работы с почтовым ящиком с помощью клиента.
    995 — SSL POP3 для загрузки почты.
    5222 — для подключения к Zimbra по протоколу XMPP.
    5223 — для защищенного подключения к Zimbra по протоколу XMPP.
    7071 — для защищенного доступа к администраторской консоли.
    8443 — SSL веб-интерфейс для чтения почты (https).
    7143 — IMAP для работы с почтовым ящиком с помощью клиента.
    7993 — SSL IMAP для работы с почтовым ящиком с помощью клиента.
    7110 — POP3 для загрузки почты.
    7995 — SSL POP3 для загрузки почты.
    9071 — для защищенного подключения к администраторской консоли.


Порты для веб:

<code>iptables -I INPUT -p tcp --match multiport --dports 80,443 -j ACCEPT</code>

Порты для почты:

<code>iptables -I INPUT -p tcp --match multiport --dports 25,110,143,465,587,993,995 -j ACCEPT</code>

Порты для Zimbra:

<code>iptables -I INPUT -p tcp --match multiport --dports 5222,5223,9071,7071,8443,7143,7993,7110,7995 -j ACCEPT </code>

Сохраняем правила: 

* netfilter-persistent save  

Eсли команда вернет ошибку, то установим пакет: 
* apt-get install iptables-persistent.

<h2>3. Настройки сети</h2>

На виртуальной машине должно быть доступно два сетевых адаптера: NAT и Host-only. Первый необходим для доступа в Интернет, второй - для установки статического IP, необходимого для работы с Zimbra.

<h3>3.1 Задаем статический IP у host-only адаптера</h3>



* sudo gedit /etc/network/interfaces
```
auto ens33  
iface eth0 inet dhcp  
```

```  
auto ens38  
iface eth1 inet static  
address 192.168.227.135  
netmask 255.255.255.0  
broadcast 192.168.227.255  
network 192.168.227.0  
gateway 192.168.227.1  
dns-nameservers 127.0.0.1  
dns-nameservers 8.8.8.8
```

<h3>3.2 Устанавливаем и настраиваем dnsmasq</h3>

* sudo apt install dnsmasq 

Будут ошибки во время установки (так и должно быть)

* sudo systemctl stop systemd-resolved
* sudo systemctl disable systemd-resolved
* sudo rm -v /etc/resolv.conf

Меняем конфиги dnsmasq:

* sudo gedit /etc/dnsmasq.conf
```
# Never forward plain names (without a domain)
domain-needed

# Turn off DHCP on eth1
no-dhcp-interface=eth1

# Never forward addresses in the non-routable address space (RFC1918)
bogus-priv

# Add domain to host names
expand-hosts

# Domain to be added if expand-hosts is set
domain=zimbra.mail

# Local domain to be served from /etc/hosts file
local=/zimbra.mail/

# Don't read /etc/resolv.conf (I deleted it). Get the external name server from this file, see 'server' below
no-resolv

# External server, works with no-resolv
server=8.8.8.8

# For MX record
mx-host=mail,zimbra.mail,10
```

Дополняем hosts:

* sudo gedit /etc/hosts
```
127.0.0.1	localhost  
192.168.227.135	zimbra.mail  
```

Запускаем dnsmasq:

* sudo systemctl start dnsmasq
* sudo systemctl enable dnsmasq

Проверяем правильность установки командами:
* dig A zimbra.mail
```
; <<>> DiG 9.11.3-1ubuntu1.15-Ubuntu <<>> A zimbra.mail
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15313
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;zimbra.mail.			IN	A

;; ANSWER SECTION:
zimbra.mail.		0	IN	A	192.168.227.135

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Aug 18 15:13:34 MSK 2021
;; MSG SIZE  rcvd: 56
```
* dig mx mail
```
; <<>> DiG 9.11.3-1ubuntu1.15-Ubuntu <<>> mx mail
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53162
;; flags: qr aa rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;mail.				IN	MX

;; ANSWER SECTION:
mail.			0	IN	MX	10 zimbra.mail.

;; ADDITIONAL SECTION:
zimbra.mail.		0	IN	A	192.168.227.135

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Aug 18 15:12:36 MSK 2021
;; MSG SIZE  rcvd: 76

```

<h2>4. Устанавливаем Zimbra</h2>

* wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_3869.UBUNTU18_64.20190918004220.tgz

...

* sudo ./install.sh

...

Во время установки будет предложено поменять домен (It is suggested that the domain name have an MX record configured in DNS). Соглашаемся, меняем на 'mail'.

...

<h2>5. Запускаем Zimbra</h2>

* https://zimbra.mail:7071
