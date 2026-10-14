# Сессионный проект по курсу «Безопасность GNU/Linux»

Настройка базовых средств защиты ОС GNU/Linux на виртуальной машине РЕД ОС 8: сеть, пакеты, файловая система, nginx, DAC, capabilities, SELinux и PAM.

**Номер зачётной книжки: 372771.**

## Описание стенда

| Параметр | Значение |
|---|---|
| Дистрибутив | РЕД ОС 8, серверная установка без графической среды |
| Виртуальная машина | VMware |
| Пользователь администратора | `alex`, административные команды выполняются через `sudo` |
| Имя хоста | `mephi-2026.domain.local` |
| Сетевой интерфейс | `ens160`, получение IPv4-адреса по DHCP |
| Адрес ВМ во время проверки | `192.168.198.128` |
| Дополнительный диск и раздел | `/dev/nvme0n2`, `/dev/nvme0n2p1` |
| Файловая система веб-контента | ext4, метка `MEPHI_WEB`, точка монтирования `/mephi-web` |
| Каталог файлов проекта | `/data/mephi-2026` |
| Веб-сервер | nginx, HTTP-порт 80 |

Настройка парольной политики из пункта **6.2 выполнена до создания пользователей из пункта 5.1**. Поэтому новые пользователи получили срок действия пароля 90 дней при создании. Для существующего пользователя `alex` срок установлен отдельно через `chage`.

## 1. Установка дистрибутива и сеть

### Описание

Для подключения `ens160` задано получение IPv4-адреса по DHCP. Имя хоста изменено на `mephi-2026.domain.local`. В проверке связи с `8.8.8.8` получены ответы на все четыре запроса: потери составили 0%.

### Выполнение

```bash
nmcli device status
nmcli connection show
sudo nmcli connection modify ens160 ipv4.method auto
nmcli -f ipv4.method connection show ens160
nmcli -f IP4,DHCP4 device show ens160
nmcli -f connection.autoconnect connection show ens160
sudo hostnamectl set-hostname mephi-2026.domain.local
hostnamectl
ping -c 4 8.8.8.8 > /home/alex/ping.out
```

## 2. Управление программным обеспечением

### Описание

Выполнено обновление пакетов дистрибутива. Через `dnf` установлены `nginx` и `libcap-ng-utils`. Бинарный RPM-пакет `tcpdump` скачан в `/tmp` и установлен командой `rpm`.

Дополнительно установлен `policycoreutils-python-utils`, предоставляющий `semanage`. Установка подтверждена отдельным запросом к базе RPM в [semanage-package.out](semanage-package.out): `policycoreutils-python-utils-3.5-1.red80.noarch`.

### Выполнение

```bash
sudo dnf upgrade -y
sudo dnf install nginx libcap-ng-utils -y
sudo dnf download --destdir /tmp --resolve tcpdump
sudo rpm -ivh /tmp/tcpdump-4.99.5-1.red80.x86_64.rpm
sudo dnf install policycoreutils-python-utils -y
rpm -q policycoreutils-python-utils > /home/alex/semanage-package.out
rpm -qf /usr/sbin/semanage
dnf history > /home/alex/dnf.out
```

## 3. Управление файловыми системами

### Описание

На дополнительном диске `/dev/nvme0n2` создан раздел `/dev/nvme0n2p1`. Раздел отформатирован в ext4 с меткой `MEPHI_WEB` и смонтирован в `/mephi-web`.

Автоматическое монтирование задано в `/etc/fstab` по метке файловой системы:

```fstab
LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2
```

### Выполнение

```bash
lsblk
sudo fdisk -l /dev/nvme0n2
sudo fdisk /dev/nvme0n2
sudo mkfs.ext4 -L MEPHI_WEB /dev/nvme0n2p1
sudo blkid /dev/nvme0n2p1
sudo mkdir /mephi-web
sudo nano /etc/fstab
sudo systemctl daemon-reload
sudo mount /mephi-web
findmnt --mountpoint /mephi-web/
cp /etc/fstab /home/alex/fstab
```

## 4. Управление сервисами

### Описание

Сервис nginx запущен. Автозапуск настроен командой `systemctl enable`. В [journalctl.out](journalctl.out) сохранены сообщения nginx для текущей загрузки: успешный запуск и последующая перезагрузка конфигурации.

### Выполнение

```bash
sudo systemctl start nginx.service
sudo systemctl enable nginx.service
systemctl status nginx.service
journalctl -u nginx.service -b > /home/alex/journalctl.out
```

