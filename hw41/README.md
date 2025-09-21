# Домашнее задание №14 «IPSEC over DMVPN»

## Цель работы

В данной самостоятельной работе необходимо настроить GRE поверх IPSec между офисами
«Москва» и «Санкт-Петербург», настроить DMVPN поверх IPSec между офисами «Москва»
и «Чокурдах», «Лабытнанги».

## Задачи

1. [Настройка GRE поверх IPSEC.](#настройка-gre-поверх-ipsec)
2. [Настройка DMVPN поверх IPSEC.](#настройка-dmvpn-поверх-ipsec)
3. [Проверка IP связности.](#проверка-ip-связности)

## Технические требования

Для IPSec использовать CA и сертификаты.

## Топология

Топология лабораторного стенда собрана в среде EVE-NG.

![Топология с IP адресами](topo_drawio.png)

## Настройка CA

Для выдачи сертификатов настроим Certificate Authority Server на маршрутизаторе **R12**.

### Начальная настройка маршрутизатора

Для корректной работы центра сертификации требуется, чтобы время на сервере и
клиентах было одинаковым. Ранее мы настраивали NTP в офисе «Москва». Но источником
времени для мастер сервера был аппаратный календарь хост-машины с EVE-NG. На
остальных маршрутизаторах время также устанавливается согласно этому источнику.

Зададим имя сервера и домен:

```text
R12(config)#hostname R12CA
R12CA(config)#ip domain name otus.ru
```

Запустим встроенный HTTP-сервер (необходим для работы протокола выдачи
сертификатов SCEP - Simple Certificate Enrollment Protocol).

```text
R12CA(config)#ip http server
```

### Создание RSA ключей

Сгенерируем RSA ключи, которые будут использоваться сервером для подписи:

```text
R12CA(config)#crypto key generate rsa general-keys label R12CA exportable 
The name for the keys will be: R12CA
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 2048
% Generating 2048 bit RSA keys, keys will be exportable...
[OK] (elapsed time was 1 seconds)

R12CA(config)#
Sep 21 00:54:49.245: %SSH-5-ENABLED: SSH 1.99 has been enabled
R12CA(config)#
```

Сгенерированные ключи можно просмотреть:

<details>
<summary>show crypto key mypubkey rsa</summary>

```text
R12CA#show crypto key mypubkey rsa
% Key pair was generated at: 00:54:49 UTC Sep 21 2025
Key name: R12CA
Key type: RSA KEYS
 Storage Device: not specified
 Usage: General Purpose Key
 Key is exportable.
 Key Data:
  30820122 300D0609 2A864886 F70D0101 01050003 82010F00 3082010A 02820101 
  00C03F3B E84C2179 810BD811 AFA00C77 17D0D971 B0ABD248 1CF762E2 603EAC85 
  8A517E91 F24C5050 E6276799 24D3CFDC FD0F9913 B85FADD4 C00D5A57 3562883F 
  28D3DEE0 64872338 4172EB3E 2A09DEA7 2A5827DD 21BBE652 D82BCEBD 0CF164AB 
  69A43B43 9BF4EDF8 C90A5CFF A261EDF0 3B5F4BBD 32DADF2D A730A3A8 05450CB5 
  B15760E6 CAD77552 0B19C3E5 53E02BB2 9AA47AE2 FCF1B392 097A9723 983448A9 
  E1F6203D 0F20BB1A 299E9E86 292E6A88 C5D796E7 99EFEDDC CFB4F2DA 0973236F 
  B6E6E716 68FD4B4F A133736C 007DF51B 8AE549DE 941A139C CF0BA28D 8BB2A8CA 
  3F708B5D 7F94430D 5A21480E 8A7E8194 AC68864C 06FBF41C 28DA2B39 2D10B102 
  57020301 0001
% Key pair was generated at: 00:54:49 UTC Sep 21 2025
Key name: R12CA.server
Key type: RSA KEYS
Temporary key
 Usage: Encryption Key
 Key is not exportable.
 Key Data:
  307C300D 06092A86 4886F70D 01010105 00036B00 30680261 00A19567 9B73F9EB 
  390D7FDC 586087E2 D4E4AFFE 2E858306 E7B3EDD6 548698FF CECEC147 38BE9405 
  0BD1C942 02933C5E 9961831A 69C7B13B F96CD5AC A8690CFB 77D65713 09D758F2 
  9945225C F7FE3C9F CF45CF12 49870F38 F9F6D9D6 EF230553 59020301 0001
R12CA#
```

Экспортируем ключи в NVRAM память:

```text
R12CA(config)#crypto key export rsa R12CA pem url nvram: 3des cisco123
% Key name: R12CA
   Usage: General Purpose Key
Exporting public key...
Destination filename [R12CA.pub]? 
Writing file to nvram:R12CA.pub
Exporting private key...
Destination filename [R12CA.prv]? 
Writing file to nvram:R12CA.prv
R12CA(config)#
```

</details>

### Запуск сервера

Включим сервер (пароль cisco123):

```text
R12CA(config)#crypto pki server R12CA
R12CA(cs-server)#no shut
%Some server settings cannot be changed after CA certificate generation.
% Please enter a passphrase to protect the private key
% or type Return to exit
Password: 

Re-enter password: 

% Certificate Server enabled.
R12CA(cs-server)#
Sep 21 01:19:16.260: %PKI-6-CS_ENABLED: Certificate server now enabled.
R12CA(cs-server)#end
R12CA#
```

Выведем сведения о trustpoint:

```text
R12CA#show crypto pki trustpoints
Trustpoint R12CA:
    Subject Name: 
    cn=R12CA
          Serial Number (hex): 01
    Certificate configured.


R12CA#
```

## Настройка GRE поверх IPSEC

GRE между офисами «Москва» и «Санкт-Петербург» был настроен в предыдущей домашней
работе. Нам осталось только включить IPSEC на туннельном интерфейсе.

### Получение сертификата

Настроим маршрутизаторы **R14**, **R15**, **R18** для получения сертификатов от CA.

Создадим host-запись с адресом CA:

```text
R15(config)#ip domain name otus.ru
R15(config)#ip host R12CA 10.0.0.12
```

Сгенерируем RSA ключи:

```text
R15(config)#crypto key generate rsa
The name for the keys will be: R15.otus.ru
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 2048
% Generating 2048 bit RSA keys, keys will be non-exportable...
[OK] (elapsed time was 1 seconds)

R15(config)#
Sep 21 02:37:26.150: %SSH-5-ENABLED: SSH 1.99 has been enabled
R15(config)#
```

Создадим trustpoint и укажем url для запроса сертификатов:

```text
R15(config)#crypto pki trustpoint R12CA
R15(ca-trustpoint)#enrollment url http://R12CA:80
R15(ca-trustpoint)#exit
R15(config)#
```

Получим сертификат CA-сервера:

```text
R15(config)#crypto pki authenticate R12CA
Certificate has the following attributes:
       Fingerprint MD5: FA3654F3 94F533B3 649F11D1 C190E669 
      Fingerprint SHA1: C7478143 D52497B4 732A26C5 E9F0A995 B6E35101 

% Do you accept this certificate? [yes/no]: yes
Trustpoint CA certificate accepted.
R15(config)#
```

Запросим собственный сертификат (пароль cisco123):

```text
R15(config)#crypto pki enroll R12CA
%
% Start certificate enrollment .. 
% Create a challenge password. You will need to verbally provide this
   password to the CA Administrator in order to revoke your certificate.
   For security reasons your password will not be saved in the configuration.
   Please make a note of it.

Password: 
Re-enter password: 

% The subject name in the certificate will include: R15.otus.ru
% Include the router serial number in the subject name? [yes/no]: yes
% The serial number in the certificate will be: 67109104
% Include an IP address in the subject name? [no]: 
Request certificate from CA? [yes/no]: yes
% Certificate request sent to Certificate Authority
% The 'show crypto pki certificate verbose R12CA' commandwill show the fingerprint.

R15(config)#
Sep 21 03:05:02.312: CRYPTO_PKI:  Certificate Request Fingerprint MD5: 2EDBC8C4 D2571528 7AAEDDE6 16585A93 
Sep 21 03:05:02.312: CRYPTO_PKI:  Certificate Request Fingerprint SHA1: 087B82E3 4ABA8B92 54FD8539 B59C46AF 8AA480D0 
R15(config)#
```

Можно увидеть, что запрос ушёл:

```text
R15#show crypto pki certificate R12CA
CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer: 
    cn=R12CA
  Subject: 
    cn=R12CA
  Validity Date: 
    start date: 01:19:16 UTC Sep 21 2025
    end   date: 01:19:16 UTC Sep 20 2028
  Associated Trustpoints: R12CA 


Certificate
  Subject:
    Name: R15.otus.ru
    Serial Number: 67109104
   Status: Pending
   Key Usage: General Purpose
   Certificate Request Fingerprint MD5: 2EDBC8C4 D2571528 7AAEDDE6 16585A93 
   Certificate Request Fingerprint SHA1: 087B82E3 4ABA8B92 54FD8539 B59C46AF 8AA480D0 
   Associated Trustpoint: R12CA 

R15#
```

Зайдём теперь на CA **R12** и просмотрим запросы:

```text
R12CA#show crypto pki server R12CA requests 
Enrollment Request Database:

Subordinate CA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

RA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
1      pending    2EDBC8C4D25715287AAEDDE616585A93 serialNumber=67109104+hostname=R15.otus.ru

R12CA#
```

Одобрим запрос:

```text
R12CA#crypto pki server R12CA grant 1
R12CA#
R12CA#
R12CA#show crypto pki server R12CA requests 
Enrollment Request Database:

Subordinate CA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

RA certificate requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
1      granted    2EDBC8C4D25715287AAEDDE616585A93 serialNumber=67109104+hostname=R15.otus.ru

R12CA#
```

Теперь **R15** получил сертификат:

```text
R15#show crypto pki certificate R12CA
Certificate
  Status: Available
  Certificate Serial Number (hex): 02
  Certificate Usage: General Purpose
  Issuer: 
    cn=R12CA
  Subject:
    Name: R15.otus.ru
    Serial Number: 67109104
    serialNumber=67109104+hostname=R15.otus.ru
  Validity Date: 
    start date: 03:08:21 UTC Sep 21 2025
    end   date: 03:08:21 UTC Sep 21 2026
  Associated Trustpoints: R12CA 

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer: 
    cn=R12CA
  Subject: 
    cn=R12CA
  Validity Date: 
    start date: 01:19:16 UTC Sep 21 2025
    end   date: 01:19:16 UTC Sep 20 2028
  Associated Trustpoints: R12CA 

R15#
```

Чтобы не одобрять вручную каждый запрос включим автоматическое одобрение
(только для учебных задач):

```text
R12CA(config)#crypto pki server R12CA
R12CA(cs-server)#shut
Certificate server 'shut' event has been queued for processing.
R12CA(cs-server)#
Sep 21 03:11:35.739: %PKI-6-CS_DISABLED: Certificate server now disabled.
R12CA(cs-server)#grant auto
R12CA(cs-server)#no
Sep 21 03:11:50.915: %PKI-6-CS_GRANT_AUTO: All enrollment requests will be automatically granted.
R12CA(cs-server)#no shut
Certificate server 'no shut' event has been queued for processing.
R12CA(cs-server)#
Sep 21 03:11:55.008: %PKI-6-CS_ENABLED: Certificate server now enabled.
R12CA(cs-server)#end
R12CA#
```

Аналогично настроим получение сертификатов на **R14** и **R18**.

### Включение шифрования

Создадим политику ISAKMP:

```text
R15(config)#crypto isakmp policy 10
R15(config-isakmp)#exit
R15(config)#
```

Создадим transform set:

```text
R15(config)#crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
R15(cfg-crypto-trans)#
```

Так как GRE обеспечивает создание туннеля, то переводим IPSEC в транспортный режим:

```text
R15(cfg-crypto-trans)#mode transport 
R15(cfg-crypto-trans)#exit
R15(config)#
```

Создаём IPSEC-профиль:

```text
R15(config)#crypto ipsec profile IPSEC_GRE
R15(ipsec-profile)#set transform-set IPSEC
R15(ipsec-profile)#exit
R15(config)#
```

Применяем профиль на туннельный интерфейс:

```text
R15(config)#int tunnel 100
R15(config-if)#tunnel protection ipsec profile IPSEC_GRE
R15(config-if)#
Sep 21 04:12:43.615: %CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON
R15(config-if)#end
R15#
```

Аналогично настроим **R14** и **R18**.

На **R18** видно оба туннеля, основной и резервный:

```text
R18#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
10.0.0.18       10.0.0.15       QM_IDLE           1002 ACTIVE
10.0.0.18       10.0.0.14       QM_IDLE           1003 ACTIVE
10.0.0.15       10.0.0.18       QM_IDLE           1001 ACTIVE

IPv6 Crypto ISAKMP SA

R18#
```

## Настройка DMVPN поверх IPSEC

Для DMVPN все шаги аналогичны: на роутерах получаем сертификаты от CA,
настраиваем IPSEC и применяем на туннельный интерфейс.

Сведения об IPSEC на **R15**:

```text
R15#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
10.0.0.18       10.0.0.15       QM_IDLE           1002 ACTIVE
80.80.100.12    80.110.100.11   QM_IDLE           1005 ACTIVE
80.80.100.12    80.110.100.15   QM_IDLE           1004 ACTIVE
10.0.0.15       10.0.0.18       QM_IDLE           1001 ACTIVE

IPv6 Crypto ISAKMP SA

R15#
```

Информация о DMVPN туннеле:

<details>
<summary>R15#show crypto ipsec sa int tunnel 200
</summary>

```text
R15#show crypto ipsec sa int tunnel 200

interface: Tunnel200
    Crypto map tag: Tunnel200-head-0, local addr 80.80.100.12

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (80.80.100.12/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (80.110.100.15/255.255.255.255/47/0)
   current_peer 80.110.100.15 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 9, #pkts encrypt: 9, #pkts digest: 9
    #pkts decaps: 9, #pkts decrypt: 9, #pkts verify: 9
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 80.80.100.12, remote crypto endpt.: 80.110.100.15
     plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb (none)
     current outbound spi: 0x527545FC(1383417340)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0x533EE7E4(1396631524)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 11, flow_id: SW:11, sibling_flags 80004000, crypto map: Tunnel200-head-0
        sa timing: remaining key lifetime (k/sec): (4248313/3377)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x527545FC(1383417340)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 12, flow_id: SW:12, sibling_flags 80004000, crypto map: Tunnel200-head-0
        sa timing: remaining key lifetime (k/sec): (4248313/3377)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     outbound ah sas:

     outbound pcp sas:

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (80.80.100.12/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (80.110.100.11/255.255.255.255/47/0)
   current_peer 80.110.100.11 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 15, #pkts encrypt: 15, #pkts digest: 15
    #pkts decaps: 15, #pkts decrypt: 15, #pkts verify: 15
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 80.80.100.12, remote crypto endpt.: 80.110.100.11
     plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb (none)
     current outbound spi: 0x5897A5F2(1486333426)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0x102933B8(271135672)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 13, flow_id: SW:13, sibling_flags 80000000, crypto map: Tunnel200-head-0
        sa timing: remaining key lifetime (k/sec): (4275241/3406)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x5897A5F2(1486333426)
        transform: esp-aes esp-sha-hmac ,
        in use settings ={Transport, }
        conn id: 14, flow_id: SW:14, sibling_flags 80000000, crypto map: Tunnel200-head-0
        sa timing: remaining key lifetime (k/sec): (4275241/3406)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)
          
     outbound ah sas:

     outbound pcp sas:
R15#
```

</details>

## Проверка IP связности


Проверим маршрутизацию между **VPC**:

```text
VPC1> trace 192.168.30.8
trace to 192.168.30.8, 8 hops max, press Ctrl+C to stop
 1   192.168.10.4   0.603 ms  0.650 ms  0.964 ms
 2   172.16.10.15   0.977 ms  0.740 ms  0.620 ms
 3   172.16.10.9   1.526 ms  1.227 ms  1.279 ms
 4   172.16.100.18   2.473 ms  2.350 ms  1.995 ms
 5   172.16.20.0   3.082 ms  2.392 ms  2.649 ms
 6   *192.168.30.8   3.730 ms (ICMP type:3, code:3, Destination port unreachable)

VPC1>
```
Видим, что траффик между VPC офисов «Москва» и «Санкт-Петербург» идёт через GRE
тоннель.

Также можно посмотреть, что в GRE туннеле на **R18** увеличились счётчики зашифрованных
пакетов:

```text
R18#show crypto ipsec sa int tunnel 100 | inc pkts
    #pkts encaps: 18, #pkts encrypt: 18, #pkts digest: 18
    #pkts decaps: 18, #pkts decrypt: 18, #pkts verify: 18
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
R18#
```

```text
VPC1> trace 192.168.50.30
trace to 192.168.50.30, 8 hops max, press Ctrl+C to stop
 1   192.168.10.4   0.915 ms  0.606 ms  0.550 ms
 2   172.16.10.15   1.177 ms  1.457 ms  0.840 ms
 3   172.16.10.9   1.293 ms  1.343 ms  1.551 ms
 4   172.16.200.28   2.972 ms  2.549 ms  3.016 ms
 5   *192.168.50.30   4.514 ms (ICMP type:3, code:3, Destination port unreachable)

VPC1>
```

Трафик в клиентскую сеть офиса «Чокурдах» идёт через DMVPN.

```text
VPC30> trace 192.168.10.1
trace to 192.168.10.1, 8 hops max, press Ctrl+C to stop
 1   192.168.50.28   0.910 ms  1.012 ms  0.692 ms
 2   172.16.200.14   3.462 ms  2.357 ms  2.201 ms
 3   172.16.10.2   2.930 ms  2.361 ms  2.966 ms
 4   172.16.10.16   2.956 ms  2.398 ms  5.156 ms
 5   *192.168.10.1   4.366 ms (ICMP type:3, code:3, Destination port unreachable)

VPC30>
```

Счётчики DMVPN туннеля 200 на хабе **R14** (два пира):

```text
R15#show crypto ipsec sa int tunnel 200 | inc pkts
    #pkts encaps: 66, #pkts encrypt: 66, #pkts digest: 66
    #pkts decaps: 55, #pkts decrypt: 55, #pkts verify: 55
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #pkts encaps: 57, #pkts encrypt: 57, #pkts digest: 57
    #pkts decaps: 58, #pkts decrypt: 58, #pkts verify: 58
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
R15#
```

## Файлы настроек

Файлы настроек устройств (конфиги) экспортированы в каталог [configs](./configs/).
Правда по какой-то причине в конфигах сертификаты сохранились в виде ссылок на
файл в nvram, хотя `show run` они были выведены текстом целиком.

Готовая лабораторная (экспорт из EVE-NG) - [41_ipsec_dmvpn.zip](./41_ipsec_dmvpn.zip).
