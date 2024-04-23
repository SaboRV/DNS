# Цель домашнего задания
Создать домашнюю сетевую лабораторию. Изучить основы DNS, научиться работать с технологией Split-DNS в Linux-based системах
<pre>
Описание домашнего задания:
1. взять стенд (https://github.com/erlong15/vagrant-bind) 
   добавить еще один сервер client2
   завести в зоне dns.lab имена:
      web1 - смотрит на клиент1
      web2  смотрит на клиент2
   завести еще одну зону newdns.lab
   завести в ней запись
      www - смотрит на обоих клиентов

2. настроить split-dns
   клиент1 - видит обе зоны, но в зоне dns.lab только web1
   клиент2 видит только dns.lab
</pre>

Формат сдачи ДЗ - vagrant + ansible

## Введение
DNS(Domain Name System, Служба доменных имён) -  это распределенная система, для получения информации о доменах. DNS используется для сопоставления IP-адресов и доменных имён.
Сопостовления IP-адресов и DNS-имён бывают двух видов: 
Прямое (DNS-bмя в IP-адрес)
Обратное (IP-адрес в DNS-имя)

Доменная структура DNS представляет собой древовидную иерархию, состоящую из узлов, зон, доменов, поддоменов и т.д. «Вершиной» доменной структуры является корневая зона. Корневая (root) зона обозначается точкой. Далее следуют домены первого уровня (.com, ,ru, .org и т. д.) и т д.

В DNS встречаются понятия зон и доменов:
Зона — это любая часть дерева системы доменных имён, размещаемая как единое целое на некотором DNS-сервере. 
Домен – определенный узел, включающий в себя все подчинённые узлы. 

Давайте разберем основное отличие зоны от домена. Возьмём для примера ресурс otus.ru — это может быть сразу и зона и домен, однако, при использовании зоны otus.ru мы можем сделать отдельную зону mail.otus.ru, которая будет управляться не нами. В случае домена так сделать нельзя...

FQDN (Fully Qualified Domain Name) - полностью указанное доменное имя, т.е. от корневого домена. Ключевой идентификатор FQDN - точка в конце имени. Максимальный размер FQDN — 255 байт, с ограничением в 63 байта на каждое имя домена. Пример FQDN: mail.otus.ru.

Вся информация о DNS-ресурсах хранится в ресурсных записях. Записи хранят следующие атрибуты:
Имя (NAME) - доменное имя, к которому привязана или которому принадлежит данная ресурсная область, либо IP-адрес. При отсутствии данного поля, запись ресурса наследуется от предыдущей записи. 
TTL (время жизни в кэше) - после указанного времени запись удаляется, данное поле может не указываться в индивидуальных записях ресурсов, но тогда оно должно быть указано в начале файла зоны и будет наследоваться всеми записями.
Класс (CLASS) - определяет тип сети (в 99% используется IN - интернет)
Тип (TYPE) - тип записи, синтаксис и назначение записи
Значение (DATA)  

Типы рекурсивных записей:
А (Address record) - отображают имя хоста (доменное имя) на адрес IPv4
AAAA - отображает доменное имя на адрес IPv6
CNAME (Canonical name record/псевдоним) - привязка алиаса к существующему доменному имени
MX (mail exchange) - указывает хосты для отправки почты, адресованной домену. При этом поле NAME указывает домен назначения, а поле DATA приоритет и доменное имя хоста, ответственного за приём почты. Данные вводятся через пробел
NS (name server) -  указывает на DNS-сервер, обслуживающий данный домен. 
PTR (pointer) -  Отображает IP-адрес в доменное имя 
SOA (Start of Authority/начальная запись зоны) - описывает основные начальные настройки зоны. 
SRV (server selection) — указывает на сервера, обеспечивающие работу тех или иных служб в данном домене (например  Jabber и Active Directory).

Для работы с DNS (как клиенту) в linux используют утилиты dig, host и nslookup
Также в Linux есть следующие реализации DNS-серверов:
bind
powerdns (умеет хранить зоны в БД)
unbound (реализация bind)
dnsmasq 
и т д. 

Split DNS (split-horizon или split-brain) — это конфигурация, позволяющая отдавать разные записи зон DNS в зависимости от подсети источника запроса. Данную функцию можно реализовать как с помощью одного DNS-сервера, так и с помощью нескольких DNS-серверов


# Решение:

## 1. Работа со стендом и настройка DNS

Развернут стенд с серверами ns01, ns02 и двумя клиентами client и client2.
С помощью Ansible развернуты зоны dns.lab и newdns.lab.
В зоне dns.lab имена:
web1 - смотрит на client
web2  смотрит на client2
В зоне newdns.lab запись www смотрит на обоих клиентов.

Проверка:
<pre>
[vagrant@client ~]$ dig @192.168.50.10 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37632
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.			IN	A

;; ANSWER SECTION:
web1.dns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns02.dns.lab.
dns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Apr 23 09:29:07 UTC 2024
;; MSG SIZE  rcvd: 127

[vagrant@client ~]$ 


[vagrant@client ~]$ dig @192.168.50.11 web2.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.11 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18153
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.			IN	A

;; ANSWER SECTION:
web2.dns.lab.		3600	IN	A	192.168.50.16

;; AUTHORITY SECTION:
dns.lab.		3600	IN	NS	ns02.dns.lab.
dns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec
;; SERVER: 192.168.50.11#53(192.168.50.11)
;; WHEN: Tue Apr 23 09:30:26 UTC 2024
;; MSG SIZE  rcvd: 127

[vagrant@client ~]$ 

[vagrant@client ~]$ dig @192.168.50.10 www.newdns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.newdns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30978
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.newdns.lab.			IN	A

;; ANSWER SECTION:
www.newdns.lab.		3600	IN	A	192.168.50.16
www.newdns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:
newdns.lab.		3600	IN	NS	ns01.dns.lab.
newdns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Apr 23 09:31:47 UTC 2024
;; MSG SIZE  rcvd: 149

[vagrant@client ~]$ 
</pre>
## 2. Настройка Split-DNS 

У нас уже есть прописанные зоны dns.lab и newdns.lab. Однако по заданию client1  должен видеть запись web1.dns.lab и не видеть запись web2.dns.lab. Client2 может видеть обе записи из домена dns.lab, но не должен видеть записи домена newdns.lab Осуществить данные настройки нам поможет технология Split-DNS.  

Для настройки Split-DNS нужно: 
### 1) Создать дополнительный файл зоны dns.lab, в котором будет прописана только одна запись: vim /etc/named/named.dns.lab.client
<pre>
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

;Web
web1            IN      A       192.168.50.15
</pre>
Имя файла может отличаться от указанной зоны. У файла должны быть права 660, владелец — root, группа — named.  

### 2) Внести изменения в файл /etc/named.conf на хостах ns01 и ns02

Прежде всего нужно сделать access листы для хостов client и client2. Сначала сгенерируем ключи для хостов client и client2, для этого на хосте ns01 запустим утилиту tsig-keygen (ключ может генериться 5 минут и более): 
<pre>
[root@ns01 named]# tsig-keygen
key "tsig-key" {
	algorithm hmac-sha256;
	secret "FGJU2PpozNkpz325eXYXlw+Gu7cE+GwZj1hf47q8BJI=";
};
[root@ns01 named]# tsig-keygen
key "tsig-key" {
	algorithm hmac-sha256;
	secret "zhyNOkCkoSpbafY7OS3t+6GpEGECqncsJsA2KViJdzo=";
};
</pre>

После их генерации добавим блок с access листами в конец файла /etc/named.conf
<pre>
#Описание ключа для хоста client
key "client-key" {
    algorithm hmac-sha256;
    secret "FGJU2PpozNkpz325eXYXlw+Gu7cE+GwZj1hf47q8BJI=";
};
#Описание ключа для хоста client2
key "client2-key" {
    algorithm hmac-sha256;
    secret "zhyNOkCkoSpbafY7OS3t+6GpEGECqncsJsA2KViJdzo=";
};
#Описание access-листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
</pre>

В данном блоке access листов мы выделяем 2 блока: 
client имеет адрес 192.168.50.15, использует client-key и не использует client2-key
client2 имеет адрес 192ю168.50.16, использует clinet2-key и не использует client-key

Описание ключей и access листов будет одинаковое для master и slave сервера.

Далее нужно создать файл с настройками зоны dns.lab для client, для этого на мастер сервере создаём файл /etc/named/named.dns.lab.client и добавляем в него следующее содержимое:
<pre>
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201409 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11
;Web
web1            IN      A       192.168.50.15

</pre>


Это почти скопированный файл зоны dns.lab, в конце которого удалена строка с записью web2. Имя зоны надо оставить такой же — dns.lab


Теперь можно внести правки в /etc/named.conf

Технология Split-DNS реализуется с помощью описания представлений (view), для каждого отдельного acl. В каждое представление (view) добавляются только те зоны, которые разрешено видеть хостам, адреса которых указаны в access листе.

Все ранее описанные зоны должны быть перенесены в модули view. Вне view зон быть недолжно, зона any должна всегда находиться в самом низу. 

После применения всех вышеуказанных правил на хосте ns01 мы получим следующее содержимое файла /etc/named.conf
<pre>
options {

    // На каком порту и IP-адресе будет работать служба 
	listen-on port 53 { 192.168.50.10; };
	listen-on-v6 port 53 { ::1; };

    // Указание каталогов с конфигурационными файлами
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";

    // Указание настроек DNS-сервера
    // Разрешаем серверу быть рекурсивным
	recursion yes;
    // Указываем сети, которым разрешено отправлять запросы серверу
	allow-query     { any; };
    // Каким сетям можно передавать настройки о зоне
    allow-transfer { any; };
    
    // dnssec
	dnssec-enable yes;
	dnssec-validation yes;

    // others
	bindkeys-file "/etc/named.iscdlv.key";
	managed-keys-directory "/var/named/dynamic";
	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.50.10 allow { 192.168.50.15; 192.168.50.16; } keys { "rndc-key"; }; 
};

key "client-key" {
    algorithm hmac-sha256;
    secret "FGJU2PpozNkpz325eXYXlw+Gu7cE+GwZj1hf47q8BJI=";
};
key "client2-key" {
    algorithm hmac-sha256;
    secret "zhyNOkCkoSpbafY7OS3t+6GpEGECqncsJsA2KViJdzo=";
};

// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key"; 

server 192.168.50.11 {
    keys { "zonetransfer.key"; };
};
// Указание Access листов 
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
// Настройка первого view 
view "client" {
    // Кому из клиентов разрешено подключаться, нужно указать имя access-листа
    match-clients { client; };

    // Описание зоны dns.lab для client
    zone "dns.lab" {
        // Тип сервера — мастер
        type master;
        // Добавляем ссылку на файл зоны, который создали в прошлом пункте
        file "/etc/named/named.dns.lab.client";
        // Адрес хостов, которым будет отправлена информация об изменении зоны
        also-notify { 192.168.50.11 key client-key; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "/etc/named/named.newdns.lab";
        also-notify { 192.168.50.11 key client-key; };
    };
};

// Описание view для client2
view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab";
        also-notify { 192.168.50.11 key client2-key; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "/etc/named/named.dns.lab.rev";
        also-notify { 192.168.50.11 key client2-key; };
    };
};

// Зона any, указана в файле самой последней
view "default" {
    match-clients { any; };

    // root zone
    zone "." IN {
        type hint;
        file "named.ca";
    };

    // zones like localhost
    include "/etc/named.rfc1912.zones";
    // root DNSKEY
    include "/etc/named.root.key";

    // dns.lab zone
    zone "dns.lab" {
        type master;
        allow-transfer { key "zonetransfer.key"; };
        file "/etc/named/named.dns.lab";
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        allow-transfer { key "zonetransfer.key"; };
        file "/etc/named/named.dns.lab.rev";
    };

    // ddns.lab zone
    zone "ddns.lab" {
        type master;
        allow-transfer { key "zonetransfer.key"; };
        allow-update { key "zonetransfer.key"; };
        file "/etc/named/named.ddns.lab";
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        allow-transfer { key "zonetransfer.key"; };
        file "/etc/named/named.newdns.lab";
    };
};

</pre>

Далее внесем изменения в файл /etc/named.conf на сервере ns02. Файл будет похож на файл, лежащий на ns01, только в настройках будет указание забирать информацию с сервера ns01:
<pre>
options {

    // network 
	listen-on port 53 { 192.168.50.11; };
	listen-on-v6 port 53 { ::1; };

    // data
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
	recursion yes;
	allow-query     { any; };
    allow-transfer { any; };

    // dnssec
	dnssec-enable yes;
	dnssec-validation yes;

    // others
	bindkeys-file "/etc/named.iscdlv.key";
	managed-keys-directory "/var/named/dynamic";
	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.50.11 allow { 192.168.50.15; 192.168.50.16; } keys { "rndc-key"; };
};

key "client-key" {
    algorithm hmac-sha256;
    secret "FGJU2PpozNkpz325eXYXlw+Gu7cE+GwZj1hf47q8BJI=";
};
key "client2-key" {
    algorithm hmac-sha256;
    secret "zhyNOkCkoSpbafY7OS3t+6GpEGECqncsJsA2KViJdzo=";
};
// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key";
server 192.168.50.10 {
   keys { "zonetransfer.key"; };
};
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };

view "client" {
    match-clients { client; };
    allow-query { any; };
    // dns.lab zone
    zone "dns.lab" {
 
    type slave;

    masters { 192.168.50.10 key client-key; };
};
// newdns.lab zone
zone "newdns.lab" {
type slave;
masters { 192.168.50.10 key client-key; };
   };
};
view "client2" {
   match-clients { client2; };
   // dns.lab zone
   zone "dns.lab" {
   type slave;
   masters { 192.168.50.10 key client2-key; };
};
// dns.lab zone reverse
zone "50.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.50.10 key client2-key; };
   };
};
view "default" {
match-clients { any; };
// root zone
zone "." IN {
type hint;
file "named.ca";
};
// zones like localhost
include "/etc/named.rfc1912.zones";
// root DNSKEY
include "/etc/named.root.key";
// dns.lab zone
zone "dns.lab" {
   type slave;
   masters { 192.168.50.10; };
};
// dns.lab zone reverse
zone "50.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.50.10; };
};
// ddns.lab zone
zone "ddns.lab" {
    type slave;
    masters { 192.168.50.10; };
};
// newdns.lab zone
    zone "newdns.lab" {
    type slave;
    masters { 192.168.50.10; };
};
};
</pre>


Так как файлы с конфигурациями получаются достаточно большими — возрастает вероятность сделать ошибку. При их правке можно воспользоваться утилитой named-checkconf. Она укажет в каких строчках есть ошибки. Использование данной утилиты рекомендуется после изменения настроек на DNS-сервере. 

