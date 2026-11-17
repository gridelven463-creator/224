# Настройка сети RTR (Alt Server) + CLI (Alt Workstation)

Пошаговая инструкция по выполнению экзаменационного задания (время на выполнение — 90 минут).

## Топология

```
   [ Интернет ]
        │
        │ WAN (ens18, DHCP)
     ┌──┴──┐
     │ RTR │  Alt Server
     └──┬──┘
        │ LAN (ens19, 192.168.10.65/26)
        │
     ┌──┴──┐
     │ CLI │  Alt Workstation
     └─────┘
```

| Устройство | ОС |
|---|---|
| RTR | Alt Server |
| CLI | Alt Workstation |

## Условные обозначения

> ⚠️ В задании подсеть, имена устройств и текст баннера указаны в билете. В инструкции использованы **примеры** — замените их своими значениями.

| Параметр | Значение в примере |
|---|---|
| Подсеть из задания 1 | `192.168.10.0/26` (пример адреса из билета: `192.168.10.100/26`) |
| Адрес сети | `192.168.10.64/26` |
| Адрес RTR (LAN) | `192.168.10.65` |
| Диапазон DHCP | `192.168.10.66 – 192.168.10.126` |
| WAN-интерфейс RTR | `ens18` |
| LAN-интерфейс RTR | `ens19` |
| Имя RTR / CLI | `rtr` / `cli` |
| DNS-суффикс | `politech-nsk.local` |

Реальные имена интерфейсов смотрите командой `ip -c a`.

## Содержание

