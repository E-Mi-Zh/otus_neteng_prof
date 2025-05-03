# Домашнее задание №3.1 «Реализация DHCPv4»

## Топология

![Топология](topology.drawio.png)

## 1. Таблица адресации

| Устройство | Интерфейс | IP-адрес | Маска подсети   | Шлюз по умолчанию |
| ---------- | --------- | -------- | --------------- | ----------------- |
| R1         | e0/0      | 10.0.0.1 | 255.255.255.252 | N/A               |
|            | e0/1      | N/A      | N/A             |                   |
|            | e0/1.100  |          |                 |                   |
|            | e0/1.200  |          |                 |                   |
|            | e0/1.1000 | N/A      | N/A             |                   |
| R2         | e0/0      | 10.0.0.2 | 255.255.255.252 | N/A               |
|            | e0/1      |          |                 |                   |
| S1         | VLAN 200  |          |                 |                   |
| S2         | VLAN 1    |          |                 |                   |
| PC-A       | NIC       | DHCP     | DHCP            | DHCP              |
| PC-A       | NIC       | DHCP     | DHCP            | DHCP              |

## 2. Таблица VLAN

| VLAN | Имя         | Назначенный интерфейс |
| ---- | ----------- | --------------------- |
| 1    | N/A         | S2: e0/3              |
| 100  | Clients     | S1: e0/2              |
| 200  | Management  | S1: VLAN 200          |
| 999  | Parking_Lot | S1: e0/0, e0/3        |
| 1000 | Native      | N/A                   |

## 3. Задачи

