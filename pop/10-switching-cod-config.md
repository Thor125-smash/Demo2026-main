# 10. Настройка коммутации между switch1-cod и switch2-cod

[← Вернуться к оглавлению](../README.md) | [← Предыдущий модуль](09-fwcod-vlan-config.md) | [Следующий модуль →](11-hosts-ip-config.md)

---

## switch1-cod — Базовая настройка

### Назначение имени устройства

Для назначения имени устройства согласно топологии используйте команду:

```bash
hostnamectl set-hostname switch1-cod.domain; exec bash
```

Также рекомендуется указать имя в файле `/etc/sysconfig/network`:

```bash
nano /etc/sysconfig/network
```

Укажите имя в параметре `HOSTNAME`:

![Hostname config](../images/sw1-cod_hostname_config.png)

Проверьте результат командой:

```bash
hostname -f
```

![Hostname check](../images/sw1-cod_hostname_check.png)

---

## firewall — Настройка авторизации

>
### Создание пользователя

Откройте браузер и перейдите по адресу **https://172.16.1.2:8443**.

Перейдите в **Пользователи → Учётные записи** и нажмите **Добавить пользователя**:

![Users menu](../images/fw-cod_users_menu.png)

Заполните форму добавления пользователя:

![Add user form](../images/fw-cod_add_user_form.png)

| Параметр | Значение |
|----------|----------|
| Имя пользователя | network |
| Логин | network |
| Пароль | (оставить сгенерированный) |

> 

Нажмите **Добавить**.

Результат успешного добавления пользователя:

![User result](../images/fw-cod_user_result.png)

### Настройка авторизации по подсетям

Перейдите в **Пользователи → Авторизация → ПО ПОДСЕТЯМ** и нажмите **+ Добавить**:

![Auth subnet menu](../images/fw-cod_auth_subnet_menu.png)

В форме **Добавление правила авторизации** укажите:

| Параметр | Значение |
|----------|----------|
| Пользователь | network |
| Подсеть | 192.168.0.0/16 |

Нажмите **Добавить**.


---

## switch1-cod — Настройка коммутации

### Подготовка интерфейсов

Проверьте интерфейсы и определите их направление (сверка по MAC-адресам):

```bash
ip -c -br a
```

![Interfaces down](../images/sw1-cod_ifaces_down.png)

В данном примере:
- `ens19` — интерфейс в сторону fw-cod
- `ens21` — интерфейс в сторону sw2-cod
- `ens22` — интерфейс в сторону sw2-cod
- `enp2s1` — интерфейс в сторону srv1-cod
- `enp2s29` — интерфейс в сторону admin-cod
- `enp3s12` — интерфейс в сторону srv2-cod

Проверьте наличие директорий для интерфейсов:

```bash
ls /etc/net/ifaces/
```

![Interfaces list](../images/sw1-cod_ifaces_list.png)

Для каждого интерфейса создайте директорию и файл `options`:

```bash
# Создание директорий
mkdir -p /etc/net/ifaces/{ens19,ens21,ens22,enp2s1,enp2s29,enp3s12}

# Создание файлов options для каждого интерфейса
for iface in ens19 ens21 ens22 enp2s1 enp2s29 enp3s12; do
    cat > /etc/net/ifaces/$iface/options << EOF
TYPE=eth
BOOTPROTO=static
EOF
done
```

Перезагрузите службу network:

```bash
systemctl restart network
```

Проверьте, что все интерфейсы перешли в статус UP:

```bash
ip -c -br a
```

![Interfaces up](../images/sw1-cod_ifaces_up.png)

✅ Все интерфейсы в статусе UP.

---

### Установка Open vSwitch

Временно создайте подинтерфейс с VLAN 300 для доступа в Интернет:

