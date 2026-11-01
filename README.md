# Сессионный проект по курсу «Безопасность GNU/Linux»

**Уникальный номер:** 372770 \
**ОС:** РЕД ОС 8.0.3 (Standard Server) \
**Виртуализация:** QEMU/KVM (Proxmox) \
**Дата:** 01.11.2026

---

## Артефакты

| Файл | Раздел | Что проверяется |
|---|---|---|
| `mephi-screenshot.png` | 8.1 | Скриншот с `Hello from Student: 372770` |
| `history.out` | Все | История команд (152 команды) |
| `ping.out` | 1.3 | `ping 8.8.8.8` - 4/4, 0% loss |
| `dnf.out` | 2.3 | История транзакций dnf |
| `stat.out` | 3.2, 5.1 | `stat /data/mephi-2026` (2770 SGID+ACL) и `stat /mephi-web` (httpd_sys_content_t) |
| `journalctl.out` | 4.2 | Журнал nginx за текущую загрузку (11 строк) |
| `getcap.out` | 5.2 | `getcap /usr/sbin/tcpdump` → `cap_net_admin,cap_net_raw=eip` |
| `getenforce.out` | 5.3 | `Enforcing` |
| `curl.out` | 7.2 | `Hello from Student: 372770` |
| `fstab` | 3.2 | Монтирование по `LABEL=MEPHI_WEB` |
| `passwd` | 5.1, 6 | Пользователи |
| `shadow` | 6.2 | Политика паролей |
| `group` | 5.1 | Группы `developers`, `curators` (GID 4444) |
| `pwquality.conf` | 6.2 | `minlen = 12` |

---

## Ход выполнения

### Раздел 1. Установка дистрибутива

**1.1. Настройка сети (DHCP).** Интерфейс `ens18` активен, адрес получен по DHCP.

```bash
nmcli -t -f NAME,DEVICE,TYPE,STATE connection show --active
nmcli -f ipv4.method connection show "ens18"   # ipv4.method: auto
ip -4 addr show ens18                          # inet 10.25.50.45/24 ... dynamic
```

**1.2. Имя хоста.** Установлено `mephi-2026.domain.local` (команда `hostnamectl set-hostname` выполнена до очистки истории; текущее значение подтверждено проверкой).

```bash
hostnamectl | grep "Static hostname"   # Static hostname: mephi-2026.domain.local
hostname -f                            # mephi-2026.domain.local
```

**1.3. Проверка связности.**

```bash
ping -c 4 8.8.8.8 > ~/ping.out
```

Результат: `4 packets transmitted, 4 received, 0% packet loss`.

---

### Раздел 2. Управление программным обеспечением

**2.1. Обновление дистрибутива.**

```bash
sudo dnf update -y
```

Система предварительно обновлена при подготовке ВМ - `dnf` сообщил «Отсутствуют действия для выполнения». Факт обновления подтверждён транзакцией №7 (`update -y`, 178 пакетов) в `dnf history`.

**2.2. Установка пакетов.**

```bash
sudo dnf install -y nginx libcap-ng-utils
```

Пакеты уже установлены (транзакция №8, 5 пакетов). Проверка: `rpm -q nginx libcap-ng-utils` → обе версии.

**2.3. Установка локального RPM.**

```bash
sudo dnf download --destdir=/tmp tcpdump
# Скачано: tcpdump-4.99.5-1.red80.x86_64.rpm и tcpdump-4.99.5-1.red80.i686.rpm

sudo rpm -ivh /tmp/tcpdump-4.99.5-1.red80.x86_64.rpm
# Установлен: tcpdump-14:4.99.5-1.red80

sudo rm -f /tmp/tcpdump-4.99.5-1.red80.i686.rpm
sudo dnf history > ~/dnf.out
```

> **Примечание.** `dnf download` скачал два пакета - `x86_64` и `i686`. Попытка установки обоих (`rpm -ivh /tmp/tcpdump*.rpm`) завершилась ошибкой для `i686` (отсутствуют 32-битные библиотеки `libc.so.6`, `libcrypto.so.3`). Установлен `x86_64`. Проверка: `rpm -q tcpdump` → `tcpdump-4.99.5-1.red80.x86_64`, `tcpdump --version` → 4.99.5.

---

### Раздел 3. Управление файловыми системами

**3.1. Создание ФС на `/dev/sdb`.**

```bash
sudo fdisk /dev/sdb
# n → p → 1 → Enter → Enter → w

sudo mkfs.ext4 -L MEPHI_WEB /dev/sdb1
lsblk -f /dev/sdb
# sdb1 ext4 1.0 MEPHI_WEB <UUID>
```

