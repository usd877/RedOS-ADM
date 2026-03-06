# 📚 Документация по развёртыванию РЕД АДМ (Промышленная редакция)

## **Эпический гайд от первой установки до работающего домена**
*Автор: Человек, который прошёл через все круги ада и победил*

---

## 📋 Содержание
1. [Введение и требования](#1-введение-и-требования)
2. [Этап 1: Подготовка сервера и установка РЕД АДМ](#2-этап-1-подготовка-сервера-и-установка-ред-адм)
3. [Этап 2: Настройка контроллера домена (reddc)](#3-этап-2-настройка-контроллера-домена-reddc)
4. [Этап 3: Настройка DNS-сервера (критически важно!)](#4-этап-3-настройка-dns-сервера-критически-важно)
5. [Этап 4: Подготовка клиента к вводу в домен](#5-этап-4-подготовка-клиента-к-вводу-в-домен)
6. [Этап 5: Автоматизация — финальные скрипты](#6-этап-5-автоматизация--финальные-скрипты)
7. [Этап 6: Решение проблем и грабли](#7-этап-6-решение-проблем-и-грабли-на-которые-мы-наступали)
8. [Приложение: Финальные конфиги](#8-приложение-финальные-конфиги)
9. [Заключение](#9-заключение)

---

## 1. 📌 Введение и требования

### Что мы настраивали:
- **Сервер:** РЕД ОС 8, РЕД АДМ (промышленная редакция)
- **Домен:** `<ВАШ_ДОМЕН>` (например, `COMPANY.LOC`)
- **Сеть:** Два интерфейса — внутренний и внешний (настройте под свою сеть)
- **Клиенты:** РЕД ОС 8

### Что должно быть готово:
- Чистая установка РЕД ОС 8
- Настроенные сетевые интерфейсы
- Доступ к репозиториям (или RPM-пакетам на флешке)
- Лицензионный ключ РЕД АДМ

---

## 2. 🚀 Этап 1: Подготовка сервера и установка РЕД АДМ

### 2.1. Настройка DNS на сервере (самое важное!)
```bash
# Чистим старый resolv.conf
rm -f /etc/resolv.conf
echo "nameserver <DNS_1>" > /etc/resolv.conf
echo "nameserver <DNS_2>" >> /etc/resolv.conf
chattr +i /etc/resolv.conf
```

### 2.2. Установка пакетов
```bash
# Установка из репозитория
dnf install -y redadm redadm-reddc-manager
```

### 2.3. Первоначальная настройка в веб-интерфейсе
1. Открыть `https://<IP_АДРЕС_СЕРВЕРА>`
2. Принять лицензию, ввести ключ
3. На шаге выбора домена выбрать **«Развернуть новый домен»**
4. Параметры домена:
   - Имя домена: `<ВАШ_ДОМЕН>`
   - Пароль администратора: `<ВАШ_ПАРОЛЬ>`
   - Имя контроллера: `<ИМЯ_КОНТРОЛЛЕРА>`
   - Служебный пользователь: `administrator` / `<ВАШ_ПАРОЛЬ>`

<img width="906" height="843" alt="2026-03-06 11_21_01-РЕД АДМ - Вход_  InPrivate _ Microsoft​ Edge" src="https://github.com/user-attachments/assets/9976b125-c65e-4f32-8070-fc8b4fc50d33" />

<img width="1196" height="838" alt="2026-02-25 14_59_07-РЕД АДМ - Первоначальная настройка_  InPrivate _ Microsoft​ Edge" src="https://github.com/user-attachments/assets/ddbd94ce-9ad0-449c-bebb-acd2a256de9c" />

---

## 3. ⚙️ Этап 2: Настройка контроллера домена (reddc)

### 3.1. Проверка служб
```bash
systemctl status reddc
ss -tlnp | grep 389   # LDAP должен слушаться
```

### 3.2. Если reddc не запускается
```bash
# Проверка конфига
cat /opt/reddc/etc/smb.conf | grep interfaces

# Если нет — добавляем
echo "interfaces = lo <СЕТЕВОЙ_ИНТЕРФЕЙС> <IP_АДРЕС_СЕРВЕРА>/<МАСКА>" >> /opt/reddc/etc/smb.conf
systemctl restart reddc
```

---

## 4. 🌐 Этап 3: Настройка DNS-сервера (критически важно!)

### 4.1. Установка и настройка BIND
```bash
dnf install -y bind
```

### 4.2. Конфигурация зоны `/etc/named.conf`
```bind
zone "<ваш_домен>" IN {
    type master;
    file "<ваш_домен>.zone";
    allow-update { any; };
};
```

### 4.3. Файл зоны `/var/named/<ваш_домен>.zone`
```bind
$TTL 86400
@   IN  SOA     <имя_контроллера>.<ваш_домен>. admin.<ваш_домен>. (
                2025030301  ; serial
                3600        ; refresh
                1800        ; retry
                604800      ; expire
                86400       ; minimum
)

@       IN  NS      <имя_контроллера>.<ваш_домен>.
@       IN  A       <IP_АДРЕС_СЕРВЕРА>
<имя_контроллера>  IN  A   <IP_АДРЕС_СЕРВЕРА>

; Критически важные SRV-записи!
_ldap._tcp.<ваш_домен>.     86400 IN SRV 0 100 389 <имя_контроллера>.<ваш_домен>.
_kerberos._tcp.<ваш_домен>. 86400 IN SRV 0 100 88  <имя_контроллера>.<ваш_домен>.
```

### 4.4. Запуск DNS
```bash
systemctl restart named
systemctl enable named
```

---

## 5. 💻 Этап 4: Подготовка клиента к вводу в домен

### 5.1. Критический шаг — проверка оболочки входа!
> ⚠️ **ВАЖНО:** На одном из ПК проблема оказалась в том, что стояла "левая" оболочка (VNC). Пользователи не заходили, пока не была установлена нормальная оболочка (GDM/LightDM).

```bash
# Установка нормальной оболочки (если есть сомнения)
dnf groupinstall -y "Server with GUI"
systemctl set-default graphical.target
```

### 5.2. Настройка DNS на клиенте
```bash
# Фиксируем resolv.conf
rm -f /etc/resolv.conf
cat > /etc/resolv.conf << 'EOF'
nameserver <IP_DNS_1>
nameserver <IP_DNS_2>
EOF
chattr +i /etc/resolv.conf
```

### 5.3. Установка пакетов
```bash
dnf install -y realmd sssd oddjob-mkhomedir adcli krb5-workstation
```

### 5.4. Настройка Kerberos (`/etc/krb5.conf`)
```ini
includedir /etc/krb5.conf.d/

[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
default_realm = <ВАШ_ДОМЕН>
dns_lookup_realm = false
dns_lookup_kdc = true
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
udp_preference_limit = 0
default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96

[realms]
<ВАШ_ДОМЕН> = {
    kdc = <IP_АДРЕС_СЕРВЕРА>
    kdc = <IP_АДРЕС_СЕРВЕРА_2>
    admin_server = <IP_АДРЕС_СЕРВЕРА>
}

[domain_realm]
.<ваш_домен> = <ВАШ_ДОМЕН>
<ваш_домен> = <ВАШ_ДОМЕН>
```

### 5.5. Ввод в домен
```bash
realm join -v -U administrator <ВАШ_ДОМЕН>
# Пароль: <ВАШ_ПАРОЛЬ>
```

### 5.6. Разрешение входа всем
```bash
realm permit --all
```

### 5.7. Настройка SSSD (`/etc/sssd/sssd.conf`)
```ini
[sssd]
domains = <ВАШ_ДОМЕН>
config_file_version = 2
services = nss, pam

[domain/<ВАШ_ДОМЕН>]
ad_domain = <ВАШ_ДОМЕН>
krb5_realm = <ВАШ_ДОМЕН>
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
auth_provider = ad
access_provider = ad
chpass_provider = ad
ldap_id_mapping = True
use_fully_qualified_names = True
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
enumerate = false
```
```bash
chmod 600 /etc/sssd/sssd.conf
systemctl restart sssd
systemctl enable sssd
```

### 5.8. Автоматическое создание домашних папок
```bash
systemctl enable --now oddjobd
```

---

## 5.9. 🟢 АЛЬТЕРНАТИВНЫЙ ПУТЬ: Быстрый ввод в домен (если настроен DHCP)

Если на сервере уже настроен DHCP-сервер, который раздаёт клиентам правильные параметры:
- **DNS-сервер:** `<IP_АДРЕС_СЕРВЕРА>`
- **Домен:** `<ВАШ_ДОМЕН>`
- **Шлюз:** `<IP_АДРЕС_СЕРВЕРА>`

то процесс ввода клиента в домен сокращается до двух шагов.  
Этот способ экономит часы ручной правки конфигов и исключает ошибки.

---

### 📸 **Как должны выглядеть настройки DHCP**

Для работы автоматического входа в домен DHCP-сервер должен быть настроен следующим образом:

| Параметр | Значение | Код опции |
|----------|----------|-----------|
| Домен | `<ВАШ_ДОМЕН>` | 15 |
| DNS-серверы | `<IP_АДРЕС_СЕРВЕРА>` | 6 |
| Шлюз | `<IP_АДРЕС_СЕРВЕРА>` | 3 |

Эти настройки гарантируют, что клиент получит:
- Правильный DNS для поиска контроллера домена
- Домен для автоматической подстановки
- Маршрут к серверу

---

### 🚀 **Процедура ввода клиента в домен**

#### Шаг 1. Подключить клиент к сети
Клиент автоматически получит по DHCP:
- IP-адрес (например, `192.168.0.101`)
- Правильный DNS-сервер
- Домен для поиска

Проверить можно командой:
```bash
cat /etc/resolv.conf
# Должен быть: nameserver <IP_АДРЕС_СЕРВЕРА>
```

#### Шаг 2. Запустить утилиту ввода в домен
```bash
join-to-domain.sh
```
В мастере выбрать:
- **Тип домена:** Домен Windows/SAMBA
- **Имя домена:** `<ВАШ_ДОМЕН>`
- **Имя компьютера:** придумать уникальное (например, `client-01`)
- **Имя администратора:** `administrator`
- **Пароль:** `<ВАШ_ПАРОЛЬ>`

#### Шаг 3. Перезагрузить компьютер
```bash
reboot
```

---

### ✅ **Результат**
После перезагрузки любой доменный пользователь сможет войти в систему.  
Никаких ручных правок `/etc/krb5.conf`, `/etc/sssd/sssd.conf` и других файлов не требуется.

---

### ⚠️ **Важное замечание**
Этот способ работает **только при правильно настроенном DHCP-сервере**.  
Если ваш DHCP не раздаёт параметры из таблицы выше, вернитесь к ручному пути (разделы 5.1–5.8) или настройте DHCP по инструкции.

---

## 6. 🤖 Этап 5: Автоматизация — финальные скрипты

### 6.1. Скрипт v3 (базовый, стабильный)
**Файл:** `setup-client-v3.sh`

```bash
#!/bin/bash
# Скрипт автоматической настройки клиента для домена <ВАШ_ДОМЕН>
# Версия: 3.0 (стабильная)

set -e

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

print_step() { echo -e "${GREEN}[INFO]${NC} $1"; }
print_error() { echo -e "${RED}[ERROR]${NC} $1"; }

if [[ $EUID -ne 0 ]]; then
    print_error "Запусти от root"
    exit 1
fi

print_step "Начинаем настройку клиента для домена <ВАШ_ДОМЕН>"

INTERFACE=$(ip -o -4 route show to default | awk '{print $5}' | head -1)
print_step "Используем интерфейс: $INTERFACE"

print_step "1. Настраиваем DNS и фиксируем..."
rm -f /etc/resolv.conf
cat > /etc/resolv.conf << 'EOF'
nameserver <IP_DNS_1>
nameserver <IP_DNS_2>
EOF
chattr +i /etc/resolv.conf

nmcli con mod "$INTERFACE" ipv4.ignore-auto-dns yes
nmcli con mod "$INTERFACE" ipv4.dns ""
nmcli con down "$INTERFACE"
nmcli con up "$INTERFACE"
print_step "DNS настроен и защищён"

print_step "2. Устанавливаем пакеты..."
dnf install -y realmd sssd oddjob-mkhomedir adcli krb5-workstation

print_step "3. Настраиваем Kerberos..."
cat > /etc/krb5.conf << 'EOF'
includedir /etc/krb5.conf.d/

[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
default_realm = <ВАШ_ДОМЕН>
dns_lookup_realm = false
dns_lookup_kdc = true
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
udp_preference_limit = 0

[realms]
<ВАШ_ДОМЕН> = {
    kdc = <IP_АДРЕС_СЕРВЕРА>
    kdc = <IP_АДРЕС_СЕРВЕРА_2>
    admin_server = <IP_АДРЕС_СЕРВЕРА>
}

[domain_realm]
.<ваш_домен> = <ВАШ_ДОМЕН>
<ваш_домен> = <ВАШ_ДОМЕН>
EOF
print_step "Kerberos настроен"

print_step "4. Настраиваем REALMD..."
cat > /etc/realmd.conf << 'EOF'
[users]
default-home = /home/%u@%d
default-shell = /bin/bash

[active-directory]
default-client = sssd
os-name = РЕД ОС
os-version = 8
EOF

print_step "5. Вводим компьютер в домен..."
realm join -v -U administrator <ВАШ_ДОМЕН> <<< "<ВАШ_ПАРОЛЬ>"
print_step "Компьютер успешно введён в домен"

print_step "6. Разрешаем вход всем пользователям..."
realm permit --all

print_step "7. Настраиваем SSSD..."
cat > /etc/sssd/sssd.conf << 'EOF'
[sssd]
domains = <ВАШ_ДОМЕН>
config_file_version = 2
services = nss, pam

[domain/<ВАШ_ДОМЕН>]
ad_domain = <ВАШ_ДОМЕН>
krb5_realm = <ВАШ_ДОМЕН>
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
auth_provider = ad
access_provider = ad
chpass_provider = ad
ldap_id_mapping = True
use_fully_qualified_names = True
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
enumerate = false
EOF
chmod 600 /etc/sssd/sssd.conf
systemctl restart sssd
systemctl enable sssd

print_step "8. Настраиваем oddjobd..."
systemctl enable --now oddjobd

print_step "9. Проверяем работу..."
sleep 5
if getent passwd administrator@<ВАШ_ДОМЕН> | head -1 > /dev/null; then
    print_step "✅ Пользователь виден. Всё работает!"
else
    print_error "❌ Пользователь не виден. Проверь логи: journalctl -u sssd"
fi

print_step "✅ Настройка завершена! Рекомендуется перезагрузить компьютер."
```

### 6.2. Скрипт v4 (с фиксом шифрования для проблемных клиентов)
**Файл:** `setup-client-v4.sh`

Отличается от v3 только добавленными строками в секцию `[libdefaults]` файла `/etc/krb5.conf`:

```ini
default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
```

---

## 7. ⚠️ Этап 6: Решение проблем и грабли (на которые мы наступали)

### ❌ Проблема 1: Пользователь не заходит, хотя скрипт отработал
**Причина:** Неправильная оболочка входа (VNC вместо GDM/LightDM)  
**Решение:** Установить нормальную графическую оболочку
```bash
dnf groupinstall -y "Server with GUI"
systemctl set-default graphical.target
reboot
```

### ❌ Проблема 2: Ошибка "KDC reply did not match expectation"
**Причина:** Рассинхронизация времени или неправильные типы шифрования  
**Решение:**
```bash
ntpdate <IP_АДРЕС_СЕРВЕРА>
# И добавить enctypes в krb5.conf
```

### ❌ Проблема 3: DNS сбрасывается после перезагрузки
**Причина:** Файл `/etc/resolv.conf` не зафиксирован  
**Решение:**
```bash
chattr +i /etc/resolv.conf
```

### ❌ Проблема 4: Компьютер в домене, но `realm join` говорит "область не найдена"
**Причина:** Нет SRV-записей в DNS или клиент смотрит не туда  
**Решение:** Проверить DNS и добавить SRV-записи на сервере

### ❌ Проблема 5: Пользователь виден, но вход в графический интерфейс откатывает
**Причина:** Не создаётся домашняя папка  
**Решение:**
```bash
mkdir -p /home/<пользователь>
chown <пользователь>:<пользователь> /home/<пользователь>
```

---

## 8. 📁 Приложение: Финальные конфиги

### ✅ `/etc/krb5.conf` (рабочий)
```ini
includedir /etc/krb5.conf.d/

[logging]
default = FILE:/var/log/krb5libs.log
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
default_realm = <ВАШ_ДОМЕН>
dns_lookup_realm = false
dns_lookup_kdc = true
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
udp_preference_limit = 0
default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96

[realms]
<ВАШ_ДОМЕН> = {
    kdc = <IP_АДРЕС_СЕРВЕРА>
    kdc = <IP_АДРЕС_СЕРВЕРА_2>
    admin_server = <IP_АДРЕС_СЕРВЕРА>
}

[domain_realm]
.<ваш_домен> = <ВАШ_ДОМЕН>
<ваш_домен> = <ВАШ_ДОМЕН>
```

### ✅ `/etc/sssd/sssd.conf` (рабочий)
```ini
[sssd]
domains = <ВАШ_ДОМЕН>
config_file_version = 2
services = nss, pam

[domain/<ВАШ_ДОМЕН>]
ad_domain = <ВАШ_ДОМЕН>
krb5_realm = <ВАШ_ДОМЕН>
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
auth_provider = ad
access_provider = ad
chpass_provider = ad
ldap_id_mapping = True
use_fully_qualified_names = True
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
enumerate = false
```

### 🏆 Результат: работающий контроллер домена

После выполнения всех шагов вы должны увидеть работающий контроллер:

<img width="1711" height="957" alt="2026-03-06 11_19_45-РЕД АДМ - Компьютеры_  InPrivate _ Microsoft​ Edge" src="https://github.com/user-attachments/assets/f0e7f1dd-86dd-4e08-93da-ff5bea97cb7b" />

<img width="1591" height="515" alt="2026-03-06 11_20_34-РЕД АДМ - Пользователи_  InPrivate _ Microsoft​ Edge" src="https://github.com/user-attachments/assets/b3ff15e0-f16b-4f7a-a97c-d356c0319058" />

<img width="1623" height="934" alt="2026-03-06 12_02_13-РЕД АДМ - Управление доменом _ Конкретный КД_  InPrivate _ Microsoft​ Edge" src="https://github.com/user-attachments/assets/11462b4b-034e-4c96-b7a6-0e1f4bb238c7" />

---

## 9. 🏁 Заключение

Прошёл путь от установки чистой ОС до полностью работающего домена с автоматической настройкой клиентов. Этот гайд — результат реального опыта, крови и пота.

> 💡 **Совет:** Если ты читаешь это и хочешь повторить — используй скрипты, не танцуй с бубном. Они уже всё делают за тебя.

---

**Автор:** Человек, который доказал, что невозможное — возможно.  
**Дата:** Март 2026  
**Лицензия:** Бери и пользуйся, но помни — грабли уже собраны. 🪄

---

> 📝 **Changelog**
> - v1.0 — Первая версия гайда
> - v1.1 — Добавлены скрипты автоматизации
> - v1.2 — Исправлены проблемы с шифрованием в krb5.conf
> - v1.3 — Добавлен раздел с частыми проблемами
> - v1.4 — Добавлен альтернативный путь (быстрый ввод через DHCP)