```bash
ip link add link ens19 name ens19.300 type vlan id 300
ip link set dev ens19.300 up
ip addr add 192.168.x.1/24 dev ens19.300
ip route add 0.0.0.0/0 via 192.168.x.254
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

Обновите список пакетов и установите Open vSwitch:

```bash
apt-get update && apt-get install -y openvswitch
```

Включите и добавьте в автозагрузку:

```bash
systemctl enable --now openvswitch
```

Отключите автоматическое удаление настроек OVS при перезагрузке:

```bash
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```

Перезагрузите сервер:

```bash
reboot
```

После перезагрузки создайте временный коммутатор для доступа sw2-cod к Интернету:

```bash
ovs-vsctl add-br br0
ovs-vsctl add-port br0 ens19
ovs-vsctl add-port br0 ens21
```

---

## switch2-cod — Базовая настройка

### Назначение имени устройства

Аналогично switch1-cod:

```bash
hostnamectl set-hostname switch2-cod.domain; exec bash
```

Проверьте результат:

```bash
hostname -f
```

![sw2-cod hostname](../images/sw2-cod_hostname_check.png)

### Установка Open vSwitch

Создайте временный подинтерфейс для доступа в Интернет:

```bash
ip link add link ens19 name ens19.300 type vlan id 300
ip link set up ens19
ip link set up ens19.300
ip addr add 192.168.x.2/24 dev ens19.300
ip route add 0.0.0.0/0 via 192.168.x.254
echo "nameserver 77.88.8.8" > /etc/resolv.conf
```

Установите Open vSwitch:

```bash
apt-get update && apt-get install -y openvswitch
systemctl enable --now openvswitch
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
reboot
```

---

## switch1-cod — Конфигурация Open vSwitch

### Создание коммутатора

Удалите временный коммутатор и создайте основной:

```bash
ovs-vsctl del-br br0
ovs-vsctl add-br switch1-cod
```

Проверьте создание коммутатора:

```bash
ovs-vsctl show
```

![OVS bridge](../images/sw1-cod_ovs_bridge.png)

### Добавление access-портов

Добавьте интерфейсы как access-порты с указанием VLAN:

```bash
# admin-cod (VLAN 300)
ovs-vsctl add-port switch1-cod enp2s29 tag=300

# srvs1-cod (VLAN 100)
ovs-vsctl add-port switch1-cod enp2s1 tag=100

# srvs2-cod (VLAN 200)
ovs-vsctl add-port switch1-cod enp3s12 tag=200
```

### Добавление trunk-порта

Добавьте интерфейс в сторону fw-cod как trunk-порт:

```bash
ovs-vsctl add-port switch1-cod ens19 trunk=100,200,300,400,500
```

Проверьте добавление портов:

```bash
ovs-vsctl show
```

![OVS ports](../images/sw1-cod_ovs_ports.png)

### Создание bond-интерфейса

Включите модуль ядра для тегированного трафика (802.1Q):

```bash
modprobe 8021q
echo "8021q" | tee -a /etc/modules
```

Создайте bond-интерфейс в режиме active-backup:

```bash
ovs-vsctl add-bond switch1-cod bond0 ens21 ens22 bond_mode=active-backup
```

Настройте bond как trunk-порт:

```bash
ovs-vsctl set port bond0 trunk=100,200,300,400,500
```

Проверьте конфигурацию:

```bash
ovs-vsctl show
```

![OVS bond](../images/sw1-cod_ovs_bond.png)

### Настройка management-интерфейса

Создайте директорию для management-интерфейса:

```bash
mkdir /etc/net/ifaces/mgmt-cod
```

Создайте файл `options`:

```bash
vim /etc/net/ifaces/mgmt-cod/options
```

![mgmt options](../images/sw1-cod_mgmt_options.png)

Содержимое файла:

```
TYPE=ovsport
BOOTPROTO=static
CONFIG_IPV4=yes
BRIDGE=switch1-cod
VID=300
```

| Параметр | Описание |
|----------|----------|
| TYPE | Тип интерфейса (ovsport) |
| BOOTPROTO | Способ назначения сетевых параметров |
| CONFIG_IPV4 | Использовать конфигурацию IPv4 |
| BRIDGE | Мост, к которому добавляется интерфейс |
| VID | Принадлежность к VLAN |

Назначьте IP-адрес и шлюз:

```bash
echo "192.168.x.1/24" > /etc/net/ifaces/mgmt-cod/ipv4address
echo "default via 192.168.x.254" > /etc/net/ifaces/mgmt-cod/ipv4route
```

Перезагрузите службу network:

```bash
systemctl restart network
```

Проверьте назначенный IP-адрес:

```bash
ip -c -br -4 a
```

![mgmt IP](../images/sw1-cod_mgmt_ip.png)

Проверьте маршрут по умолчанию:

```bash
ip -c r
```

![mgmt route](../images/sw1-cod_mgmt_route.png)

Проверьте, что интерфейс добавился в коммутатор:

```bash
ovs-vsctl show
```

![OVS mgmt](../images/sw1-cod_ovs_mgmt.png)

Настройте NativeVLAN для management-интерфейса:

```bash
ovs-vsctl set port mgmt-cod vlan_mode=native-untagged
```

Проверьте настройку:

```bash
ovs-vsctl list port mgmt-cod
```

![mgmt native](../images/sw1-cod_mgmt_native.png)

---

## switch2-cod — Конфигурация Open vSwitch

### Подготовка интерфейсов switch2-cod

Проверьте интерфейсы и определите их направление (сверка по MAC-адресам):

```bash
ip -c -br a
```

![sw2-cod interfaces down](../images/sw2-cod_ifaces_down.png)

```bash
# Создание директорий
mkdir -p /etc/net/ifaces/{ens19,ens20,ens21,ens22,enp2s29,enp3s12}