После внесения данных изменений можно перезапустить (по очереди) службу named на серверах ns01 и ns02.

Далее, нужно будет проверить работу Split-DNS с хостов client и client2. Для проверки можно использовать утилиту ping:

Проверка на client:
<pre>
[vagrant@client etc]$ ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.007 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.026 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.028 ms
^C
--- www.newdns.lab ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.007/0.020/0.028/0.010 ms
[vagrant@client etc]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.008 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.027 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.029 ms
^C
--- web1.dns.lab ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.008/0.021/0.029/0.010 ms
[vagrant@client etc]$ ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
[vagrant@client etc]$ 
</pre>

На хосте мы видим, что client видит обе зоны (dns.lab и newdns.lab), однако
информацию о хосте web2.dns.lab он получить не может.

Проверка на client2: 
<pre>
[root@client2 ~]# ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[root@client2 ~]# 
[root@client2 ~]# ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=0.809 ms
^C
--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.809/0.809/0.809/0.000 ms
[root@client2 ~]# ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from client2 (192.168.50.16): icmp_seq=2 ttl=64 time=0.065 ms
^C
--- web2.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1038ms
rtt min/avg/max/mdev = 0.037/0.051/0.065/0.014 ms
[root@client2 ~]# 
</pre>
Тут мы понимаем, что client2 видит всю зону dns.lab и не видит зону newdns.lab


