**3.2. Монтирование по метке.**

```bash
sudo mkdir -p /mephi-web
echo 'LABEL=MEPHI_WEB  /mephi-web  ext4  defaults  0 2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
mount | grep mephi-web
# /dev/sdb1 on /mephi-web type ext4 (rw,relatime,seclabel)
```

**Проверка автозагрузки:** `umount` + `mount -a` → раздел снова примонтирован.

---

### Раздел 4. Управление сервисами

**4.1. nginx - запуск и автозагрузка.**

```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager    # enabled; active (running)
sudo ss -tlnp | grep :80                  # LISTEN 0.0.0.0:80 nginx
```

**4.2. Журналирование.**

```bash
sudo journalctl -u nginx -b > ~/journalctl.out
```

Файл содержит записи `Starting nginx.service`, `syntax is ok`, `test is successful`, `Started nginx.service` (4 строки при первом старте; пересобран после перезапуска - 11 строк).

---

### Раздел 5. Управление доступом

**5.1. DAC + ACL.**

Созданы группы и пользователи:

```bash
sudo groupadd developers
sudo groupadd -g 4444 curators

sudo useradd -u 5501 -m -s /bin/bash user1
sudo useradd -u 5502 -m -s /bin/bash user2
sudo useradd -u 5503 -m -s /bin/bash user3
sudo useradd -m -s /bin/bash curator1
sudo useradd -m -s /bin/bash curator2

sudo usermod -aG developers user1 user2 user3
sudo usermod -aG curators curator1 curator2
```

Создана директория проекта:

```bash
sudo mkdir -p /data/mephi-2026
sudo chown root:developers /data/mephi-2026
sudo chmod 2770 /data/mephi-2026           # SGID + rwx для developers
```

Настроены **default ACL** (для будущих файлов):

```bash
sudo setfacl -d -m u::rwx,g::rwx,o::--- /data/mephi-2026
sudo setfacl -d -m g:developers:rwx /data/mephi-2026
sudo setfacl -d -m g:curators:r-x /data/mephi-2026
```

**Проблема и исправление.** После default ACL попытка `curator1` прочитать файл дала `Permission denied` - default ACL применяется только к **новым** файлам, а сама директория осталась без access-записи для `curators`. Исправлено:

```bash
sudo setfacl -m g:curators:r-x /data/mephi-2026
getfacl /data/mephi-2026
# user::rwx
# group::rwx
# group:curators:r-x
# mask::rwx
# other::---
# default:user::rwx
# default:group::rwx
# default:group:developers:rwx
# default:group:curators:r-x
# default:mask::rwx
# default:other::---
```

**Проверки:**

| Действие | Результат |
|---|---|
| `sudo -u user1 touch /data/mephi-2026/test_from_user1.txt` | создан `-rw-rw----+ user1:developers` (SGID сработал) |
| `sudo -u curator1 ls /data/mephi-2026/` | видит файл |
| `sudo -u curator1 cat /data/mephi-2026/test_from_user1.txt` | читает |
| `sudo -u curator1 touch /data/mephi-2026/test_from_user1.txt` | `Permission denied` ✅ |
| `sudo -u user2 touch /data/mephi-2026/test_from_user2.txt` | создаёт |
| `sudo -u user2 cat /data/mephi-2026/test_from_user1.txt` | читает (developers имеют rw) |
| `sudo getfacl ...test_from_user1.txt` | `group:developers:rwx #effective:rw-`, `group:curators:r-x #effective:r--`, `mask::rw-` |

**5.2. Привилегии tcpdump.**

Проверено исходное состояние:

```bash
ls -l /usr/sbin/tcpdump        # -rwxr-xr-x (SUID отсутствует)
getcap /usr/sbin/tcpdump       # пусто
sudo -u user1 tcpdump -c 1 -i any -nn
# Permission denied: "socket: Operation not permitted"
```

> В РЕД ОС 8 пакет `tcpdump` поставляется **без SUID-бита**, поэтому снимать SUID не требуется. Для запуска обычным пользователем нужны capabilities.

Установлены capabilities:

```bash
sudo setcap cap_net_raw,cap_net_admin+eip /usr/sbin/tcpdump
getcap /usr/sbin/tcpdump
# /usr/sbin/tcpdump cap_net_admin,cap_net_raw=eip

sudo -u user1 timeout 5 tcpdump -c 1 -i any -nn
# listening on any, link-type LINUX_SLL2
# 1 packet captured, 0 packets dropped by kernel
```

**5.3. SELinux + nginx на `/mephi-web`.**

