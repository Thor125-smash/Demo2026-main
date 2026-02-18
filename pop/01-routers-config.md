# 1. Настройка имён и IP-адресации на устройствах

[← Вернуться к оглавлению](../README.md)

---

## Содержание

- [1(EcoRouter)](#rtr-cod-ecorouter)
  - [Назначение имени устройства](#назначение-имени-на-устройство)
  - [Назначение IP-адресов](#назначение-ip-адресов-на-устройство)
- [2(EcoRouter)](#rtr-a-ecorouter)
  - [Назначение имени устройства](#назначение-имени-на-устройство-1)
  - [Назначение IP-адресов](#назначение-ip-адресов-на-устройство-1)
  - [Настройка маршрутизации между VLAN](#настройка-маршрутизации-между-vlan)

---

## cod(EcoRouter)

### Назначение имени на устройство

Для назначения имени устройства согласно требованиям задания используем следующие команды:

1. Переходим в режим администрирования (`enable`)
2. Переходим в режим конфигурации (`configure terminal`)
3. Задаём имя устройству (`hostname <NAME>`)
4. Задаём доменное имя (`ip domain-name <DOMAIN_NAME>`)
5. Сохраняем конфигурацию (`write memory`)

```
ecorouter>enable
ecorouter#configure terminal 
Enter configuration commands, one per line.  End with CNTL/Z.
ecorouter(config)#hostname введите имя устройства
cod(config)#ip domain-name Введите доменное имя
cod(config)#write memory
Building configuration...


```

#### Проверка имени устройства

Проверить имя устройства можно командой `show hostname` из режима администрирования (enable):

![Проверка hostname](../images/rtr-cod_hostname.png)

#### Проверка доменного имени

Проверить доменное имя устройства можно командой `show running-config | include domain-name` из режима администрирования (enable):

![Проверка domain-name](../images/rtr-cod_domain.png)

---

### Назначение IP-адресов на устройство

#### Основные понятия EcoRouter

| Уровень | Компонент | Описание |
|---------|-----------|----------|
| L1 | **Порт (port)** | Устройство в составе EcoRouter, работает на физическом уровне |
| L3 | **Интерфейс (interface)** | Логический интерфейс для адресации, работает на сетевом уровне |
| L2 | **Service instance (SI)** | Логический сабинтерфейс, работает на канальном уровне, связывает L1, L2 и L3 |

> **Service instance** необходим для соединения физического порта с интерфейсами L3, интерфейсами bridge, портами. Используется для гибкого управления трафиком на основании наличия меток VLAN в фреймах или их отсутствия. Сквозь сервисный интерфейс проходит весь трафик, приходящий на порт.

#### Алгоритм назначения IPv4-адреса на EcoRouter

1. Создать интерфейс с произвольным именем и назначить на него IPv4-адрес
2. В режиме конфигурирования порта создать service-instance с произвольным именем:
   - Указать (инкапсулировать), будет обрабатываться тегированный или не тегированный трафик
   - Указать, в какой интерфейс (ранее созданный) нужно отправить обработанные кадры

#### Просмотр физических портов

Посмотреть физические порты можно командой `show port brief` из режима администрирования (enable):

![Порты rtr-cod](../images/rtr-cod_ports.png)

- **te0** — направлен в сторону интернета
- **te1** — направлен в сторону firewall

#### Создание интерфейса Internet

Создадим интерфейс с именем и назначим на него IP-адрес `Провайдера`:

```
cod(config)#interface isternet
cod(config-if)#ip address (ip провайдера)
cod(config-if)#description "Connecting to an Internet provider"
cod(config-if)#exit
```

#### Создание интерфейса firewall

Создадим интерфейс с именем `firewall` и назначим на него IP-адрес `Задайте ip`:

```
cod(config)#interface firewall
cod(config-if)#ip address (Назначенный ip)
cod(config-if)#description "Connecting to firewall"
cod(config-if)#exit
cod(config)#
```

#### Проверка интерфейсов (до привязки к портам)

Проверить назначенные IP-адреса можно командой `show ip interface brief`:

![IP интерфейсы - down](../images/rtr-cod_ip_down.png)

> ⚠️ Созданные интерфейсы пока не добавлены в какие-либо Service instance, а значит не привязаны к порту — отсюда статус **down**

#### Создание Service Instance для Internet (порт te0)

В режиме конфигурирования порта `te0` создаём service-instance:

```
cod(config)#port te0
cod(config-port)#service-instance te0/internet
cod(config-service-instance)#encapsulation untagged 
cod(config-service-instance)#connect ip interface internet 
cod(config-service-instance)#exit
cod(config-port)#exit
```

#### Создание Service Instance для firewall (порт te1)

В режиме конфигурирования порта `te1` создаём service-instance:

```
rtr-cod(config)#port te1
rtr-cod(config-port)#service-instance te1/firewall
rtr-cod(config-service-instance)#encapsulation untagged 
rtr-cod(config-service-instance)#connect ip interface firewall 
rtr-cod(config-service-instance)#exit
rtr-cod(config-port)#exit
rtr-cod(config)#write memory
Building configuration...
```

#### Проверка интерфейсов (после привязки к портам)

![IP интерфейсы - up](../images/rtr-cod_ip_up.png)

#### Проверка Service Instance

Проверить созданные Service instance можно командой `show service-instance brief`:

![Service Instance](../images/rtr-cod_service_instance.png)

#### О маршруте по умолчанию

> ⚠️ IP-адрес шлюза по умолчанию на данном устройстве **не задаётся** вручную!
> 
> По условиям задания маршрутизатор ЦОД должен получать маршрут по умолчанию **по BGP**.
> Ручное создание маршрута по умолчанию **ЗАПРЕЩЕНО!**

#### Проверка связности с Internet

![Ping Internet](../images/rtr-cod_ping_isp.png)

---

## a(EcoRouter)

### Назначение имени на устройство

Реализация аналогична cod:

- Задаём имя устройству (hostname <NAME>)
- Задаём доменное имя (ip domain-name <DOMAIN_NAME>)


![Hostname a](../images/rtr-a_hostname.png)

![Domain-name a](../images/rtr-a_domain.png)

---

### Назначение IP-адресов на устройство

Реализация аналогична cod, за исключением того, что на базе физического порта `te1` должны быть созданы интерфейсы и Service instance с целью обработки тегированного трафика для маршрутизации между VLAN.

#### Создание интерфейса Internet

Должен быть создан интерфейс для подключения к интернет-провайдеру Internet с IP-адресом `ip провайдера`.

#### Проверка интерфейса Internet (до и после)

До привязки к порту:

![Internet down](../images/rtr-a_isp_down.png)

После создания Service instance:

![Internet up](../images/rtr-a_isp_up.png)

#### Настройка маршрута по умолчанию

> В отличие от cod, на a по заданию нет требований про настройку BGP, поэтому маршрут по умолчанию можно задать **вручную**.

```
a(config)#ip route 0.0.0.0/0 Шлюз провайдера
a(config)#
```

#### Проверка маршрута по умолчанию

Проверить назначенный маршрут можно командой `show ip route static`:

![Static route](../images/rtr-a_static_route.png)

#### Проверка доступа в Интернет

![Ping Internet](../images/rtr-a_ping_internet.png)

---

### Настройка маршрутизации между VLAN

#### Создание интерфейсов для VLAN

Создаём интерфейсы с произвольными именами для каждого VLAN и назначаем IP-адреса к примеру 172.20.x.x/24

```
a(config)#interface vl100
a(config-if)#ip address Назначить адресс
a(config-if)#description "VLAN - SRV"
a(config-if)#exit

a(config)#interface vl200
a(config-if)#ip address Назначить адресс
a(config-if)#description "VLAN - CLI"
a(config-if)#exit

a(config)#interface vl300
a(config-if)#ip address Назначить адресс
a(config-if)#description "VLAN - MGMT"
a(config-if)#exit
```

#### Проверка VLAN интерфейсов (до привязки)

![VLAN interfaces down](../images/rtr-a_vlans_down.png)

> ⚠️ Созданные интерфейсы пока не добавлены в Service instance — статус **down**

#### Операции над метками в сервисных интерфейсах

Есть три варианта операций над метками:
1. Удаление существующей метки/меток
2. Добавление новой метки (меток)
3. Трансляция метки/меток из одного значения в другое

**Пояснение команд:**

| Команда | Описание |
|---------|----------|
| `encapsulation dot1q <VID> exact` | Указание номера обрабатываемого VLAN. Опция `exact` показывает, что под это правило попадут кадры только с меткой равной `<VID>` |
| `rewrite pop 1` | Снимаем только одну верхнюю метку. На L3 кадр должен поступать без признаков VLAN |

#### Создание Service Instance для VLAN (порт te1)

```
a(config)#port te1

a(config-port)#service-instance te1/vl100
a(config-service-instance)#encapsulation dot1q 100
a(config-service-instance)#rewrite pop 1
a(config-service-instance)#connect ip interface vl100                   
a(config-service-instance)#exit

a(config-port)#service-instance te1/vl200
a(config-service-instance)#encapsulation dot1q 200
a(config-service-instance)#rewrite pop 1
a(config-service-instance)#connect ip interface vl200 
a(config-service-instance)#exit

a(config-port)#service-instance te1/vl300
a(config-service-instance)#encapsulation dot1q 300
a(config-service-instance)#rewrite pop 1
a(config-service-instance)#connect ip interface vl300 
a(config-service-instance)#exit
```

#### Проверка VLAN интерфейсов (после привязки)

![VLAN interfaces up](../images/rtr-a_vlans_up.png)

#### Проверка Service Instance

![Service Instance a](../images/rtr-a_service_instance.png)

---

[← Вернуться к оглавлению](../README.md) | [Следующий модуль →](02-fw-cod-config.md)