## 5. Управление доступом

### 5.1. Дискреционное управление доступом — DAC

Созданы разработчики `user1`, `user2`, `user3` с UID 5501, 5502, 5503. Они включены в группу `developers` с GID 4445. Кураторы `curator1` и `curator2` включены в группу `curators` с GID 4444. Всем пяти пользователям установлены пароли.

Каталог `/data/mephi-2026` принадлежит `root:developers` и имеет режим `2770`. Бит setgid обеспечивает наследование группы каталога при создании файлов. ACL и default ACL задают разработчикам чтение и запись, кураторам — чтение файлов и доступ к каталогам, остальным обычным пользователям — отсутствие доступа. Default ACL применяется при создании новых объектов; обычные файлы создаются без права исполнения.

```bash
sudo groupadd -g 4444 curators
sudo groupadd developers
sudo useradd -u 5501 -m -s /bin/bash user1
sudo useradd -u 5502 -m -s /bin/bash user2
sudo useradd -u 5503 -m -s /bin/bash user3
sudo usermod -aG developers user1
sudo usermod -aG developers user2
sudo usermod -aG developers user3
sudo useradd -m -s /bin/bash curator1
sudo useradd -m -s /bin/bash curator2
sudo usermod -aG curators curator1
sudo usermod -aG curators curator2
sudo passwd user1
sudo passwd user2
sudo passwd user3
sudo passwd curator1
sudo passwd curator2
sudo mkdir -p /data/mephi-2026
sudo chown root:developers /data/mephi-2026/
sudo chmod 770 /data/mephi-2026/
sudo chmod g+s /data/mephi-2026/
sudo setfacl -m g:curators:5 /data/mephi-2026/
sudo setfacl -d -m g:curators:5 /data/mephi-2026/
getfacl /data/mephi-2026 > /home/alex/getfacl.out
```

Фактические права, владельцы и наследуемые ACL сохранены в [stat.out](stat.out) и [getfacl.out](getfacl.out).

### 5.2. Привилегии — capabilities

Исполняемому файлу `/usr/sbin/tcpdump` назначена capability `cap_net_raw=eip`, позволяющая выполнять захват пакетов обычному пользователю. Захват пакетов разрешён через файловую capability.

Проверки от имени `user1` и дополнительно `curator2` завершились захватом пяти пакетов на интерфейсе `ens160`; потерь ядром не зафиксировано.

```bash
sudo setcap cap_net_raw=eip /usr/sbin/tcpdump
getcap /usr/sbin/tcpdump > /home/alex/getcap.out
sudo -u user1 tcpdump -nn -c 5 2>&1 | tee /home/alex/tcpdump-user1.out > /dev/null
sudo -u curator2 tcpdump -nn -c 5 2>&1 | tee /home/alex/tcpdump-curator2.out > /dev/null
```

### 5.3. Мандатное управление доступом — SELinux

SELinux работает в режиме `Enforcing`. Для `/mephi-web` и вложенных объектов сохранено правило типа `httpd_sys_content_t`, после чего метки применены через `restorecon`.

```bash
getenforce > /home/alex/getenforce.out
sudo semanage fcontext -a -t httpd_sys_content_t '/mephi-web(/.*)?'
sudo restorecon -Rv /mephi-web/
ls -Zd /mephi-web/
sudo semanage fcontext -l -C > /home/alex/semanage-fcontext.out
```

В HTTP-блоке `server` файла `/etc/nginx/nginx.conf` установлен корень веб-контента:

```nginx
root /mephi-web;
```

После изменения конфигурация проверена и применена через reload:

```bash
sudo nginx -T
sudo systemctl reload nginx.service
stat /data/mephi-2026/ /mephi-web/ > /home/alex/stat.out
cp /etc/nginx/nginx.conf /home/alex/nginx.conf
```

## 6. Аутентификация

### 6.1. Ограничение локального входа

Ограничение настроено для консольного входа через PAM-сервис `login`. В `/etc/pam.d/login` добавлена строка:

```text
account required pam_access.so accessfile=/etc/security/login-access.conf
```

Отдельный файл правил `/etc/security/login-access.conf` содержит:

```text
-:(curators):ALL
+:ALL:ALL
```

Правила запрещают консольный вход членам группы `curators`. Остальные пользователи проходят обычные проверки аутентификации. Оболочки кураторов сохранены как `/bin/bash`; настройка не добавлялась в PAM-сервисы `sshd`, `su` или `sudo`.