```bash
getenforce                     # Enforcing

# Замена корневой директории nginx (только активная строка)
sudo sed -i 's|^[[:space:]]*root[[:space:]]*/usr/share/nginx/html;|        root /mephi-web;|' /etc/nginx/nginx.conf
grep -n "root" /etc/nginx/nginx.conf
# 41:        root /mephi-web;
# 54:#        root         /usr/share/nginx/html;   ← закомментированная, не тронута

# SELinux-контекст
sudo semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"
sudo restorecon -Rv /mephi-web
# Relabeled /mephi-web ... to system_u:object_r:httpd_sys_content_t:s0
ls -Zd /mephi-web
# system_u:object_r:httpd_sys_content_t:s0 /mephi-web

sudo systemctl restart nginx   # active (running)
curl -I http://localhost/
# HTTP/1.1 403 Forbidden (нет index.html - ожидаемо)
sudo ausearch -m AVC -ts recent
# пусто (нет отказов SELinux)
```

---

### Раздел 6. Аутентификация

**6.1. Ограничение входа для кураторов через локальную консоль.**

`pam_access` в `/etc/pam.d/login` не был подключён. Добавлен:

```bash
sudo sed -i '/^account[[:space:]]\+include[[:space:]]\+system-auth/i account    required     pam_access.so' /etc/pam.d/login
grep -n "account" /etc/pam.d/login
# 4:account    required     pam_nologin.so
# 5:account    required     pam_access.so
# 6:account    include      system-auth
```

Правила в `/etc/security/access.conf`:

```bash
printf '%s\n' '-:curators:LOCAL' '+:ALL:ALL' | sudo tee -a /etc/security/access.conf
tail -5 /etc/security/access.conf
# -:curators:LOCAL
# +:ALL:ALL
```

Группа `curators` не сможет войти с локального терминала, остальные - могут отовсюду (включая SSH, т.к. `PAM_RHOST` для SSH не `LOCAL`).

**6.2. Управление паролями.**

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS\t90/' /etc/login.defs
grep "^PASS_MAX_DAYS" /etc/login.defs       # PASS_MAX_DAYS   90

sudo sed -i 's/^\s*#\s*minlen\s*=.*/minlen = 12/' /etc/security/pwquality.conf
grep -n '^minlen' /etc/security/pwquality.conf   # 11:minlen = 12

sudo chage -M 90 user1
sudo chage -M 90 user2
sudo chage -M 90 user3
sudo chage -M 90 curator1
sudo chage -M 90 curator2
```

Проверка: `sudo chage -l user1` и `sudo chage -l curator1` → `Максимальное количество дней между сменой пароля : 90`.

---

### Раздел 7. Тестирование

**7.1. index.html.**

```bash
sudo bash -c 'echo "Hello from Student: 372770" > /mephi-web/index.html'
sudo restorecon -Rv /mephi-web
ls -lZ /mephi-web/index.html
# -rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 ... index.html
```

**7.2. Проверка доступности.**

```bash
curl http://localhost/
# Hello from Student: 372770

curl http://localhost/ > ~/curl.out
```

---

### Раздел 8. Публикация

Все 14 артефактов собраны в репозитории. Скриншот `mephi-screenshot.png` содержит вывод `curl http://localhost/` с текстом `Hello from Student: 372770`.

---

## Примечания

1. **ВМ и окружение.** РЕД ОС 8.0.3 Standard Server, развёрнута в Proxmox (QEMU/KVM). Работа ведётся по SSH; графическая подсистема не используется.
2. **Сохранность после перезагрузки.** Все настройки сохраняются: `/etc/fstab` (монтирование по `LABEL=MEPHI_WEB`), `/etc/nginx/nginx.conf`, `/etc/pam.d/login`, `/etc/security/access.conf`, `/etc/login.defs`, `/etc/security/pwquality.conf`, а также SELinux-контекст (`semanage fcontext` + `restorecon`), capabilities на `/usr/sbin/tcpdump`.
3. **`tcpdump` установлен через `rpm -ivh`** (по требованию задания), поэтому в `dnf history` установка не отражена. Факт установки подтверждается `rpm -q tcpdump` и `tcpdump --version`.
4. **`/data/mephi-2026`** имеет SELinux-контекст `default_t` - задание для этой директории требует только DAC+ACL, специфичный SELinux-тип не задан. Для `/mephi-web` установлен `httpd_sys_content_t` (требование 5.3).
5. **`vladblednov` не имеет доступа к `/data/mephi-2026`** - это ожидаемое поведение: текущий администратор не входит в группы `developers`/`curators`, для него действует `other::---`. Проверки раздела 5.1 выполнялись от имени `user1`/`user2`/`curator1`.
