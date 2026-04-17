# 📘 Сессионный проект по курсу «Операционные системы семейства Unix» (МИФИ 2026)

**Сиудент:** Амелякин Борис (зачётная книжка ДН001)

**Среда:** Fedora 43 (minimal, без GUI), VirtualBox

---

# 🔷 1. Настройка сети

## Выполненные действия

* Определён сетевой интерфейс:

```bash
nmcli device status
```

* Настроен статический IP:

```bash
nmcli con mod enp0s3 ipv4.addresses 192.168.0.100/24 \
ipv4.gateway 192.168.0.1 \
ipv4.dns 8.8.8.8 \
ipv4.method manual
```

* Применены настройки:

```bash
nmcli con up enp0s3
```

* Установлено имя хоста:

```bash
hostnamectl set-hostname mephi-2026.domain.local
```

---

## Проверка

```bash
ip addr show
ip route
ping -c 3 8.8.8.8
```

---

## Результаты сохранены:

```bash
ping -c 4 8.8.8.8 > /tmp/network_check.txt
ping -c 4 192.168.0.1 >> /tmp/network_check.txt
```

---

## ⚠️ Проблемы

* Изначально был указан неверный диапазон IP (192.168.1.x)
* Реальная сеть оказалась 192.168.0.x → отсутствовала связность
* Решение: синхронизация подсети с хостом

---

# 🔷 2. Управление программным обеспечением

## Установка пакетов

```bash
sudo dnf install nginx tcpdump libcap-ng-utils -y
```

## Проверка

```bash
nginx -v
tcpdump --version
capsh --print
```

---

## Работа с RPM

```bash
cd /tmp
dnf download tcpdump
sudo dnf remove tcpdump
sudo rpm -ivh tcpdump-4.99.6-2.fc43.x86_64.rpm
```

---

## ⚠️ Проблемы

* Конфликт версий при установке через rpm
* Причина: пакет уже установлен через dnf
* Решение: удалить и установить вручную

---

# 🔷 3. Файловые системы и сервисы

## Работа с диском

```bash
lsblk
sudo fdisk /dev/sdb
sudo mkfs.ext4 -L MEPHI_DATA /dev/sdb1
```

## Монтирование

```bash
sudo mkdir -p /data/mephi-web
sudo vi /etc/fstab
```

Добавлено:

```
LABEL=MEPHI_DATA /data/mephi-web ext4 defaults 0 0
```

```bash
sudo mount -a
df -h
```

---

## Сервис nginx

```bash
sudo systemctl enable --now nginx
```

## Логи

```bash
sudo journalctl -u nginx --since "5 minutes ago" > /tmp/nginx_recent_logs.txt
```

---

## ⚠️ Проблемы

* Отсутствие второго диска → решено через VirtualBox
* Риск ошибки в fstab (критично для загрузки)

---

# 🔷 4. Управление доступом

## Пользователи и группы

```bash
sudo groupadd mephi-devs
sudo useradd -m -g mephi-devs mephi-admin
echo "mephi-admin:P@ssw0rd2026" | sudo chpasswd
```

## Права

```bash
sudo chown mephi-admin:mephi-devs /data/mephi-web
sudo chmod 2775 /data/mephi-web
```

---

## SELinux

```bash
getenforce
sudo semanage fcontext -a -t httpd_sys_content_t "/data/mephi-web(/.*)?"
sudo restorecon -Rv /data/mephi-web
```

Проверка:

```bash
ls -Z /data/mephi-web
```

---

## Capabilities

```bash
sudo chmod u-s /usr/sbin/tcpdump
sudo setcap "cap_net_raw,cap_net_admin+ep" /usr/sbin/tcpdump
```

Проверка:

```bash
sudo -u mephi-admin tcpdump --help
```

---

## ⚠️ Проблемы

* Ошибка `Invalid argument` при setcap
* Причина: лишний пробел
* Решение: корректный синтаксис без пробелов

---

# 🔷 5. Аутентификация и безопасность

## Блокировка root

```bash
echo "root" | sudo tee /etc/ssh/denied_users
sudo vi /etc/pam.d/postlogin
```

Добавлено:

```
auth required pam_listfile.so item=user sense=deny file=/etc/ssh/denied_users onerr=succeed
```

---

## SSH

```bash
sudo sed -i 's/#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

# 🔷 6. Web-сервер и Firewall

## Firewall

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## Страница

```bash
sudo -u mephi-admin bash -c 'echo "Hello from Student: ДН001" > /data/mephi-web/index.html'
```

---

## Конфигурация nginx

```bash
sudo vi /etc/nginx/conf.d/mephi.conf
```

```nginx
server {
    listen 80 default_server;
    server_name _;
    root /data/mephi-web;

    location / {
        index index.html;
    }
}
```

---

## Перезапуск

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Проверка

```bash
curl http://localhost
curl http://192.168.0.100
```

---

## ⚠️ Проблемы

* Показывалась стандартная страница nginx
* Причина: конфликт конфигураций
* Решение:

  * добавить `default_server`
  * правка nginx.conf

---

# 🔷 7. Сбор артефактов

Сохранены результаты:

```bash
history > final_work/project_history.txt
cp /etc/fstab fstab.txt
getenforce > selinux_status.txt
getcap /usr/sbin/tcpdump > tcpdump_capabilities.txt
stat /data/mephi-web > permissions.txt
id mephi-admin > users_groups.txt
curl -s http://192.168.0.100 > curl_output.txt
```

---

# 🔷 Итог

## Выполнено:

* Настроена сеть (статический IP)
* Установлены пакеты
* Подключён и настроен диск
* Настроены права доступа (DAC + SELinux)
* Реализованы capabilities
* Ограничен доступ root
* Развёрнут web-сервер
* Открыт firewall
* Проверена доступность сервиса

---

## 📊 Общая оценка выполнения

Работа выполнена полностью, включая:

* устранение сетевых проблем
* исправление конфликтов пакетов
* настройку SELinux и nginx

---

## 💡 Вывод

Система приведена к состоянию:

* **устойчивой (fstab, systemd)**
* **безопасной (SELinux, PAM, firewall)**
* **готовой к эксплуатации (nginx)**

---