* [Часть 1. Создание сети и настройка основных параметров устройства.](#часть-1-создание-сети-инастройка-основных-параметров-устройства)
* [Часть 2. Настройка и проверка двух серверов DHCPv4 на R1.](#часть-2-настройка-и-проверка-двух-серверов-dhcpv4-на-r1)
* [Часть 3. Настройка и проверка DHCP-ретрансляции на R2.](#часть-3-настройка-и-проверка-dhcp-ретрансляции-на-r2)

## Общие сведения/сценарий

Протокол динамической конфигурации сетевого узла (DHCP) — сетевой протокол,
позволяющий сетевым администраторам управлять и автоматизировать назначение
IP-адресов. Без использования DHCP  для IPv4 администратору необходимо вручную
назначать и настраивать IP-адреса, предпочтительные DNS-серверы и шлюзы по
умолчанию. По мере увеличения сети и перемещении устройств из одной внутренней
сети в другую это становится административной проблемой.

В предложенном сценарии размеры компании увеличились, и сетевые администраторы
больше не имеют возможности назначать IP-адреса для устройств вручную. Наша
задача заключается в настройке маршрутизатора **R1** для назначения IPv4-адресов в
двух разных подсетях.

**Примечание:** маршрутизаторы, используемые в практических лабораторных работах
CCNA, - это Cisco 4221 с Cisco IOS XE Release 16.9.3 (образ universalk9). В
лабораторных работах используются коммутаторы Cisco Catalyst 2960 с Cisco IOS
версии 15.0(2) (образ lanbasek9). Можно использовать другие маршрутизаторы,
коммутаторы и версии Cisco IOS. В зависимости от модели устройства и версии Cisco
IOS доступные команды и результаты их выполнения могут отличаться от тех, которые
показаны в лабораторных работах. Правильные идентификаторы интерфейса см. в
сводной таблице по интерфейсам маршрутизаторов в конце лабораторной работы.

**Примечание:** убедитесь, что все настройки коммутаторов удалены и загрузочная
конфигурация отсутствует. Если вы не уверены, обратитесь к инструктору.

**Примечание:** данная лабораторная работа, как и другие в этом курсе выполнялась
в среде EVE-NG. Названия интерфейсов изменены на фактически используемые (вместо
F0/0 или G0/0 используются e0/0 и т.п.).

## Часть 1. Создание сети и настройка основных параметров устройства

Для моделирования сети будем использовать ПО EVE-NG 5.0.1-24. Создадим
новую конфигурацию, используя следующие ресурсы:

* 2 маршрутизатора (Cisco IOL, образ L3-ADVENTERPRISEK9-M-15.4-2T);
* 2 коммутатора (Cisco IOL, образ L2-ADVENTERPRISEK9-M-15.2-20150703);
* 2 ПК (Eve-NG Virtual PC).

В первой части лабораторной работы создадим топологию сети и настроим базовые
параметры для узлов ПК и коммутаторов.

### Шаг 1. Создание схемы адресации

Разобьём сеть 192.168.1.0/24 на подсети в соответствии со следующими требованиями:

#### a. «Подсеть А»

Клиентская VLAN на **R1**: 58 хостов.

Запишем первый IP-адрес в таблице адресации для **R1** e0/1.100: 192.168.1.1,
подсеть 192.168.1.0/26 (62 хоста, маска 255.255.255.192).

#### b. «Подсеть B»

Управляющая VLAN на **R1**: 28 хостов.

Запишем первый IP-адрес в таблице адресации для **R1** e0/1.200. Запишем второй
IP-адрес в таблице адресов для **S1** VLAN 200 и введём соответствующий шлюз по
умолчанию.

Необходимая подсеть: 192.168.1.64/27 (30 хостов, маска 255.255.255.224), первый
IP 192.168.1.65, второй - 192.168.1.66.

#### c. «Подсеть C»

Клиентская сеть на **R2**: 12 узлов.

Запишем первый IP-адрес в таблице адресации для **R2** e0/1: 192.168.1.97.
Подсеть 192.168.1.96/28 (14 хостов, маска 255.255.255.240).

### Шаг 2. Создание сети

Подключим устройства, как показано в топологии, и подсоединим необходимые кабели.

![Топология в CPT](topo_eveng_1.png)

### Шаг 3. Базовая настройка маршрутизаторов

Настроим базовые параметры каждого маршрутизатора.

#### a. Установка имени устройства

Подключимся к маршрутизатору с помощью консольного подключения, активируем
привилегированный режим и сменим имя:

```text
Router>en
Router#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#host R1
R1(config)#
```

#### b. Отключение поиска DNS

Чтобы предотвратить попытки маршрутизатора неверно преобразовывать введённые
команды таким образом, как будто они являются именами узлов, отключим поиск DNS:

```text
R1(config)#no ip domain-lookup
R1(config)#
```

#### c. Установка пароля привилегированного режима

Назначим **class** в качестве зашифрованного пароля привилегированного режима EXEC:

```text
R1(config)#enable secret class
R1(config)#
```

#### d. Установка пароля консоли

Назначим **cisco** в качестве пароля консоли и включим вход в систему
по паролю:

```text
R1(config)#line con 0
R1(config-line)#password cisco
R1(config-line)#login
R1(config-line)#exit
R1(config)#
```

#### e. Установка пароля VTY

Назначим **cisco** в качестве пароля VTY каналов и включим вход в систему
по паролю:

```text
R1(config)#line vty 0 4
R1(config-line)#password cisco
R1(config-line)#login
R1(config-line)#exit
R1(config)#
```

#### f. Включение шифрования паролей

Зашифруем открытые пароли в файле конфигурации:

```text
R1(config)#service password-encryption
R1(config)#
```

#### g. Создание баннера

Для предупреждения пользователей о запрете несанкционированного доступа, настроим
баннерное сообщение дня (MOTD):

```text
R1(config)#banner motd # ATTENTION! Unauthorized access is strictly prohibited. #
R1(config)#
```

#### h. Сохранение конфигурации

Скопируем текущую конфигурацию в файл загрузочной конфигурации.

```text
R1(config)#exit
R1#copy run start
Destination filename [startup-config]? 
Building configuration...
[OK]
R1#
```

<details>
<summary>R1# show running-config</summary>

```text
R1#sh run
Building configuration...

Current configuration : 1067 bytes
!
! Last configuration change at 12:04:41 UTC Fri May 2 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$NAN2$3L0xiFyz9IMRzzyHwxOgA.
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!         
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!         
!
!
!
!
redundancy
!
!
! 
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 no ip address
 shutdown 
!
interface Ethernet0/1
 no ip address
 shutdown
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
!
!
!
!
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 094F471A1A0A
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 01100F175804
 login
 transport input none
!
!
end

R1#
```

</details>

<details>
<summary>R2# show running-config</summary>

```text
R2#sh run
Building configuration...

Current configuration : 1067 bytes
!
! Last configuration change at 12:05:05 UTC Fri May 2 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R2
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$QFFr$fapdzxpn96wsv4bWeTFnM/
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!         
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!         
!
!
!
!
redundancy
!
!
! 
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 no ip address
 shutdown 
!
interface Ethernet0/1
 no ip address
 shutdown
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
!
!
!
!
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 070C285F4D06
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 00071A150754
 login
 transport input none
!
!
end

R2#
```

</details>

#### i. Установка времени

Для установки времени и даты воспользуемся командой **clock set**. Просмотреть
текущее время и дату можно командой **show clock**.

```text
R1#clock set 22:07:00 02 May 2025
R1#
*May  2 22:07:00.000: %SYS-6-CLOCKUPDATE: System clock has been updated from 12:07:22 UTC Fri May 2 2025 to 22:07:00 UTC Fri May 2 2025, configured from console by console.
R1#

```

### Шаг 4. Настройка маршрутизации между сетями VLAN

Настроим маршрутизацию между сетями VLAN на маршрутизаторе **R1**.

#### a. Активация интерфейса

Активируем интерфейс e0/1 на маршрутизаторе **R1**.

```text
R1(config)#int e0/1
R1(config-if)#no shut
R1(config-if)#
May  2 22:58:28.505: %LINK-3-UPDOWN: Interface Ethernet0/1, changed state to up
May  2 22:58:29.510: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
R1(config-if)#exit
R1(config)#
```

#### b. Настройка подинтерфейсов

Настроим подинтерфейсы для каждой VLAN в соответствии с требованиями таблицы
IP-адресации. Все субинтерфейсы используют инкапсуляцию 802.1Q, им назначается
первый полезный адрес из вычисленного пула IP-адресов. Убедимся, что подинтерфейсу
для native VLAN не назначен IP-адрес. Включим описание для каждого подинтерфейса.

```text
R1(config)#int e0/1.100
R1(config-subif)#desc Clients
R1(config-subif)#en
R1(config-subif)#enc dot1Q 100 
R1(config-subif)#ip add 192.168.1.1 255.255.255.192
R1(config-subif)#exit
R1(config)#int e0/1.200
R1(config-subif)#desc Management
R1(config-subif)#enc dot1Q 200
R1(config-subif)#ip add 192.168.1.65 255.255.255.224
R1(config-subif)#exit
R1(config)#int e0/1.999                       
R1(config-subif)#desc Parking_Lot
R1(config-subif)#enc dot1Q 999
R1(config-subif)#exit
R1(config)#int e0/1.1000
R1(config-subif)#desc Native
R1(config-subif)#enc dot1Q 1000 native
R1(config-subif)#exit
R1(config)#
```

#### c. Проверка работы подинтерфейсов

Убедимся, что вспомогательные интерфейсы работают.

```text
R1#show interfaces description 
Interface                      Status         Protocol Description
Et0/0                          admin down     down     
Et0/1                          up             up       
Et0/1.100                      up             up       Clients
Et0/1.200                      up             up       Management
Et0/1.999                      up             up       Parking_Lot
Et0/1.1000                     up             up       Native
Et0/2                          admin down     down     
Et0/3                          admin down     down     
R1#show ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                unassigned      YES unset  administratively down down    
Ethernet0/1                unassigned      YES unset  up                    up      
Ethernet0/1.100            192.168.1.1     YES manual up                    up      
Ethernet0/1.200            192.168.1.65    YES manual up                    up      
Ethernet0/1.999            unassigned      YES unset  up                    up      
Ethernet0/1.1000           unassigned      YES unset  up                    up      
Ethernet0/2                unassigned      YES unset  administratively down down    
Ethernet0/3                unassigned      YES unset  administratively down down    
R1#
```

### Шаг 5. Настройка статической маршрутизации

Настроим e0/1 на **R2**, затем e0/0 и статическую маршрутизацию для обоих
маршрутизаторов.

#### a. Настройка e0/1 на R2

Настроим e0/1 на **R2** с первым IP-адресом подсети C, рассчитанным ранее.

```text
R2(config)#int e0/1
R2(config-if)#ip add 192.168.1.97 255.255.255.240
R2(config-if)#no shut
R2(config-if)#
May  2 23:51:27.379: %LINK-3-UPDOWN: Interface Ethernet0/1, changed state to up
May  2 23:51:28.379: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
R2(config-if)#exit
R2(config)#
```

#### b. Настройка e0/0 на маршрутизаторах

Настроим интерфейс e0/0 для каждого маршрутизатора на основе приведенной выше
таблицы IP-адресации.

```text
R1(config)#int e0/0
R1(config-if)#ip add 10.0.0.1 255.255.255.252
R1(config-if)#no shut
R1(config-if)#
May  2 23:52:20.427: %LINK-3-UPDOWN: Interface Ethernet0/0, changed state to up
May  2 23:52:21.431: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to up
R1(config-if)#exit
R1(config)#
```

```text
R2(config)#int e0/0
R2(config-if)#ip add 10.0.0.2 255.255.255.252
R2(config-if)#no shut
R2(config-if)#
May  2 23:52:44.972: %LINK-3-UPDOWN: Interface Ethernet0/0, changed state to up
May  2 23:52:45.978: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to up
R2(config-if)#exit
R2(config)#
```

#### c. Настройка маршрутов по умолчанию

Настроим маршрут по умолчанию на каждом маршрутизаторе, указываемом на IP-адрес
e0/0 на другом маршрутизаторе.

```text
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2
R1(config)#
```

```text
R2(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.1
R2(config)#
```

#### d. Проверка работы маршрутизации

Убедимся, что статическая маршрутизация работает с помощью пинга до адреса
e0/1 **R2** от **R1**.

```text
R1#ping 10.0.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
R1#
```

#### e. Сохранение конфигурации

Сохраним текущую конфигурацию в файл загрузочной конфигурации.

```text
R1#copy run start
Destination filename [startup-config]? 
Building configuration...
[OK]
R1#
```

<details>
<summary>R1# show running-config</summary>

```text
R1#sh run
Building configuration...

Current configuration : 1554 bytes
!
! Last configuration change at 00:05:08 UTC Sat May 3 2025
! NVRAM config last updated at 00:05:45 UTC Sat May 3 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$NAN2$3L0xiFyz9IMRzzyHwxOgA.
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!         
!
!
!
!
!
redundancy
!
!
! 
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 ip address 10.0.0.1 255.255.255.252
!
interface Ethernet0/1
 no ip address
!
interface Ethernet0/1.100
 description Clients
 encapsulation dot1Q 100
 ip address 192.168.1.1 255.255.255.192
!
interface Ethernet0/1.200
 description Management
 encapsulation dot1Q 200
 ip address 192.168.1.65 255.255.255.224
!
interface Ethernet0/1.999
 description Parking_Lot
 encapsulation dot1Q 999
!
interface Ethernet0/1.1000
 description Native
 encapsulation dot1Q 1000 native
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 0.0.0.0 0.0.0.0 10.0.0.2
!
!
!
!
control-plane
!
!
!
!
!         
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 094F471A1A0A
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 01100F175804
 login
 transport input none
!
!
end

R1#
```

</details>


<details>
<summary>R2# show running-config</summary>

```text
R2#sh run
Building configuration...

Current configuration : 1188 bytes
!
! Last configuration change at 00:05:04 UTC Sat May 3 2025
! NVRAM config last updated at 00:06:39 UTC Sat May 3 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R2
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$QFFr$fapdzxpn96wsv4bWeTFnM/
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!         
!
!
!
!
!
redundancy
!
!
! 
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 ip address 10.0.0.2 255.255.255.252
!
interface Ethernet0/1
 ip address 192.168.1.97 255.255.255.240
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 0.0.0.0 0.0.0.0 10.0.0.1
!
!
!
!
control-plane
!
!
!
!
!
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 070C285F4D06
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 00071A150754
 login
 transport input none
!
!
end

R2#
```

</details>

### Шаг 6. Настройка базовых параметров коммутаторов

Настроим базовые параметры каждого коммутатора.

#### a. Установка имени устройства

Подключимся к коммутатору с помощью консольного подключения, активируем
привилегированный режим и сменим имя:

```text
Switch>en
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#host S1
S1(config)#
```

#### b. Отключение поиска DNS

Чтобы предотвратить попытки коммутатора неверно преобразовывать введённые
команды таким образом, как будто они являются именами узлов, отключим поиск DNS:

```text
S1(config)#no ip domain-lookup
S1(config)#
```

#### c. Установка пароля привилегированного режима

Назначим **class** в качестве зашифрованного пароля привилегированного режима EXEC:

```text
S1(config)#enable secret class
S1(config)#
```

#### d. Установка пароля консоли

Назначим **cisco** в качестве пароля консоли и включим вход в систему
по паролю:

```text
S1(config)#line con 0
S1(config-line)#password cisco
S1(config-line)#login
S1(config-line)#exit
S1(config)#
```

#### e. Установка пароля VTY

Назначим **cisco** в качестве пароля VTY каналов и включим вход в систему
по паролю:

```text
S1(config)#line vty 0 4
S1(config-line)#password cisco
S1(config-line)#login
S1(config-line)#exit
S1(config)#
```

#### f. Включение шифрования паролей

Зашифруем открытые пароли в файле конфигурации:

```text
S1(config)#service password-encryption
S1(config)#
```

#### g. Создание баннера

Для предупреждения пользователей о запрете несанкционированного доступа, настроим
баннерное сообщение дня (MOTD):

```text
S1(config)#banner motd # ATTENTION! Unauthorized access is strictly prohibited. #
S1(config)#
```

#### h. Сохранение конфигурации

Скопируем текущую конфигурацию в файл загрузочной конфигурации.

```text
S1#copy run start
Destination filename [startup-config]? 
Building configuration...
Compressed configuration from 903 bytes to 665 bytes[OK]
S1#
```

<details>
<summary>S1# show running-config</summary>

```text
S1#sh run
Building configuration...

Current configuration : 903 bytes
!
! Last configuration change at 14:30:54 UTC Fri May 2 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service compress-config
!
hostname S1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$B/PL$0B.kEHdic8i.OfuJwrwVS0
!
no aaa new-model
!
!
!
!         
!
!
!
!
no ip domain-lookup
ip cef
no ipv6 cef
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
! 
!
!
!
!
!
!
!
!         
!
!
!
interface Ethernet0/0
!
interface Ethernet0/1
!
interface Ethernet0/2
!
interface Ethernet0/3
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
!
!
!
!
!
control-plane
!         
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 1511021F0725
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 13061E010803
 login
!
!
end

S1#
```

</details>

<details>
<summary>S2# show running-config</summary>

```text
S2#sh run
Building configuration...

Current configuration : 903 bytes
!
! Last configuration change at 14:31:40 UTC Fri May 2 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service compress-config
!
hostname S2
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$hN1n$yH.Lsa8.AZQHUvGH2g1hJ/
!
no aaa new-model
!
!
!
!         
!
!
!
!
no ip domain-lookup
ip cef
no ipv6 cef
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
! 
!
!
!
!
!
!
!
!         
!
!
!
interface Ethernet0/0
!
interface Ethernet0/1
!
interface Ethernet0/2
!
interface Ethernet0/3
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
!
!
!
!
!
control-plane
!         
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 104D000A0618
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 094F471A1A0A
 login
!
!
end

S2#
```

</details>

#### i. Установка времени

Для установки времени и даты воспользуемся командой **clock set**. Просмотреть
текущее время и дату можно командой **show clock**.

**Примечание.** Вопросительный знак (?) позволяет открыть справку с правильной
последовательностью параметров, необходимых для выполнения этой команды.

```text
S1#clock set 01:35 04 Jan 2025
S1#show clock
1:35:3.467 UTC Sat Jan 4 2025
S1#
```

#### j. Сохранение конфигурации

Повторно сохраним конфигурацию коммутаторов.

### Шаг 7. Создание сетей VLAN на коммутаторе S1

Создадим и настроим сети VLAN на коммутаторе **S1**.

Примечание. **S2** настроен только с базовыми настройками.

#### a. Создание сетей VLAN

Создадим необходимые VLAN на коммутаторе **S1** и присвоим им имена из приведённой
выше таблицы.

```text
S1(config)#vlan 100
S1(config-vlan)#name Clients
S1(config-vlan)#exit
S1(config)#vlan 200
S1(config-vlan)#name Management
S1(config-vlan)#exit
S1(config)#vlan 999
S1(config-vlan)#name Parking_Lot
S1(config-vlan)#exit
S1(config)#vlan 1000
S1(config-vlan)#name Native
S1(config-vlan)#exit
S1(config)#
```

#### b. Настройка интерфейса управления на S1

Настроим и активируем интерфейс управления на **S1** (VLAN 200), используя второй
IP-адрес из подсети, рассчитанный ранее. Кроме того установим шлюз по умолчанию
на **S1**.

```text
S1(config)#int vlan 200
S1(config-if)#
May  3 00:33:24.909: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan200, changed state to down
S1(config-if)#ip add 192.168.1.66 255.255.255.224
S1(config-if)#no shut
S1(config-if)#
May  3 00:33:38.742: %LINK-3-UPDOWN: Interface Vlan200, changed state to down
S1(config-if)#exit
S1(config)#ip default-gateway 192.168.1.65
S1(config)#
```

#### c. Настройка интерфейса управления на S2

Настроим и активируем интерфейс управления на **S2** (VLAN 1), используя второй
IP-адрес из подсети, рассчитанный ранее. Кроме того, установим шлюз по умолчанию
на **S2**

```text
S2(config)#int vlan 1
S2(config-if)#
May  3 00:34:25.552: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan1, changed state to down
S2(config-if)#ip add 192.168.1.98 255.255.255.240
S2(config-if)#no shut
S2(config-if)#
May  3 00:34:46.561: %LINK-3-UPDOWN: Interface Vlan1, changed state to up
May  3 00:34:47.565: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan1, changed state to up
S2(config-if)#exit
S2(config)#ip default-gateway 192.168.1.97
S2(config)#
```

#### d. Деактивация неиспользуемых портов

Назначим все неиспользуемые порты **S1** VLAN Parking_Lot, настроим их для
статического режима доступа и административно деактивируем их. На **S2**
административно деактивируем все неиспользуемые порты.

**Примечание.** Команда interface range полезна для выполнения этой задачи с
минимальным количеством команд.

<details>
<summary>Коммутатор <strong>S1</strong></summary>

```text
S1(config)#int e0/0
S1(config-if)#sw m ac
S1(config-if)#sw ac vl 999
S1(config-if)#shut
S1(config-if)#
May  3 00:42:41.130: %LINK-5-CHANGED: Interface Ethernet0/0, changed state to administratively down
S1(config-if)#
May  3 00:42:42.134: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to down
S1(config-if)#exit
S1(config)#int e0/3
S1(config-if)#sw m ac
S1(config-if)#sw ac vl 999
S1(config-if)#shut
S1(config-if)#
May  3 00:43:05.892: %LINK-5-CHANGED: Interface Ethernet0/3, changed state to administratively down
May  3 00:43:06.897: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/3, changed state to down
S1(config-if)#exit
S1(config)#
```

</details>

<details>
<summary>Коммутатор <strong>S2</strong></summary>

```text
S2(config)#int e0/0
S2(config-if)#sw m ac
S2(config-if)#shut
S2(config-if)#
May  3 00:45:16.919: %LINK-5-CHANGED: Interface Ethernet0/0, changed state to administratively down
May  3 00:45:17.923: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to down
S2(config-if)#exit
S2(config)#int e0/2
S2(config-if)#sw m ac
S2(config-if)#shut
S2(config-if)#
May  3 00:45:34.649: %LINK-5-CHANGED: Interface Ethernet0/2, changed state to administratively down
May  3 00:45:35.653: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/2, changed state to down
S2(config-if)#exit
S2(config)#
```

</details>

### Шаг 8. Назначение сетей VLAN соответствующим интерфейсам коммутатора

#### a. Назначение используемых портов

Назначим используемые порты соответствующей VLAN (указанной в таблице VLAN выше)
и настроим их для режима статического доступа.

```text
S1(config)#int e0/2
S1(config-if)#sw m acc
S1(config-if)#sw acc vl 100
S1(config-if)#no shut
S1(config-if)#exit
S1(config)#
```

#### b. Проверка назначения портов

Убедимся, что VLAN назначены на правильные интерфейсы.

```text
S1#show vlan

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/1
100  Clients                          active    Et0/2
200  Management                       active    
999  Parking_Lot                      active    Et0/0, Et0/3
1000 Native                           active    
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 

VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
1    enet  100001     1500  -      -      -        -    -        0      0   
100  enet  100100     1500  -      -      -        -    -        0      0   
200  enet  100200     1500  -      -      -        -    -        0      0   
999  enet  100999     1500  -      -      -        -    -        0      0   
1000 enet  101000     1500  -      -      -        -    -        0      0   
1002 fddi  101002     1500  -      -      -        -    -        0      0   
1003 tr    101003     1500  -      -      -        -    -        0      0   
1004 fdnet 101004     1500  -      -      -        ieee -        0      0   
          
VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
1005 trnet 101005     1500  -      -      -        ibm  -        0      0   

Primary Secondary Type              Ports
------- --------- ----------------- ------------------------------------------

S1#
```

**Вопрос:** почему интерфейс e0/1 указан в VLAN 1?

**Ответ:** потому что мы не назначили ему явно VLAN.

### Шаг 9. Настройка транка на S1

Вручную настроим интерфейс **S1** e0/1 в качестве транка 802.1Q.

#### a. Включение транка для порта

Изменим режим порта коммутатора, чтобы принудительно создать магистральный канал.

В отличие от Cisco Packet Tracer при работе с используемым образом свитча в
EVE-NG потребовалось явно задать тип инкапсуляции
(**switchport trunk encapsulation dot1q**) перед переводом порта в транк.

```text
S1(config)#int e0/1
S1(config-if)#sw tr enc dot1q
S1(config-if)#sw m tr
S1(config-if)#no shut
S1(config-if)#
```

#### b. Установка VLAN для транка

В рамках конфигурации транка установим для native VLAN значение 1000.

```text
S1(config-if)#sw tr native vl 1000
S1(config-if)#
```

#### c. Настройка допустимых VLAN для транка

В качестве другой части конфигурации магистрали укажем, что VLAN 100, 200 и 1000
могут проходить по транку.

```text
S1(config-if)#sw tr all vl 100,200,1000         
S1(config-if)#exit
S1(config)#
```

#### d. Сохранение конфигурации

Сохраним текущую конфигурацию в файл загрузочной конфигурации.

```text
S1#copy run start
Destination filename [startup-config]? 
Building configuration...
Compressed configuration from 1371 bytes to 930 bytes[OK]
S1#
```

<details>
<summary>S1#show run</summary>>

```text
S1#sh run
Building configuration...

Current configuration : 1371 bytes
!
! Last configuration change at 00:55:14 UTC Sat May 3 2025
! NVRAM config last updated at 00:55:19 UTC Sat May 3 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service compress-config
!
hostname S1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$B/PL$0B.kEHdic8i.OfuJwrwVS0
!
no aaa new-model
!
!
!         
!
!
!
!
!
no ip domain-lookup
ip cef
no ipv6 cef
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
! 
!
!
!
!
!
!
!         
!
!
!
!
interface Ethernet0/0
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet0/1
 switchport trunk allowed vlan 100,200,1000
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 1000
 switchport mode trunk
!
interface Ethernet0/2
 switchport access vlan 100
 switchport mode access
!
interface Ethernet0/3
 switchport access vlan 999
 switchport mode access
 shutdown 
!
interface Vlan200
 ip address 192.168.1.66 255.255.255.224
!
ip default-gateway 192.168.1.65
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
!
!
!
!
!
control-plane
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 1511021F0725
 logging synchronous
 login    
line aux 0
line vty 0 4
 password 7 13061E010803
 login
!
!
end

S1#
```

</details>

<details>
<summary>S2#show run</summary>>

```text
S2#sh run
Building configuration...

Current configuration : 1121 bytes
!
! Last configuration change at 00:57:13 UTC Sat May 3 2025
! NVRAM config last updated at 00:57:20 UTC Sat May 3 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
service compress-config
!
hostname S2
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$hN1n$yH.Lsa8.AZQHUvGH2g1hJ/
!
no aaa new-model
!
!
!         
!
!
!
!
!
no ip domain-lookup
ip cef
no ipv6 cef
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
! 
!
!
!
!
!
!
!         
!
!
!
!
interface Ethernet0/0
 switchport mode access
 shutdown
!
interface Ethernet0/1
!
interface Ethernet0/2
 switchport mode access
 shutdown
!
interface Ethernet0/3
!
interface Vlan1
 ip address 192.168.1.98 255.255.255.240
!
ip default-gateway 192.168.1.97
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
!
!
!
!
!
control-plane
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 104D000A0618
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 094F471A1A0A
 login
!
!
end
          
S2#
```

</details>

#### e. Проверка транка

Проверим состояние транка.

```text
S1#show interfaces e0/1 switchport
Name: Et0/1
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: On
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 1000 (Native)
Administrative Native VLAN tagging: enabled
Voice VLAN: none
Administrative private-vlan host-association: none 
Administrative private-vlan mapping: none 
Administrative private-vlan trunk native VLAN: none
Administrative private-vlan trunk Native VLAN tagging: enabled
Administrative private-vlan trunk encapsulation: dot1q
Administrative private-vlan trunk normal VLANs: none
Administrative private-vlan trunk associations: none
Administrative private-vlan trunk mappings: none
Operational private-vlan: none
Trunking VLANs Enabled: 100,200,1000
Pruning VLANs Enabled: 2-1001
Capture Mode Disabled
Capture VLANs Allowed: ALL

Protected: false
Appliance trust: none
S1#
```

**Вопрос:** какой IP-адрес был бы у ПК, если бы он был подключен к сети с помощью
DHCP?

**Ответ:** 192.168.1.67 (если считать, что сервер DHCP выдаёт адреса в той же
подсети, что и VLAN 200).

## Часть 2. Настройка и проверка двух серверов DHCPv4 на R1

В части 2 настроим и проверим сервер DHCPv4 на **R1**. Сервер DHCPv4 будет
обслуживать две подсети, подсеть A и подсеть C.

### Шаг 1. Настройка R1 с пулами DHCPv4

Настроим **R1** с пулами DHCPv4 для двух поддерживаемых подсетей. Ниже приведен
только пул DHCP для подсети A.

#### a. Исключение используемых адресов

Исключим первые пять используемых адресов из каждого пула адресов.

```text
R1(config)#ip dhcp excluded-address 192.168.1.1 192.168.1.5
R1(config)#
```

#### b. Создание пула DHCP

Создадим пул DHCP (используя уникальное имя для каждого пула).

```text
R1(config)#ip dhcp pool R1_Client_LAN
R1(dhcp-config)#
```

#### c. Настройка сети для пула

Укажем сеть, поддерживающую этот DHCP-сервер.

```text
R1(dhcp-config)#network 192.168.1.0 255.255.255.192
R1(dhcp-config)#
```

#### d. Настройка домена

В качестве имени домена укажем CCNA-lab.com.

```text
R1(dhcp-config)#domain-name CCNA-lab.com
R1(dhcp-config)#
```

#### e. Установка шлюза по умолчанию

Настроим соответствующий шлюз по умолчанию для каждого пула DHCP.

```text
R1(dhcp-config)#default-router 192.168.1.1
R1(dhcp-config)#
```

#### f. Настройка времени аренды

Настроим время аренды на 2 дня 12 часов и 30 минут.

```text
R1(dhcp-config)#lease 2 12 30
R1(dhcp-config)#
```

#### g. Настройка второго пула

Создадим и настроим второй пул DHCPv4, используя имя пула R2_Client_LAN и
вычислим сеть, маршрутизатор по умолчанию, используем то же имя домена и время
аренды, что и предыдущий пул DHCP.

```text
R1(config)#ip dhcp excluded-address 192.168.1.96 192.168.1.101
R1(config)#ip dhcp pool R2_Client_LAN
R1(dhcp-config)#network 192.168.1.96 255.255.255.240
R1(dhcp-config)#default-router 192.168.1.97
R1(dhcp-config)#domain-name CCNA-lab.com
R2(dhcp-config)#lease 2 12 30
R1(dhcp-config)#exit
R1(config)#
```

### Шаг 2. Сохранение конфигурации R1

Сохраним текущую конфигурацию в файл загрузочной конфигурации.

```text
R1#copy run start
Destination filename [startup-config]? 
Building configuration...
[OK]
R1#
```

<details>
<summary>R1#show run</summary>

```text
R1#sh run
Building configuration...

Current configuration : 1929 bytes
!
! Last configuration change at 01:17:17 UTC Sat May 3 2025
! NVRAM config last updated at 01:17:20 UTC Sat May 3 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$NAN2$3L0xiFyz9IMRzzyHwxOgA.
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!


!
ip dhcp excluded-address 192.168.1.1 192.168.1.5
ip dhcp excluded-address 192.168.1.96 192.168.1.101
!
ip dhcp pool R1_Client_LAN
 network 192.168.1.0 255.255.255.192
 domain-name CCNA-lab.com
 default-router 192.168.1.1 
 lease 2 12 30
!
ip dhcp pool R2_Client_LAN
 network 192.168.1.96 255.255.255.240
 default-router 192.168.1.97 
 domain-name CCNA-lab.com
 lease 2 12 30
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
!
!
!
!
redundancy
!
!
!         
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 ip address 10.0.0.1 255.255.255.252
!
interface Ethernet0/1
 no ip address
!
interface Ethernet0/1.100
 description Clients
 encapsulation dot1Q 100
 ip address 192.168.1.1 255.255.255.192
!         
interface Ethernet0/1.200
 description Management
 encapsulation dot1Q 200
 ip address 192.168.1.65 255.255.255.224
!
interface Ethernet0/1.999
 description Parking_Lot
 encapsulation dot1Q 999
!
interface Ethernet0/1.1000
 description Native
 encapsulation dot1Q 1000 native
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!         
!
no ip http server
no ip http secure-server
ip route 0.0.0.0 0.0.0.0 10.0.0.2
!
!
!
!
control-plane
!
!
!
!
!
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 094F471A1A0A
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 01100F175804
 login
 transport input none
!
!
end

R1#
```

</details>

### Шаг 3. Проверка конфигурации сервера DHCPv4

Проверим настройки сервера DHCPv4.

#### a. Просмотр сведений о пуле

Чтобы просмотреть сведения о пуле, выполним команду **show ip dhcp pool**.

```text
R1#show ip dhcp pool

Pool R1_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 62
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.1          192.168.1.1      - 192.168.1.62      0

Pool R2_Client_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 14
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.97         192.168.1.97     - 192.168.1.110     0
R1#
```

#### b. Проверка установленных назначений адресов DHCP

Для проверки установленных назначений адресов DHCP выполним команду
**show ip dhcp bindings**.

```text
R1#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/	 	    Lease expiration        Type
		    Hardware address/
		    User name
R1#
```

#### c. Проверка сообщений DHCP

Выполним команду **show ip dhcp server statistics** для проверки сообщений DHCP.

```text
R1#show ip dhcp server statistics
Memory usage         25175
Address pools        2
Database agents      0
Automatic bindings   0
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         0
DHCPREQUEST          0
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            0
DHCPACK              0
DHCPNAK              0
R1#
```

### Шаг 4. Проверка получения адреса на PC-A

Проверим получение IP-адреса компьютером **PC-A**.

#### a. Просмотр конфигурации IP

Из командной строки компьютера **PC-A** выполним команду **ipconfig /all**
(в случае с VPC - **show ip**).

```text
PC-A> show ip

NAME        : PC-A[1]
IP/MASK     : 0.0.0.0/0
GATEWAY     : 0.0.0.0
DNS         : 
MAC         : 00:50:79:66:68:05
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
```

#### b. Просмотр информации после обновления

После завершения процесса обновления (можно принудительно инициировать командой
**ipconfig /renew**) выполним команду **ipconfig** для просмотра новой информации
об IP-адресе.

Для EVE-NG VPS аналогичной командой будет **ip dhcp**.

```text
PC-A> ip dhcp
DDORA IP 192.168.1.6/26 GW 192.168.1.1

PC-A> show ip

NAME        : PC-A[1]
IP/MASK     : 192.168.1.6/26
GATEWAY     : 192.168.1.1
DNS         : 
DHCP SERVER : 192.168.1.1
DHCP LEASE  : 217756, 217800/108900/190575
DOMAIN NAME : CCNA-lab.com
MAC         : 00:50:79:66:68:05
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

PC-A>
```

#### c. Проверка подключения

Проверим подключение с помощью пинга IP-адреса интерфейса **R1** e0/1.

```text
PC-A> ping 192.168.1.1

84 bytes from 192.168.1.1 icmp_seq=1 ttl=255 time=0.484 ms
84 bytes from 192.168.1.1 icmp_seq=2 ttl=255 time=0.562 ms
84 bytes from 192.168.1.1 icmp_seq=3 ttl=255 time=0.862 ms
84 bytes from 192.168.1.1 icmp_seq=4 ttl=255 time=0.692 ms
84 bytes from 192.168.1.1 icmp_seq=5 ttl=255 time=0.565 ms

PC-A>
```

## Часть 3. Настройка и проверка DHCP-ретрансляции на R2

В части 3 настроим **R2** для ретрансляции DHCP-запросов из локальной сети на
интерфейсе e0/1 на DHCP-сервер (**R1**).

### Шаг 1. Настройка R2 в качестве агента DHCP-ретрансляции

Настроим **R2** в качестве агента DHCP-ретрансляции для локальной сети на e0/1.

#### a. Настройка ретрансляции DHCP

Выполним команду **ip helper-address** на e0/1, указав IP-адрес e0/0 **R1**.

```text
R2(config)#int e0/1
R2(config-if)#ip helper-address 10.0.0.1
R2(config-if)#exit
R2(config)#
```

#### b. Сохранение конфигурации

Сохраним конфигурацию.

```text
R2#copy run start
Destination filename [startup-config]? 
Building configuration...
[OK]
R2#
```

<details>
<summary>R2#show run</summary>

```text
R2#sh run
Building configuration...

Current configuration : 1216 bytes
!
! Last configuration change at 01:31:32 UTC Sat May 3 2025
! NVRAM config last updated at 01:31:35 UTC Sat May 3 2025
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
service password-encryption
!
hostname R2
!
boot-start-marker
boot-end-marker
!
!
enable secret 5 $1$QFFr$fapdzxpn96wsv4bWeTFnM/
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!


!
!
!
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!         
!
!
!
!
!
redundancy
!
!
! 
!
!
!
!
!
!
!
!
!
!
!
!
interface Ethernet0/0
 ip address 10.0.0.2 255.255.255.252
!
interface Ethernet0/1
 ip address 192.168.1.97 255.255.255.240
 ip helper-address 10.0.0.1
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 0.0.0.0 0.0.0.0 10.0.0.1
!
!
!
!         
control-plane
!
!
!
!
!
!
!
banner motd ^C ATTENTION! Unauthorized access is strictly prohibited. ^C
!
line con 0
 password 7 070C285F4D06
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 00071A150754
 login
 transport input none
!
!
end
          
R2#
```

</details>

### Шаг 2. Получение IP-адреса от DHCP на PC-B

Проверим получение IP-адреса компьютером **PC-B**.

#### a. Просмотр информации ipconfig

Из командной строки компьютера **PC-B** выполним команду **ipconfig /all**
(в случае с VPC - **show ip**).

```text
PC-B> show ip

NAME        : PC-B[1]
IP/MASK     : 0.0.0.0/0
GATEWAY     : 0.0.0.0
DNS         : 
MAC         : 00:50:79:66:68:06
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

PC-B> 
```

#### b. Просмотр информации ipconfig после обновления

После завершения процесса обновления выполним команду **ipconfig** для просмотра
новой информации об IP-адресе.

Для EVE-NG VPS аналогичной командой будет **ip dhcp**.

```text
PC-B> ip dhcp
DDORA IP 192.168.1.102/28 GW 192.168.1.97

PC-B> show ip

NAME        : PC-B[1]
IP/MASK     : 192.168.1.102/28
GATEWAY     : 192.168.1.97
DNS         : 
DHCP SERVER : 10.0.0.1
DHCP LEASE  : 217795, 217800/108900/190575
DOMAIN NAME : CCNA-lab.com
MAC         : 00:50:79:66:68:06
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

PC-B> 
```


#### c. Проверка подключения к R1

Проверьте подключение с помощью пинга IP-адреса интерфейса **R1** e0/1.

```text
PC-B> ping 192.168.1.1

84 bytes from 192.168.1.1 icmp_seq=1 ttl=254 time=1.160 ms
84 bytes from 192.168.1.1 icmp_seq=2 ttl=254 time=0.844 ms
84 bytes from 192.168.1.1 icmp_seq=3 ttl=254 time=0.892 ms
84 bytes from 192.168.1.1 icmp_seq=4 ttl=254 time=0.766 ms
84 bytes from 192.168.1.1 icmp_seq=5 ttl=254 time=0.833 ms

PC-B>
```

#### d. Просмотр назначений DHCP

Выполним **show ip dhcp binding** для **R1** для проверки назначений адресов в DHCP.

```text
R1#show ip dhcp binding 
Bindings from all pools not associated with VRF:
IP address          Client-ID/	 	    Lease expiration        Type
		    Hardware address/
		    User name
192.168.1.6         0100.5079.6668.05       May 05 2025 01:57 PM    Automatic
192.168.1.102       0100.5079.6668.06       May 05 2025 02:03 PM    Automatic
R1#
```

#### e. Просмотр сообщения DHCP

Выполним команду **show ip dhcp server statistics** для проверки сообщений DHCP.

Маршрутизатор **R1**:

```text
R1#show ip dhcp server statistics
Memory usage         42093
Address pools        2
Database agents      0
Automatic bindings   2
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         4
DHCPREQUEST          2
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            2
DHCPACK              2
DHCPNAK              0
R1#
```

Маршрутизатор **R2**:

```text
R2#show ip dhcp server statistics
Memory usage         22565
Address pools        0
Database agents      0
Automatic bindings   0
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         0
DHCPREQUEST          0
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            0
DHCPACK              0
DHCPNAK              0
R2#
```
