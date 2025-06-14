### ОГЛАВЛЕНИЕ

[МОДУЛЬ 1](#Модуль-1)                       |  [МОДУЛЬ 2](#Модуль-2)                      |  [МОДУЛЬ 3](#Модуль-3)

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

[назад](#ОГЛАВЛЕНИЕ)
### 9. Настройте доменный контроллер Samba на машине HQ-SRV

Перед началом настройки отключаем все мосты, если они есть и лишние интерфейсы.

```
control bind-chroot disabled
grep -q 'bind-dns' /etc/bind/named.conf || echo 'include "/var/lib/samba/bind-dns/named.conf";' >> /etc/bind/named.conf
nano /etc/bind/options.conf
```

Вносим в файл следующие строки:

```
tkey-gssapi-keytab "/var/lib/samba/bind-dns/dns.keytab";
minimal-responses yes;

category lame-servers {null;};
```

как на фото:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/options.png"/>

```
systemctl stop bind
nano /etc/sysconfig/network
```

Прописываем хостнейм: HQ-SRV.au-team.irpo

```
domainname au-team.irpo
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

Проверяем есть ли необходимые записи в /etc/resolv.conf. Если нет вносим согласно картинке:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/5285432947287127148.jpg"/>


```
samba-tool domain provision
```

Вносим данные по требованию: BIND9_DLZ, P@ssw0rd

```
nano /etc/bind/named.conf
```

Комментируем строку как на фото:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/5287485572287425229.jpg"/>

```
systemctl enable --now samba
systemctl enable --now bind
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
nano /etc/krb5.conf
```

Проверяем на наличие необходимых записей как на фото:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/5285432947287127149.jpg"/>

Проверка:

```
samba-tool domain info 127.0.0.1
host -t SRV _kerberos._udp.au-team.irpo.
host -t SRV _ldap._tcp.au-team.irpo.
host -t A hq-srv.au-team.irpo
```

```
kinit administrator@AU-TEM.IRPO
```

На приглашение ввода пароля пишем: P@ssw0rd

Ввод машны в домен. Перед вводом указываем в параметрах сети адрес днс: 172.16.0.2 и поисковой домен: au-team.irpo

На HQ-CLI:

```
kinit administrator@AU-TEM.IRPO
acc
Аутентификация
```
Далее как на фото, не забывая галочки:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/5287485572287425247.jpg"/>

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/5287485572287425257.jpg"/>

Для создания пользователей: в открывшимся окне разворачиваем au-team.irpo, открываем вкладку users и создаем пользователей. Имена пользователей формата user№.hq

```
admc
```

### 10. Сконфигурируйте файловое хранилище: 

Проверяем наличие дисков: 

```
lsblk
```

Создание RAID:

```
mdadm --create --level=5 --raid-devices=3 /dev/md0 /dev/sdb /dev/sdc /dev/sdd
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
cat /etc/mdadm/mdadm.conf
mkfs.ext4 /dev/md0
```

Чтобы данный раздел также монтировался при загрузке системы, добавляем в fstab.

```
nano /etc/fstab
mkdir /root/raid5

/dev/md0        /root/raid5 ext4        defaults        0        0

mount –a
df –h
```

## Настройка nfs:
Создаём директорию для общего доступа в директории /root/raid5, куда ранее был смонтирован RAID

```
mkdir /root/raid5/nfs
chmod 777 /root/raid5/nfs
```

Производим конфигурацию NFS. В файл /etc/exports вносим строку:

```
/root/raid5/nfs 172.16.0.0/26(rw,subtree_check,no_root_squash)
```

как на фото:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/Рисунок2.png"/>

Экспортируем файловую систему, указанную выше в /etc/exports

```
exportfs –arv
systemctl enable --now nfs-server
```

Продолжаем на HQ-CLI:

Создадим директорию для монтирования общего ресурса:

```
mkdir /mnt/nfs
chmod 777 /mnt/nfs
```
Настраиваем автомонтирование общего ресурса через fstab:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/автомонтирование%20через%20fstab.png"/>

172.16.0. не 1, а 2, т.е 172.16.0.2 - адрес файлового сервера.

```
mount –a
df –h
```

Проверка, пробуем создать файл: 

```
touch /mnt/nfs/TEST
```

Проверяем наличие файла на сервере:

```
ls –l  /root/raid5/nfs
```

### 11. Сконфигурируйте ansible на сервере BR-SRV :

Заходим в файл /etc/ansible/hosts и заносим данные о хостах:

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/инвентарь%20энсибл.png"/>

С помощью следующей команды генерируем ssh-ключ:

```
ssh-keygen –C “$(whoami)@$(hostname)-$(date –I)”
```

На всех клиентах Ansible кроме HQ-SRV необходимо в файле /etc/openssh/sshd_conf раскомментировать строку Port 22 и прописать AllowUsers user. Обязательно перезапустить демон, а там где он раннее не был включен включить его --now:

```
systemctl restart sshd
```

Далее копируем этот ключ по  ssh на все клиенты, которые должны входить в инвентарь: ssh-copy-id имя_пользователя_из_файла_hosts@ip_адрес_из_того_же_файла

```
ssh-copy-id sshuser@172.16.0.2 –p 2024
ssh-copy-id user@172.16.0.3
ssh-copy-id user@172.16.0.1
ssh-copy-id user@172.16.5.2
```

Отправить ping:

```
ansible all –m ping 
```

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/pong.png"/>

Чтобы предупреждение не вылезало прописываем:

```
/etc/ansible/ansible.cfg
```

<img src="https://github.com/fsalikhovaa/demo2025/blob/main/Рисунок6.png"/>

# Модуль 3 

[назад](#ОГЛАВЛЕНИЕ)

### 12. Реализуйте логирование при помощи rsyslog на устройствах HQ-RTR, BR-RTR, BR-SRV:

Сервером сбора логов является HQ-SRV. На HQ-SRV:

```
nano /etc/rsyslog.d/00_common.conf
```

Нужно раскомментировать модули imudp и imtcp чтобы rsyslog мог получать логи с удаленных узлов, и создать шаблон сбора логов с клиентов (мы добавим его в самый низ). Окончательный файл будет выглядеть так:

<img src="https://github.com/fsalikhovaa/D2025/blob/main/рсислог%20хк-ртр.png"/>

Шаблон в виде текста:

```
$template RemoteLogs, "/opt/%HOSTNAME%.log"
*.* ?RemoteLogs
& ~
```

```
systemctl enable --now rsyslog
```

Настройка клиентов HQ-RTR, BR-RTR, BR-SRV:

Выполняем на всех поочередно действия:

```
nano /etc/rsyslog.d/00_common.conf
```

Раскоменчиваем первые 4 модуля, которые начнут собирать логи:

<img src="https://github.com/fsalikhovaa/D2025/blob/main/рсислог%20клиент.png"/>

```
systemctl enable --now rsyslog
```

Проверям наличие логов: 

<img src="https://github.com/fsalikhovaa/D2025/blob/main/логи%20в%20опт.png"/>

### Ротация логов

Создайте /etc/logrotate.d/rsyslog:

```
nano /etc/logrotate.d/rsyslog

/opt/*.log {
    weekly
    size 10M
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    sharedskripts
    postrotate
        /usr/bin/systemctl reload rsyslog > /dev/null 2>&1 || true
    endscript
}
```

<img src="https://github.com/fsalikhovaa/D2025/blob/main/5357130940893229674.jpg"/>

```
nano /etc/crontab

0 0 * * 0 /usr/sbin/logrotate -f /etc/logrotate.d/rsyslog
```

<img src="https://github.com/fsalikhovaa/D2025/blob/main/5357130940893229673.jpg"/>

### Реализуйте механизм инвентаризации машин HQ-SRV и HQ-CLI через Ansible на BR-SRV

```
mkdir /etc/ansible/PC_INFO
cd /etc/ansible/PC_INFO
nano playbook.yml
```

<img src="https://github.com/fsalikhovaa/D2025/blob/main/5357130940893229626.jpg"/>

Текстом:

```
- name: PC_INFO
  hosts:
    - hq-srv
    - hq-cli
  tasks:
  - name: create file and append hostname
    lineinfile:
      path: /etc/ansible/PC_INFO/{{ ansible_hostname }}.yml
      line: "Имя компьютера: {{ ansible_hostname }} \n"
      create: true
    delegate_to: 127.0.0.1

  - name: append ip address to file
    lineinfile:
      path: /etc/ansible/PC_INFO/{{ ansible_hostname }}.yml
      line: "IP address: {{ ansible_default_ipv4.address }} \n"
      create: true
    delegate_to: 127.0.0.1
```

```
ansible-playbook /etc/ansible/PC_INFO/playbook.yml
```
