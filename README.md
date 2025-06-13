---[МОДУЛЬ 1](#Модуль-1)-----[МОДУЛЬ 2](#Модуль-2)-----[МОДУЛЬ 3](#Модуль-3)

# Модуль 1
### 1. Базовая настройка устройств 
Включает в себя: наименование устройств полным доменным именем;

!!! HQ-RTR, BR-RTR и не забыть хвост .au-team.irpo

```
hostnamectl set-hostname имя_устройства; exec bash
```

конфигурацию IPv4 согласно схеме:

| Имя устройства | IP-адрес | Шлюз по умолчанию |
| -------------- | -------- | ----------------- |
| ISP to HQ | 172.16.4.1/28 | - |
| ISP to BR | 172.16.5.1/28 | - |
| HQ-RTR | 172.16.4.2/28 | 172.16.4.1 |
| BR-RTR | 172.16.5.2/28 | 172.16.5.1 |
| HQ in | 172.16.0.1/28 | - |
| BR in | 172.16.6.1/27 | - |
| HQ-SRV | 172.16.0.2/28 | 172.16.0.1 |
| BR-SRV | 172.16.6.2/27 | 172.16.6.1 |
| HQ-CLI | 172.16.0.3/28 | 172.16.0.1 |

!!! Заносим в отчет необходимые данные. Проверяем сетевую связанность с помощью ping.

### 2. Создание локальных учетных записей 
На серверах HQ-SRV и BR-SRV: пользователь sshuser, пароль P@ssw0rd, id 1010

```
useradd -m -u 1010 sshuser
passwd sshuser
nano /etc/sudoers
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
ctrl+x
y
enter
usermod -aG wheel sshuser
```

На роутерах HQ-RTR и BR-RTR: пользователь net_admin, пароль P@$$word

```
useradd -m net_admin
passwd net_admin
nano /etc/sudoers
net_admin ALL=(ALL:ALL)NOPASSWD:ALL
ctrl+x
y
enter
usermod -aG wheel net_admin
```

### 3. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:
```
nano /etc/mybanner
в файл: Authorized access only!
ctrl+x
y
enter

nano /etc/openssh/sshd_config
port 2024
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
ctrl+x
y
enter

systemctl restart sshd.service
```
!!! Обязательно проверяем, в том числе на правильность паролей.

Возможно на этом моменте можно сделать снапшоты: ISP, HQ-RTR, BR-RTR, HQ-SRV, BR-SRV
### 4. Между офисами HQ и BR необходимо сконфигурировать ip туннель
Перед настройкой: проверяем отключен ли на всех роутерах firewall, если нет - отключаем. Включаем на всех роутерах ip forwarding.

```
nano /etc/net/sysctl.conf
net.ipv4.ip_forward = 1
```

Настраиваем GRE через nmtui

HQ-RTR:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/hq.png">

BR-RTR:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/br.png">

!!! проводим проверку сетевой связанности. Если тунель не поднялся с первого раза, пока оставляем как есть, чтобы ребутнуть после настройки OSPF.

### 5. Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте link state протокол на ваше усмотрение

Настраиваем OSPF

```
nano /etc/frr/daemons
ospfd=yes
```


```
systemctl enable --now frr
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/28 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```

!!! Проводим данную настройку на HQ-RTR и BR-RTR проверяем на правильность сетевых адресов.

Обоснование выбора OSPF: В качестве link state протокола динамической маршрутизации был выбран OSPF, поскольку данный протокол полноценно интегрируется в среду ALT linux, при изменении сети обеспечивает динамическое изменение маршрутов и адекватную скорость при первичном построении маршрута. 

```
vtysh
sh ip ospf ne
```

!!! Смотрим соседей. Если еще не отображается ничего - reboot. Проверяем сетевую связанность. Если все работает заряженный снапшот тоже делаем. Насчет снапшотов "Патимат сейчас" ничего не смогла придумать, поэтому "Патимат в день икс" смотри сама, ты - изобретательна :) NO PANIC. Возможно можно оставить сделать снапшоты в конце 1 модуля.

### 6. Настройка протокола динамической конфигурации хостов 
Для офиса HQ сервером DHCP выступает HQ-RTR. Клиентом является машина HQ-CLI

```
nano /etc/sysconfig/dhcpd
DHCPARGS=ens35
ctrl+x
y
enter

cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
```

```
option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;

authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
        range 172.16.0.3 172.16.0.8;
        option routers 172.16.0.1;
}
```

```
ctrl+x
y
enter
systemctl enable --now dhcpd
```

### 7. Настройка DNS для офисов HQ и BR
Настройки прводятся на HQ-SRV:

```
nano /etc/bind/options.conf
```

Меняем выделенные строки:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/меняем%20строки%20в%20бинде.png"/>

```
systemctl enable --now bind
nano /etc/bind/local.conf
```

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/локалконф%20днс.png"/>

```
cd /etc/bind/zone
cp localdomain au.db
cp 127.in-addr.arpa 0.db
chown root:named {au,0}.db
nano au.db
```

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/audb.png"/>

```
nano 0.db
```

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/0db.png"/>

```
systemctl restart bind
```

Проверка:

```
host hq-rtr.au-team.irpo
```

Должен выдать IP-адрес

### 8. Настройка часового пояса согласно месту проведения демо

```
timedatectl set-timezone Europe/Moscow
```

!!! Если до этого момента не был сделан снапшот имеет смысл сделать его сейчас на всех задействованных машинах.

# Модуль 2
### 9. Настройте доменный контроллер Samba на машине HQ-SRV





# Модуль 3