# Создание файлов options для каждого интерфейса
for iface in ens19 ens20 ens21 ens22 enp2s29 enp3s12; do
    cat > /etc/net/ifaces/$iface/options << EOF
TYPE=eth
BOOTPROTO=static
EOF
done
```

Перезагрузите службу network:

```bash
systemctl restart network
```

Проверьте, что все интерфейсы перешли в статус UP:

```bash
ip -c -br a
```

![sw2-cod interfaces up](../images/sw2-cod_ifaces_up.png)

### Создание коммутатора switch2-cod

Создайте коммутатор с именем switch2-cod:

```bash
ovs-vsctl add-br switch2-cod
```

Проверьте создание коммутатора:

```bash
ovs-vsctl show
```

![sw2-cod OVS bridge](../images/sw2-cod_ovs_bridge.png)

### Добавление access-портов switch2-cod

Добавьте интерфейсы как access-порты с указанием VLAN:

```bash
# srvs1-cod (VLAN 100)
ovs-vsctl add-port switch2-cod ens21 tag=100

# srvs1-cod (VLAN 200)
ovs-vsctl add-port switch2-cod ens22 tag=200

# sips-cod (VLAN 500)
ovs-vsctl add-port switch2-cod enp2s29 tag=500

# client-cod (VLAN 400)
ovs-vsctl add-port switch2-cod enp3s12 tag=400
```

Проверьте добавление портов:

```bash
ovs-vsctl show
```

![sw2-cod OVS ports](../images/sw2-cod_ovs_ports.png)

### Создание bond-интерфейса switch2-cod

Включите модуль ядра для тегированного трафика (802.1Q):

```bash
modprobe 8021q
echo "8021q" | tee -a /etc/modules
```

Создайте bond-интерфейс в режиме active-backup на базе интерфейсов ens19 и ens20:

```bash
ovs-vsctl add-bond switch2-cod bond0 ens19 ens20 bond_mode=active-backup
```

Настройте bond как trunk-порт:

```bash
ovs-vsctl set port bond0 trunk=100,200,300,400,500
```

Проверьте конфигурацию:

```bash
ovs-vsctl show
```

![sw2-cod OVS bond](../images/sw2-cod_ovs_bond.png)

Проверьте режим работы bond-интерфейса:

```bash
ovs-appctl bond/show
```

![sw2-cod bond show](../images/sw2-cod_bond_show.png)

### Настройка management-интерфейса switch2-cod

Создайте директорию для management-интерфейса:

```bash
mkdir /etc/net/ifaces/mgmt-cod
```

Создайте файл `options`:

```bash
nano /etc/net/ifaces/mgmt-cod/options
```

![sw2-cod mgmt options](../images/sw2-cod_mgmt_options.png)

Содержимое файла:

```
TYPE=ovsport
BOOTPROTO=static
CONFIG_IPV4=yes
BRIDGE=switch2-cod
VID=300
```

| Параметр | Описание |
|----------|----------|
| TYPE | Тип интерфейса (ovsport) |
| BOOTPROTO | Способ назначения сетевых параметров |
| CONFIG_IPV4 | Использовать конфигурацию IPv4 |
| BRIDGE | Мост, к которому добавляется интерфейс |
| VID | Принадлежность к VLAN |

Назначьте IP-адрес и шлюз:

```bash
echo "192.168.x.2/24" > /etc/net/ifaces/mgmt-cod/ipv4address
echo "default via 192.168.x.254" > /etc/net/ifaces/mgmt-cod/ipv4route
```

Перезагрузите службу network:

```bash
systemctl restart network
```

Проверьте назначенный IP-адрес:

```bash
ip -c -br -4 a
```

![sw2-cod mgmt IP](../images/sw2-cod_mgmt_ip.png)

Проверьте маршрут по умолчанию:

```bash
ip -c r
```

![sw2-cod mgmt route](../images/sw2-cod_mgmt_route.png)

Проверьте, что интерфейс добавился в коммутатор:

```bash
ovs-vsctl show
```

![sw2-cod OVS mgmt](../images/sw2-cod_ovs_mgmt.png)

Настройте NativeVLAN для management-интерфейса:

```bash
ovs-vsctl set port mgmt-cod vlan_mode=native-untagged
```

Проверьте настройку:

```bash
ovs-vsctl list port mgmt-cod
```

![sw2-cod mgmt native](../images/sw2-cod_mgmt_native.png)

---

## Итоговая конфигурация

### switch1-cod — Open vSwitch

| Порт | Интерфейс | Тип | VLAN/Trunk |
|------|-----------|-----|------------|
| enp2s29 | admin-cod | access | 300 |
| enp2s1 | srv1-cod | access | 100 |
| enp3s12 | srv2-cod | access | 200 |
| ens19 | fw-cod | trunk | 100,200,300,400,500 |
| bond0 | sw2-cod | trunk | 100,200,300,400,500 |
| mgmt-cod | management | internal | 300 (native-untagged) |

### switch1-cod — Сетевые параметры

| Параметр | Значение |
|----------|----------|
| Hostname | sw1-cod.cod.ssa2026.region |
| Management IP | 192.168.30.1/24 |
| Gateway | 192.168.30.254 |
| VLAN | 300 (MGMT-COD) |

### switch2-cod — Open vSwitch

| Порт | Интерфейс | Тип | VLAN/Trunk |
|------|-----------|-----|------------|
| ens21 | srv1-cod | access | 100 |
| ens22 | srv1-cod | access | 200 |
| enp2s29 | sip-cod | access | 500 |
| enp3s12 | cli-cod | access | 400 |
| bond0 | sw1-cod | trunk | 100,200,300,400,500 |
| mgmt-cod | management | internal | 300 (native-untagged) |

### switch2-cod — Сетевые параметры

| Параметр | Значение |
|----------|----------|
| Hostname | sw2-cod.cod.ssa2026.region |
| Management IP | 192.168.30.2/24 |
| Gateway | 192.168.30.254 |
| VLAN | 300 (MGMT-COD) |

---

[← Вернуться к оглавлению](../README.md) | [← Предыдущий модуль](09-fwcod-vlan-config.md) | [Следующий модуль →](11-hosts-ip-config.md)