1. [Расчёт сети](#задание-1-расчёт-сети)
2. [Базовая настройка RTR](#задание-2-базовая-настройка-rtr)
3. [Пользователь user_ssh](#задание-3-пользователь-user_ssh)
4. [Безопасный SSH](#задание-4-безопасный-ssh)
5. [DHCP-сервер](#задание-5-dhcp-сервер)
6. [Кэширующий DNS](#задание-6-кэширующий-dns-bind)
7. [NAT](#задание-7-динамическая-трансляция-адресов-nat)
8. [Настройка CLI](#задание-8-настройка-cli)
9. [Часовой пояс и время](#задание-9-часовой-пояс-и-время)
10. [Итоговый чек-лист](#итоговый-чек-лист)

## Рекомендуемый порядок и время

| Этап | Время |
|---|---|
| Задание 1 | 5 мин |
| Задания 2–7 (RTR) | ~45 мин |
| Задание 8 (CLI) | ~20 мин |
| Задание 9 | 5 мин |
| Итоговая проверка | 10 мин |

> 💡 Установку Яндекс Браузера запускайте как можно раньше — она долгая.

---

## Задание 1. Расчёт сети

Допустим, в билете дан адрес `192.168.10.100/26`.

1. Маска `/26` = `255.255.255.192`. Хостовых бит: 32 − 26 = 6.
2. Размер блока: 2⁶ = 64 адреса. Сети идут шагом 64: `.0`, `.64`, `.128`, `.192`.
3. Адрес `.100` попадает в блок `.64 – .127`.

| Параметр | Как считать | Результат |
|---|---|---|
| Адрес сети | IP AND маска (начало блока) | `192.168.10.64` |
| Широковещательный адрес | конец блока | `192.168.10.127` |
| Host min | сеть + 1 | `192.168.10.65` |
| Host max | broadcast − 1 | `192.168.10.126` |
| Количество хостов | 2^(32 − prefix) − 2 | `62` |

---

## Задание 2. Базовая настройка RTR

Установите Alt Server. У ВМ должно быть **две** сетевые карты: первая — в сторону провайдера, вторая — во внутреннюю сеть, общую с CLI.

### WAN (DHCP)

```bash
mkdir -p /etc/net/ifaces/ens18
cat > /etc/net/ifaces/ens18/options <<'EOF'
TYPE=eth
BOOTPROTO=dhcp
DISABLED=no
NM_CONTROLLED=no
EOF
```

### LAN (статический адрес из сети задания 1)

```bash
mkdir -p /etc/net/ifaces/ens19
cat > /etc/net/ifaces/ens19/options <<'EOF'
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
EOF
echo "192.168.10.65/26" > /etc/net/ifaces/ens19/ipv4address
```

### Включение маршрутизации

```bash
sed -i 's/^net.ipv4.ip_forward.*/net.ipv4.ip_forward = 1/' /etc/net/sysctl.conf
grep ip_forward /etc/net/sysctl.conf
```

### Имя устройства и перезапуск сети

```bash
hostnamectl set-hostname rtr
systemctl enable --now network
systemctl restart network
```

### Проверка

```bash
ip -c a            # на ens18 адрес от провайдера, на ens19 — 192.168.10.65/26
ping -c3 ya.ru
hostname
```

---

## Задание 3. Пользователь user_ssh

```bash
useradd -m user_ssh
echo "user_ssh:Qwer1234" | chpasswd
usermod -aG wheel user_ssh

apt-get update && apt-get install -y sudo
control sudowheel enabled

echo "user_ssh ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/user_ssh
chmod 440 /etc/sudoers.d/user_ssh
visudo -c
```

### Проверка

```bash
su - user_ssh
sudo whoami        # должно вывести root без запроса пароля
```

---

## Задание 4. Безопасный SSH

Конфигурация в Alt: `/etc/openssh/sshd_config`.

```bash
cat > /etc/openssh/banner <<'EOF'
ТЕКСТ БАННЕРА ИЗ БИЛЕТА
EOF

sed -i -E 's/^#?\s*Port .*/Port 2224/' /etc/openssh/sshd_config
sed -i -E 's/^#?\s*MaxAuthTries .*/MaxAuthTries 2/' /etc/openssh/sshd_config
sed -i -E 's|^#?\s*Banner .*|Banner /etc/openssh/banner|' /etc/openssh/sshd_config
grep -q '^AllowUsers' /etc/openssh/sshd_config || echo "AllowUsers user_ssh" >> /etc/openssh/sshd_config

grep -E '^(Port|MaxAuthTries|Banner|AllowUsers)' /etc/openssh/sshd_config
sshd -t && systemctl enable --now sshd && systemctl restart sshd
ss -tlnp | grep 2224
```

### Проверка

```bash
ssh -p 2224 user_ssh@192.168.10.65   # виден баннер, вход проходит
ssh -p 2224 root@192.168.10.65       # отказ
```

Две неверные попытки пароля подряд должны разорвать соединение.

---

## Задание 5. DHCP-сервер

```bash
apt-get install -y dhcp-server
echo 'DHCPDARGS="ens19"' > /etc/sysconfig/dhcpd

cat > /etc/dhcp/dhcpd.conf <<'EOF'
authoritative;
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.10.64 netmask 255.255.255.192 {
    range 192.168.10.66 192.168.10.126;
    option routers 192.168.10.65;
    option subnet-mask 255.255.255.192;
    option broadcast-address 192.168.10.127;
    option domain-name-servers 192.168.10.65;
    option domain-name "politech-nsk.local";
    option domain-search "politech-nsk.local";
}
EOF

systemctl enable --now dhcpd
systemctl status dhcpd --no-pager
```

Адрес роутера `.65` не входит в диапазон `.66 – .126`, поэтому он исключён из выдачи.

### Проверка

```bash
journalctl -u dhcpd -n 20
```

Когда CLI запросит адрес, в логе появятся `DHCPDISCOVER` / `DHCPOFFER` / `DHCPACK`.

---

## Задание 6. Кэширующий DNS (bind)

```bash
apt-get install -y bind bind-utils
```

В файле `/etc/bind/options.conf` приведите блок `options` к виду (остальное оставьте как есть):

```
options {
    listen-on { 127.0.0.1; 192.168.10.65; };
    listen-on-v6 { none; };
    directory "/etc/bind/zone";
    dump-file "/var/run/named/named_dump.db";
    statistics-file "/var/run/named/named.stats";
    recursing-file "/var/run/named/named.recursing";
    secroots-file "/var/run/named/named.secroots";

    forward first;
    forwarders { 77.88.8.8; 8.8.8.8; };

    allow-query { 127.0.0.1; 192.168.10.64/26; };
    allow-recursion { 127.0.0.1; 192.168.10.64/26; };
    dnssec-validation no;
};
```

```bash
named-checkconf
systemctl enable --now bind
systemctl status bind --no-pager
```

### Проверка

```bash
dig @192.168.10.65 ya.ru     # статус NOERROR
dig @192.168.10.65 ya.ru     # повторный запрос быстрее (Query time ~0 ms)
```

---

## Задание 7. Динамическая трансляция адресов (NAT)

```bash
apt-get install -y iptables

iptables -t nat -A POSTROUTING -s 192.168.10.64/26 -o ens18 -j MASQUERADE

iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

### Проверка

```bash
iptables -t nat -L -n -v
cat /proc/sys/net/ipv4/ip_forward    # должно быть 1
systemctl is-active iptables         # после перезагрузки правило должно сохраниться
```

---

## Задание 8. Настройка CLI

1. Установите **Alt Workstation**. Сетевую карту CLI подключите к той же внутренней сети, что и `ens19` на RTR.
2. В настройках сети выберите «Автоматически (DHCP)» (обычно включено по умолчанию).
3. Задайте имя и проверьте сеть (от root):

```bash
hostnamectl set-hostname cli

ip -c a                 # адрес из диапазона 192.168.10.66–126
ip r                    # default via 192.168.10.65
cat /etc/resolv.conf    # nameserver 192.168.10.65, search politech-nsk.local
ping -c3 192.168.10.65
ping -c3 8.8.8.8
ping -c3 ya.ru
```

4. Установите Яндекс Браузер:

```bash
apt-get update
apt-get install -y yandex-browser-stable
```

Если пакета нет в репозитории:

```bash
epm play yandex-browser
```

5. Убедитесь, что браузер запускается и открывает сайты.

> Если CLI не получил адрес: `systemctl restart NetworkManager`. Если не помогло — смотрите `journalctl -u dhcpd` на RTR.

---

## Задание 9. Часовой пояс и время

Новосибирск — UTC+7. Выполните на **обоих** устройствах (RTR и CLI) от root:

```bash
timedatectl set-timezone Asia/Novosibirsk
apt-get install -y chrony
systemctl enable --now chronyd
timedatectl set-ntp true
chronyc makestep
timedatectl
```

В выводе должно быть:

```
Time zone: Asia/Novosibirsk (+07, +0700)
System clock synchronized: yes
```

Если NTP недоступен — задайте время вручную:

```bash
timedatectl set-ntp false
timedatectl set-time "2026-11-17 18:30:00"   # текущее новосибирское время
hwclock --systohc
```

---

## Итоговый чек-лист

- [ ] Задание 1: рассчитаны адрес сети, host min/max, broadcast, количество хостов
- [ ] RTR: WAN получает адрес по DHCP, LAN = `192.168.10.65/26`, имя задано
- [ ] `user_ssh` выполняет `sudo` без пароля
- [ ] SSH: порт 2224, только `user_ssh`, `MaxAuthTries 2`, баннер виден
- [ ] DHCP: CLI получил адрес, шлюз, DNS и суффикс `politech-nsk.local`
- [ ] DNS: `dig @192.168.10.65 ya.ru` работает
- [ ] NAT: CLI пингует `8.8.8.8` и `ya.ru`
- [ ] Яндекс Браузер установлен, имя CLI задано
- [ ] На обоих устройствах часовой пояс `Asia/Novosibirsk`, время синхронизировано