Общий файл `system-auth`, управляемый `authselect`, вручную не изменялся. Конфигурации сохранены в [login](login) и [login-access.conf](login-access.conf).

### 6.2. Управление паролями

В `/etc/security/pwquality.conf` включены минимальная длина 12 символов и обязательная проверка качества при установке пароля через root:

```ini
minlen = 12
enforce_for_root
```

В `/etc/login.defs` задан максимальный возраст пароля для создаваемых пользователей:

```text
PASS_MAX_DAYS 90
```

Для уже существовавшего `alex` пароль изменён и срок установлен отдельно:

```bash
passwd
sudo chage -M 90 alex
sudo chage -l alex
```

В собранной копии `shadow` для `alex`, `user1`, `user2`, `user3`, `curator1`, `curator2` подтверждены максимальный возраст пароля 90 дней и предупреждение за 7 дней. Последняя смена паролей — **14.10.2026**, дата истечения при таком сроке — **12.01.2027**.

## 7. Создание и проверка веб-страницы

В `/mephi-web/index.html` создана HTML-страница с заголовком:

```html
<h1>Hello from Student: 372771</h1>
```

Веб-сервер проверен HTTP-запросом на этой же виртуальной машине:

```bash
ls -lZ /mephi-web/index.html
curl -sS http://localhost/ > /home/alex/curl.out
cat /home/alex/curl.out
history > /home/alex/history.out
```

Полученный HTML сохранён в [curl.out](curl.out). Скриншот показывает команду `curl` и ответ сервера с номером зачётной книжки.

# Артефакты выполнения работы

Обязательные системные файлы представлены под собственными именами, без префикса `/etc/`. История сохранена из оболочки пользователя `alex`; административные действия отражены командами с `sudo`.

| Обязательный файл | Содержание и назначение |
|---|---|
| [mephi-screenshot.png](mephi-screenshot.png) | Скриншот HTTP-проверки с текстом `Hello from Student: 372771` |
| [history.out](history.out) | История выполненных команд |
| [ping.out](ping.out) | Проверка связи с `8.8.8.8`, четыре ответа, потери 0% |
| [dnf.out](dnf.out) | История обновления и установки пакетов |
| [stat.out](stat.out) | Владельцы, права и SELinux-контексты обоих каталогов |
| [journalctl.out](journalctl.out) | Сообщения nginx для текущей загрузки системы |
| [getcap.out](getcap.out) | Capability `cap_net_raw=eip` у tcpdump |
| [getenforce.out](getenforce.out) | Режим SELinux `Enforcing` |
| [curl.out](curl.out) | HTML-ответ nginx с номером зачётной книжки |
| [fstab](fstab) | Автоматическое монтирование по метке `MEPHI_WEB` |
| [passwd](passwd) | Пользователи, UID, домашние каталоги и оболочки |
| [shadow](shadow) | Собранная копия `/etc/shadow`: параметры паролей и сроков действия; также содержит парольные хэши |
| [group](group) | Группы, GID и членство пользователей |
| [pwquality.conf](pwquality.conf) | Минимальная длина пароля и `enforce_for_root` |

Пароли и хэши не воспроизводятся в тексте README. Для локальной копии `shadow` выполнены назначение владельца `alex:alex` и установка прав `600`. Перед размещением копии в публичном репозитории следует скрыть только второе поле с парольными хэшами, сохранив имена пользователей и все поля сроков действия паролей. Изменение публикационной копии необходимо явно указать при сдаче.

| Дополнительный файл | Содержание и назначение |
|---|---|
| [getfacl.out](getfacl.out) | Access ACL и default ACL каталога проекта |
| [tcpdump-user1.out](tcpdump-user1.out) | Успешный захват пяти пакетов от имени `user1` |
| [tcpdump-curator2.out](tcpdump-curator2.out) | Дополнительная проверка tcpdump от имени `curator2` |
| [semanage-package.out](semanage-package.out) | Отдельное подтверждение установки `policycoreutils-python-utils` |
| [semanage-fcontext.out](semanage-fcontext.out) | Сохранённое локальное правило SELinux для веб-каталога |
| [nginx.conf](nginx.conf) | Конфигурация nginx с корнем `/mephi-web` |
| [login](login) | Подключение `pam_access` для консольного входа |
| [login-access.conf](login-access.conf) | Правила доступа для группы кураторов |
| [login.defs](login.defs) | Максимальный срок пароля 90 дней для новых пользователей |

Комплект предназначен для размещения в публичном репозитории **`mephi-session-project-2026`** и сдачи ссылки через форму проекта.
