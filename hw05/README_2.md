# Домашнее задание №3.2 «Настройка DHCPv6»

## Топология

![Топология](topology.drawio.png)

## Таблица адресации

| Устройство | Интерфейс | IPv6-адрес            |
| ---------- | --------- | --------------------- |
| R1         | e0/0      | 2001:db8:acad:2::1/64 |
|            |           | fe80::1               |
|            | e0/1      | 2001:db8:acad:1::1/64 |
|            |           | fe80::1               |
| R2         | e0/0      | 2001:db8:acad:2::2/64 |
|            |           | fe80::2               |
|            | e0/1      | 2001:db8:acad:3::1/64 |
|            |           | fe80::1               |
| PC-A       | NIC       | DHCP                  |
| PC-A       | NIC       | DHCP                  |

## Задачи

* [Часть 1. Создание сети и настройка основных параметров устройства.](#часть-1-создание-сети-инастройка-основных-параметров-устройства)
* [Часть 2. Проверка назначения адреса SLAAC от R1.](#часть-2-проверка-назначения-адреса-slaac-от-r1)
* [Часть 3. Настройка и проверка сервера DHCPv6 на R1.](#часть-3-настройка-и-проверка-сервера-dhcpv6-на-r1)
* [Часть 4. Настройка сервера DHCPv6 с сохранением состояния на R1.](#часть-4-настройка-сервера-dhcpv6-с-сохранением-состояния-на-r1)
* [Часть 5. Настройка и проверка ретрансляции DHCPv6 на R2.](#часть-5-настройка-и-проверка-ретрансляции-dhcpv6-на-r2)

## Общие сведения/сценарий

Динамическое назначение глобальных индивидуальных IPv6-адресов можно настроить
тремя способами:

* автоматическая конфигурация адреса без сохранения состояния (Stateless Address
  Autoconfiguration, SLAAC);
* DHCPv6 без отслеживания состояния;
* адресация DHCPv6 с учетом состояний.

При использовании SLAAC для назначения адресов IPv6 хостам сервер DHCPv6 не
используется. Поскольку DHCPv6 сервер не используется при реализации SLAAC, хосты
не могут получать дополнительную важную сетевую информацию, включая адрес сервера
доменных имен (DNS), а также имя домена.

При использовании Stateless DHCPv6 для назначения адресов IPv6 хосту сервер DHCPv6
используется для назначения дополнительной важной информации о сети, однако адрес
IPv6 назначается с помощью SLAAC.

При использовании DHCPv6 с отслеживанием состояния, сервер DHCP назначает всю
информацию, включая IPv6-адрес узла.

Определение способа получения динамической IPv6-адресации зависит от установленных
значений флагов, содержащихся в объявлениях маршрутизатора (сообщениях RA).

В предложенном сценарии размеры компании увеличились, и сетевые администраторы
больше не имеют возможности назначать IP-адреса для устройств вручную. Наша
задача - настроить маршрутизатор **R2** для назначения адресов IPv6 в двух разных
подсетях, подключенных к маршрутизатору **R1**.

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

### Шаг 1. Создание сети

Подключим устройства, как показано в топологии, и подсоединим необходимые кабели.

![Топология в CPT](topo_eveng_1.png)

### Шаг 2. Настройка базовых параметров коммутаторов

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

#### h. Выключение неиспользуемых портов

Отключим все неиспользуемые порты.

<details>
<summary>Коммутатор <strong>S1</strong></summary>

```text
S1(config)#int e0/0
S1(config-if)#shut
S1(config-if)#
*May  2 16:03:23.690: %LINK-5-CHANGED: Interface Ethernet0/0, changed state to administratively down
*May  2 16:03:24.694: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to down
S1(config-if)#exit
S1(config)#int e0/3
S1(config-if)#shut
S1(config-if)#
*May  2 16:03:37.569: %LINK-5-CHANGED: Interface Ethernet0/3, changed state to administratively down
*May  2 16:03:38.573: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/3, changed state to down
S1(config-if)#exit
S1(config)#
```

</details>

<details>
<summary>Коммутатор <strong>S2</strong></summary>

```text
S2(config)#int e0/0
S2(config-if)#shut
S2(config-if)#
*May  2 16:04:07.888: %LINK-5-CHANGED: Interface Ethernet0/0, changed state to administratively down
*May  2 16:04:08.892: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to down
S2(config-if)#exit
S2(config)#int e0/2
S2(config-if)#shut
S2(config-if)#
*May  2 16:04:18.971: %LINK-5-CHANGED: Interface Ethernet0/2, changed state to administratively down
*May  2 16:04:19.975: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/2, changed state to down
S2(config-if)#exit
S2(config)#
```

</details>


#### i. Сохранение конфигурации

Скопируем текущую конфигурацию в файл загрузочной конфигурации.

```text
S1#copy run start
Destination filename [startup-config]? 
Building configuration...
Compressed configuration from 923 bytes to 686 bytes[OK]
S1#
```

<details>
<summary>S1# show run</summary>

```text
S1#sh run
Building configuration...

Current configuration : 923 bytes
!
! Last configuration change at 16:06:13 UTC Fri May 2 2025
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
enable secret 5 $1$xyOY$zjUKH2U7uQ8Bz7eii04qm.
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
 shutdown
!
interface Ethernet0/1
!
interface Ethernet0/2
!
interface Ethernet0/3
 shutdown
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
 password 7 01100F175804
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 05080F1C2243
 login
!
!
end

S1#
```

</details>

<details>
<summary>S2# show run</summary>

```text
S2#sh run
Building configuration...

Current configuration : 923 bytes
!
! Last configuration change at 16:06:18 UTC Fri May 2 2025
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
enable secret 5 $1$/RRF$0FCj6cfwyyxu0tbGtuxdv.
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
 shutdown
!
interface Ethernet0/1
!
interface Ethernet0/2
 shutdown
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
 password 7 01100F175804
 login
!
!
end

S2#
```

</details>

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

#### h. Активация IPv6-маршрутизации

Включим IPv6-маршрутизацию.

```text
R1(config)#ipv6 unicast-routing
R1(config)#
```

#### i. Сохранение конфигурации

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
<summary>R1# show run</summary>

```text
R1#sh run
Building configuration...

Current configuration : 1085 bytes
!
! Last configuration change at 16:21:25 UTC Fri May 2 2025
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
enable secret 5 $1$0x35$OhilpQd8O9xQ7iESVlGQf/
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
ipv6 unicast-routing
ipv6 cef
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
 password 7 110A1016141D
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 094F471A1A0A
 login
 transport input none
!
!
end
          
R1#
```

</details>

<details>
<summary>R2# show run</summary>

```text
R2#sh run
Building configuration...

Current configuration : 1085 bytes
!
! Last configuration change at 16:21:33 UTC Fri May 2 2025
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
enable secret 5 $1$3EBu$VqxuRSxFc5.eAnOHtlXCx1
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
ipv6 unicast-routing
ipv6 cef
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
 password 7 0822455D0A16
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
          
R2#
```

</details>

### Шаг 4. Настройка интерфейсов и маршрутизации

Настроим интерфейсы и маршрутизацию для обоих маршрутизаторов.

#### a. Установка IPv6-адресов на интерфейсах

Настроим интерфейсы e0/0 и e0/1 на **R1** и **R2** с адресами IPv6, указанными
в таблице выше.

```text
R1(config)#int e0/0
R1(config-if)#ipv6 addr 2001:db8:acad:2::1/64
R1(config-if)#ipv6 addr fe80::1 link-local
R1(config-if)#no shut
R1(config-if)#
*May  2 16:23:40.331: %LINK-3-UPDOWN: Interface Ethernet0/0, changed state to up
R1(config-if)#
*May  2 16:23:41.336: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to up
R1(config-if)#exit
R1(config)#int e0/1
R1(config-if)#ipv6 addr 2001:db8:acad:1::1/64
R1(config-if)#ipv6 addr fe80::1 link-local
R1(config-if)#no shut
R1(config-if)#
*May  2 16:24:07.877: %LINK-3-UPDOWN: Interface Ethernet0/1, changed state to up
*May  2 16:24:08.881: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
R1(config-if)#exit
R1(config)#
```

```text
R2(config)#int e0/0
R2(config-if)#ipv6 addr 2001:db8:acad:2::2/64
R2(config-if)#ipv6 addr fe80::2 link-local
R2(config-if)#no shut
R2(config-if)#
*May  2 16:24:54.410: %LINK-3-UPDOWN: Interface Ethernet0/0, changed state to up
*May  2 16:24:55.410: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to up
R2(config-if)#exit
R2(config)#int e0/1
R2(config-if)#ipv6 addr 2001:db8:acad:3::1/64
R2(config-if)#ipv6 addr fe80::1 link-local
R2(config-if)#no shut
R2(config-if)#
*May  2 16:25:29.431: %LINK-3-UPDOWN: Interface Ethernet0/1, changed state to up
*May  2 16:25:30.431: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
R2(config-if)#exit
R2(config)#
```

#### b. Настройка маршрута по умолчанию

Настроим маршрут по умолчанию на каждом маршрутизаторе, который указывает на
IP-адрес e0/0 на другом маршрутизаторе.

```text
R1(config)#ipv6 route ::/0 2001:db8:acad:2::2
R1(config)#
```

```text
R2(config)#ipv6 route ::/0 2001:db8:acad:2::1
R2(config)#
```

#### c. Проверка работы маршрутизации

Убедимся, что маршрутизация работает с помощью пинга адреса e0/1 **R2** из **R1**.

```text
R1#ping 2001:db8:acad:3::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:3::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
R1#
```

#### d. Сохранение конфигурации

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

Current configuration : 1238 bytes
!
! Last configuration change at 16:26:32 UTC Fri May 2 2025
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
enable secret 5 $1$0x35$OhilpQd8O9xQ7iESVlGQf/
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
ipv6 unicast-routing
ipv6 cef
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
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:2::1/64
!
interface Ethernet0/1
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:1::1/64
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
ipv6 route ::/0 2001:DB8:ACAD:2::2
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
 password 7 110A1016141D
 logging synchronous
 login
line aux 0
line vty 0 4
 password 7 094F471A1A0A
 login
 transport input none
!         
!
end

R1#
```

</details>

<details>
<summary>R2#show run</summary>

```text
R2#sh run
Building configuration...

Current configuration : 1238 bytes
!
! Last configuration change at 16:27:02 UTC Fri May 2 2025
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
enable secret 5 $1$3EBu$VqxuRSxFc5.eAnOHtlXCx1
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
ipv6 unicast-routing
ipv6 cef
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
 ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:ACAD:2::2/64
!
interface Ethernet0/1
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:3::1/64
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
ipv6 route ::/0 2001:DB8:ACAD:2::1
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
 password 7 0822455D0A16
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

R2#
```

</details>

## Часть 2. Проверка назначения адреса SLAAC от R1

В части 2 мы убедимся, что узел **PC-A** получает адрес IPv6 с помощью метода
SLAAC.

Включим **PC-A** и убедимся, что сетевой адаптер настроен для автоматической
настройки IPv6.

```text
PC-A> show ipv6

NAME              : PC-A[1]
LINK-LOCAL SCOPE  : fe80::250:79ff:fe66:6805/64
GLOBAL SCOPE      : 2001:db8:acad:1:2050:79ff:fe66:6805/64
DNS               : 
ROUTER LINK-LAYER : aa:bb:cc:00:10:10
MAC               : 00:50:79:66:68:05
LPORT             : 20000
RHOST:PORT        : 127.0.0.1:30000
MTU:              : 1500

PC-A>
```

**Вопрос:** откуда взялась часть адреса с идентификатором хоста?

**Ответ:** была получена от **R1** в рамках SLAAC как ответ на RS запрос.

## Часть 3. Настройка и проверка сервера DHCPv6 на R1

В части 3 выполним настройку и проверку состояния DHCP-сервера на **R1**. Цель
состоит в том, чтобы предоставить **PC-A** информацию о DNS-сервере и домене.

### Шаг 1. Просмотр конфигурации PC-A

Более подробно изучим конфигурацию **PC-A**.

#### a. Просмотр информации о настройках IP

Выполним команду **ipconfig /all** на **PC-A** и посмотрим на результат (для
EVE-NG VPC **show ipv6**).

```text
PC-A> show ipv6

NAME              : PC-A[1]
LINK-LOCAL SCOPE  : fe80::250:79ff:fe66:6805/64
GLOBAL SCOPE      : 2001:db8:acad:1:2050:79ff:fe66:6805/64
DNS               : 
ROUTER LINK-LAYER : aa:bb:cc:00:10:10
MAC               : 00:50:79:66:68:05
LPORT             : 20000
RHOST:PORT        : 127.0.0.1:30000
MTU:              : 1500

PC-A> 
```

#### b. Анализ информации об адресах

Обратим внимание, что основной DNS-суффикс отсутствует. Также обратим внимание,
что предоставленные адреса DNS-сервера являются адресами «локального сайта
anycast», а не одноадресными адресами, как ожидалось.

### Шаг 2. Настройка R1 как stateless DHCP

Настроим **R1** для предоставления DHCPv6 без состояния для **PC-A**.

#### a. Настройка пула DHCP на R1

Создадим пул DHCP IPv6 на **R1** с именем R1-STATELESS. В составе этого пула
назначим адрес DNS-сервера как 2001:db8:acad::1, а имя домена — как stateless.com.

```text
R1(config)#ipv6 dhcp pool R1-STATELESS
R1(config-dhcpv6)#dns-server 2001:db8:acad::1
R1(config-dhcpv6)#domain-name STATELESS.com
R1(config-dhcpv6)#exit
R1(config)#
```

#### b. Настройка флагов DHCP на интерфейсе R1 e0/1

Настроим интерфейс e0/1 на **R1**, чтобы предоставить флаг конфигурации OTHER
для локальной сети **R1** и укажем только что созданный пул DHCP в качестве ресурса
DHCP для этого интерфейса.

```text
R1(config)#int e0/1
R1(config-if)#ipv6 nd other-config-flag
R1(config-if)#ipv6 dhcp server R1-STATELESS
R1(config-if)#exit
R1(config)#
```

#### c. Сохранение конфигурации

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

Current configuration : 1382 bytes
!
! Last configuration change at 16:37:33 UTC Fri May 2 2025
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
enable secret 5 $1$0x35$OhilpQd8O9xQ7iESVlGQf/
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
ipv6 unicast-routing
ipv6 cef
ipv6 dhcp pool R1-STATELESS
 dns-server 2001:DB8:ACAD::1
 domain-name STATELESS.com
!
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
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:2::1/64
!
interface Ethernet0/1
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:1::1/64
 ipv6 nd other-config-flag
 ipv6 dhcp server R1-STATELESS
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
ipv6 route ::/0 2001:DB8:ACAD:2::2
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
 password 7 110A1016141D
 logging synchronous
 login    
line aux 0
line vty 0 4
 password 7 094F471A1A0A
 login
 transport input none
!
!
end

R1#
```

</details>

#### d. Перезапуск PC-A

Перезапустим **PC-A**.

Так как EVE-NG VPC не поддерживает получение DNS через SLAAC, то заменим VPC
на виртуальную машину с Windows 10.

#### e. Просмотр информации об IP-адресе

Проверим вывод **ipconfig /all** и обратим внимание на изменения.

```text
C:\Users\user>ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : DESKTOP-L382O2N
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : STATELESS.com

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : STATELESS.com
   Description . . . . . . . . . . . : Intel(R) PRO/1000 MT Network Connection
   Physical Address. . . . . . . . . : 50-00-00-07-00-00
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:1:b800:3e54:b0e0:f895(Preferred)
   Temporary IPv6 Address. . . . . . : 2001:db8:acad:1:a0f9:1352:6ddb:db76(Preferred)
   Link-local IPv6 Address . . . . . : fe80::b800:3e54:b0e0:f895%7(Preferred)
   Autoconfiguration IPv4 Address. . : 169.254.248.149(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::1%7
   DHCPv6 IAID . . . . . . . . . . . : 55574528
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2F-A7-3F-41-50-00-00-07-00-00
   DNS Servers . . . . . . . . . . . : 2001:db8:acad::1
   NetBIOS over Tcpip. . . . . . . . : Enabled
   Connection-specific DNS Suffix Search List :
                                       STATELESS.com
```

#### f. Проверка подключения к R2

Протестируем подключение с помощью пинга IP-адреса интерфейса e0/1 **R2**.

```text
C:\Users\user>ping 2001:db8:acad:3::1

Pinging 2001:db8:acad:3::1 with 32 bytes of data:
Reply from 2001:db8:acad:3::1: time=1ms
Reply from 2001:db8:acad:3::1: time=1ms
Reply from 2001:db8:acad:3::1: time=1ms
Reply from 2001:db8:acad:3::1: time=1ms

Ping statistics for 2001:db8:acad:3::1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 1ms, Average = 1ms

C:\Users\user>
```

## Часть 4. Настройка сервера DHCPv6 с сохранением состояния на R2

В части 4 настроим сервер DHCP для ответа на запросы от локальной сети на **R2**.

#### a. Настройка пула DHCP

Создадим пул DHCPv6 на **R1** для сети 2001:db8:acad:3:aaa::/80. Это предоставит
адреса локальной сети, подключенной к интерфейсу e0/1 на **R2**. В составе пула
зададим DNS-сервер 2001:db8:acad::254 и доменное имя STATEFUL.com.

```text
R1(config)#ipv6 dhcp pool R2-STATEFUL
R1(config-dhcpv6)#address prefix 2001:db8:acad:3:aaa::/80
R1(config-dhcpv6)#dns-server 2001:db8:acad::254
R1(config-dhcpv6)#domain-name STATEFUL.com  
R1(config-dhcpv6)#exit
R1(config)#
```

#### b. Назначение пула интерфейсу

Назначим только что созданный пул DHCPv6 интерфейсу e0/0 на **R1**.

```text
R1(config)#int e0/0
R1(config-if)#ipv6 dhcp server R2-STATEFUL
R1(config-if)#exit
R1(config)#
```

## Часть 5. Настройка и проверка ретрансляции DHCPv6 на R2

В части 5 настроим и проверим ретрансляцию DHCPv6 на **R2**, позволив **PC-B**
получать адрес IPv6.

### Шаг 1. Проверка получения адреса PC-B

Включим **PC-B** и проверим адрес SLAAC, который он генерирует.

Также, как и с PC-A, вместо VPC будем использовать виртуальную машину с Windows 10.

```text
C:\Users\user>ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : DESKTOP-L382O2N
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Description . . . . . . . . . . . : Intel(R) PRO/1000 MT Network Connection
   Physical Address. . . . . . . . . : 50-00-00-08-00-00
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:3:e0a4:9175:28d6:bb2e(Preferred)
   Temporary IPv6 Address. . . . . . : 2001:db8:acad:3:45a4:3f7e:f0ae:cd71(Preferred)
   Link-local IPv6 Address . . . . . : fe80::e0a4:9175:28d6:bb2e%3(Preferred)
   Autoconfiguration IPv4 Address. . : 169.254.187.46(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::1%3
   DHCPv6 IAID . . . . . . . . . . . : 55574528
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2F-A7-47-EE-50-00-00-08-00-00
   DNS Servers . . . . . . . . . . . : fec0:0:0:ffff::1%1
                                       fec0:0:0:ffff::2%1
                                       fec0:0:0:ffff::3%1
   NetBIOS over Tcpip. . . . . . . . : Enabled
```

Обратим внимание на вывод, что используется префикс 2001:db8:acad:3::

### Шаг 2. Настройка R2 в качестве агента DHCP-ретрансляции

Выполним настройку **R2** в качестве агента DHCP-ретрансляции для локальной сети
на e0/1.

#### a. Настройка ретрансляции DHCP

Для включения ретрансляции DHCP-запросов воспользуемся командой **ipv6 dhcp relay**
(в CPT не поддерживается, в EVE-NG работает), указав в качестве адреса назначения
интерфейс e0/0 на **R1**. Также выполним команду **managed-config-flag**.

```text
R2(config)#int g0/0/1
R2(config-if)#ipv6 nd managed-config-flag
R2(config-if)#exit
R2(config)#
```

#### b. Сохранение конфигурации

Сохраним текущую конфигурацию в файл загрузочной конфигурации.

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

Current configuration : 1327 bytes
!
! Last configuration change at 04:47:22 UTC Sat May 3 2025
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
enable secret 5 $1$3EBu$VqxuRSxFc5.eAnOHtlXCx1
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
ipv6 unicast-routing
ipv6 cef
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
 ipv6 address FE80::2 link-local
 ipv6 address 2001:DB8:ACAD:2::2/64
!
interface Ethernet0/1
 no ip address
 ipv6 address FE80::1 link-local
 ipv6 address 2001:DB8:ACAD:3::1/64
 ipv6 nd managed-config-flag
 ipv6 dhcp relay destination 2001:DB8:ACAD:2::1 Ethernet0/0
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
ipv6 route ::/0 2001:DB8:ACAD:2::1
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
 password 7 0822455D0A16
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

R2#
```

</details>

### Шаг 3. Проверка получения IPv6-адреса PC-B

Проверим, получает ли **PC-B** адрес IPv6 из DHCPv6.

#### a. Перезапуск PC-B

Перезапустим **PC-B**.

#### b. Просмотр информации об IP-адресах

Откроем командную строку на **PC-B**, выполним команду **ipconfig /all** и
проверим выходные данные, чтобы увидеть результаты операции ретрансляции DHCPv6.

```text
C:\Users\user>ipconfig /all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : DESKTOP-L382O2N
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : STATEFUL.com

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : STATEFUL.com
   Description . . . . . . . . . . . : Intel(R) PRO/1000 MT Network Connection
   Physical Address. . . . . . . . . : 50-00-00-08-00-00
   DHCP Enabled. . . . . . . . . . . : Yes
   Autoconfiguration Enabled . . . . : Yes
   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:3:aaa:b14c:4e74:808(Preferred)
   Lease Obtained. . . . . . . . . . : Saturday, May 3, 2025 4:51:56 AM
   Lease Expires . . . . . . . . . . : Monday, May 5, 2025 4:46:37 AM
   IPv6 Address. . . . . . . . . . . : 2001:db8:acad:3:e0a4:9175:28d6:bb2e(Preferred)
   Temporary IPv6 Address. . . . . . : 2001:db8:acad:3:293c:6b6e:afa6:e13(Preferred)
   Link-local IPv6 Address . . . . . : fe80::e0a4:9175:28d6:bb2e%3(Preferred)
   Autoconfiguration IPv4 Address. . : 169.254.187.46(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::1%3
   DHCPv6 IAID . . . . . . . . . . . : 55574528
   DHCPv6 Client DUID. . . . . . . . : 00-01-00-01-2F-A7-47-EE-50-00-00-08-00-00
   DNS Servers . . . . . . . . . . . : 2001:db8:acad::254
   NetBIOS over Tcpip. . . . . . . . : Enabled
   Connection-specific DNS Suffix Search List :
                                       STATEFUL.com
```

Обратим внимание, что **PC-B** получил адрес DNS-сервера по DHCP.

#### c. Проверка подключения

Проверим подключение с помощью пинга IP-адреса интерфейса **R1** e0/1.

```text
C:\Users\user>ping 2001:DB8:ACAD:3::1

Pinging 2001:db8:acad:3::1 with 32 bytes of data:
Reply from 2001:db8:acad:3::1: time=10ms
Reply from 2001:db8:acad:3::1: time=1ms
Reply from 2001:db8:acad:3::1: time=1ms
Reply from 2001:db8:acad:3::1: time<1ms

Ping statistics for 2001:db8:acad:3::1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 10ms, Average = 3ms

C:\Users\user>
```
