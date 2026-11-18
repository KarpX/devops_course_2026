
```markdown
# Конфигурация виртуальной машины devops-vm

Документ содержит исчерпывающее описание конфигурации целевого сервера и служит эталоном для ручного восстановления системы со снимка `01-clean-install`, а также основой для разработки ролей Ansible.

---

## 1. Параметры машины

- **Гипервизор:** Oracle VirtualBox 7.x
- **Операционная система:** Ubuntu Server 24.04 LTS (64-bit, без GUI)
- **Имя узла (Hostname):** `devops-vm`
- **Полное доменное имя (FQDN):** `devops-vm.devops.local`
- **Количество ядер CPU:** 2 vCPU
- **Объём оперативной памяти (RAM):** 2048 МБ (2 ГБ)
- **Объём накопителя:** 25 ГБ (динамически расширяемый VDI-диск)

---

## 2. Сетевые интерфейсы

| Интерфейс | Тип адаптера VirtualBox | IP-адрес / CIDR | Назначение |
| :--- | :--- | :--- | :--- |
| `enp0s3` | NAT | `10.0.2.15/24` (DHCP) | Выход гостевой ОС в сеть Интернет, загрузка пакетов и обновлений |
| `enp0s8` | Виртуальный адаптер хоста (Host-only) | `192.168.56.101/24` (DHCP/Static) | Прямой доступ к гостевой ОС с хостовой машины без NAT |

- **Локальное разрешение имён на хосте (`/etc/hosts` или `C:\Windows\System32\drivers\etc\hosts`):**
  ```text
  192.168.56.101 devops.local
  ```

---

## 3. Правило проброса портов

Настроено на Адаптере 1 (NAT) для обеспечения SSH-доступа через интерфейс обратной петли хоста:

| Название | Протокол | IP хоста | Порт хоста | IP гостя | Порт гостя | Примечание |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `ssh` | TCP | `127.0.0.1` | `2222` | *(пусто)* | `2222` | Изменён с 22 на 2222 в ПР № 5 |

---

## 4. Учётные записи

| Пользователь | Группы | Способ аутентификации | Назначение |
| :--- | :--- | :--- | :--- |
| `student` | `student`, `sudo` | Пароль (первичная установка) | Первичная инициализация системы |
| `devops` | `devops`, `sudo` | Открытый ключ Ed25519 | Основная учётная запись администрирования и автоматизации (деплоя) |
| `root` | `root` | Локальный вход (SSH-вход заблокирован) | Суперпользователь (UID 0) |
| `www-data` | `www-data` | Без оболочки входа (`/usr/sbin/nologin`) | Непривилегированный пользователь процессов Nginx |

### Настройка аутентификации по ключу:
- Путь к открытому ключу: `/home/devops/.ssh/authorized_keys`
- Права на каталог: `chmod 700 /home/devops/.ssh`
- Права на файл ключей: `chmod 600 /home/devops/.ssh/authorized_keys`
- Владелец: `devops:devops`

---

## 5. Служба SSH

- **Служба:** `ssh.service` (OpenSSH Server)
- **Слушающий порт:** `2222`
- **Резервная копия эталонной конфигурации:** `/etc/ssh/sshd_config.backup`
- **Файл кастомной конфигурации:** `/etc/ssh/sshd_config.d/99-hardening.conf`

### Директивы усиления безопасности (Hardening):
```text
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers devops
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

### Особенности Ubuntu 24.04 (socket activation):
Для применения нестандартного порта служба сокетов отключена в пользу стандартного сервиса:
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
```
*Примечание:* Строка `PasswordAuthentication yes` в файле `50-cloud-init.conf` (при наличии) закомментирована.

---

## 6. Правила межсетевого экрана (UFW)

- **Статус:** Активен (`Status: active`)
- **Уровень логирования:** `medium` (`/var/log/ufw.log`)
- **Политики по умолчанию:**
  - Входящий трафик: `deny` (`incoming (deny)`)
  - Исходящий трафик: `allow` (`outgoing (allow)`)
  - Маршрутизируемый трафик: `disabled`

### Список правил:
| Порт / Протокол | Действие | Источник | Комментарий |
| :--- | :--- | :--- | :--- |
| `2222/tcp` | **LIMIT** | Любой | SSH rate-limited (не более 6 коннектов за 30 сек) |
| `80/tcp` | **ALLOW** | Любой | HTTP (перенаправление на HTTPS) |
| `443/tcp` | **ALLOW** | Любой | HTTPS (защищённый веб-трафик) |

---

## 7. Снимки состояния (Snapshots)

1. **`01-clean-install`** — Базовая установка Ubuntu Server 24.04 LTS, сконфигурированы сетевые адаптеры NAT и Host-only, создан пользователь `student`, система обновлена (`apt upgrade`). Точка полного отката.
2. **`02-keys-configured`** — Создан пользователь `devops`, добавлен в группу `sudo`, настроена аутентификация по ключу Ed25519 с хоста без ввода пароля.
3. **`03-ssh-hardened`** — Применён hardening службы SSH (порт 2222, отключён парольный вход и root), настроен и активирован межсетевой экран UFW (LIMIT 2222/tcp).
4. **`04-nginx-https`** — Установлен веб-сервер Nginx, выпущен самоподписанный TLS-сертификат с SAN, настроен виртуальный хост `devops.local` с HTTPS и редиректом c HTTP, протестирован автоматический деплой через `rsync`.

---

## 8. Веб-сервер

- **Устанавливаемый пакет:** `nginx` (`sudo apt install -y nginx`).
- **Главный процесс (master):** запускается от `root` (открывает порты 80 и 443, считывает закрытый ключ).
- **Рабочие процессы (worker):** функционируют от пользователя `www-data`.

### 8.1 Каталог публикуемого ресурса
- **Путь:** `/var/www/devops-site`
- **Владелец и группа:** `devops:devops`
- **Модель прав («владелец пишет, веб-сервер читает»):**
  - Для каталогов: `755` (`drwxr-xr-x`) — владелец имеет полный доступ, остальные (включая `www-data`) — вход и чтение содержимого.
  - Для файлов: `644` (`-rw-r--r--`) — владелец имеет право записи, остальные — только чтение.
  - Файлы с правами записи для всех (`o+w` / `666` / `777`) категорически запрещены.

### 8.2 TLS-сертификат и закрытый ключ
- **Закрытый ключ:** `/etc/ssl/private/devops.key`
  - Владелец: `root:root`
  - Права: `600` (`-rw-------`) — доступен исключительно суперпользователю.
- **Сертификат:** `/etc/ssl/certs/devops.crt`
  - Владелец: `root:root`
  - Права: `644` (`-rw-r--r--`) — публично доступен для чтения.
- **Срок действия:** 365 дней.
- **Команда генерации сертификата:**
  ```bash
  sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/devops.key \
    -out /etc/ssl/certs/devops.crt \
    -subj "/CN=devops.local" \
    -addext "subjectAltName=DNS:devops.local"
  ```

### 8.3 Конфигурация виртуального хоста Nginx
- **Путь к файлу конфигурации:** `/etc/nginx/sites-available/devops-site`
- **Активация:** Символическая ссылка в `/etc/nginx/sites-enabled/devops-site`
- **Стандартная конфигурация:** `/etc/nginx/sites-enabled/default` удалена.

**Содержимое конфигурации:**
```nginx
# Блок 1: перенаправление HTTP -> HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name devops.local;

    return 301 https://$host$request_uri;
}

# Блок 2: основной HTTPS-ресурс
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name devops.local;

    ssl_certificate /etc/ssl/certs/devops.crt;
    ssl_certificate_key /etc/ssl/private/devops.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/devops-site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /404.html;

    access_log /var/log/nginx/devops-site.access.log;
    error_log /var/log/nginx/devops-site.error.log;
}
```

- **Команда проверки синтаксиса:** `sudo nginx -t`
- **Команда применения изменений без простоя:** `sudo systemctl reload nginx`
```

## 9. Контейнеризация
* Способ установки: официальный репозиторий Docker (download.docker.com), Docker Engine CE (версия X.X.X), плагин docker-compose-plugin.
* Пользователи группы docker: devops.
* Расположение проекта: /home/devops/app.
* Переменные окружения: /home/devops/app/.env, права доступа 600 (чтение/запись только владельцу).
* Публикация портов: только 127.0.0.1:5000 (проксируется в Nginx), порт СУБД 5432 не публикуется на хосте.
* Тома данных: именованный том db_data для каталога PostgreSQL /var/lib/postgresql/data.
* Обратный прокси-сервер: Nginx перенаправляет запросы с /app/ на http://127.0.0.1:5000/ со сбросом префикса и пробросом заголовков Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto.

## 10. Ansible

### 10.1. Общие сведения и архитектура
* **Расположение проекта:** каталог `ansible/` в корне репозитория `devops_course_2026`.
* **Управляющий узел (Control Node):** рабочая станция разработчика под управлением macOS (`ansible-core 2.21`, Python 3.15, коллекции `ansible.posix 1.6.2`, `community.docker 4.5.0`, `community.general 9.5.0`).
* **Управляемый узел (Managed Node):** виртуальная машина VirtualBox (`devops-vm`), сетевой адаптер Host-only (`192.168.56.2`), порт SSH `22`, ОС Ubuntu Server 26.04+ LTS. Не требует установки постоянного агентского ПО — управление осуществляется модулями Ansible по SSH с выполнением через системный интерпретатор Python.

### 10.2. Инвентарь и учетные записи
* **Inventory bootstrap (`inventory/bootstrap.ini`):** используется исключительно для первичной настройки чистой виртуальной машины. Подключение выполняется от учетной записи `student` с интерактивным запросом паролей:
  ```ini
  [vm]
  devops-vm ansible_host=192.168.56.2 ansible_port=22 ansible_user=student
  ```
* **Inventory prod (`inventory/prod.ini`):** используется по умолчанию (`ansible.cfg`) для всех регулярных запусков, развертывания обновлений и аудита. Подключение выполняется от сервисного пользователя `devops` строго по SSH-ключу:
  ```ini
  [vm]
  devops-vm ansible_host=192.168.56.2 ansible_port=22 ansible_user=devops ansible_private_key_file=~/.ssh/devops_vm
  ```
* **Учетные записи:**
  * `student` — пользователь начальной установки системы. После выполнения bootstrap парольная аутентификация по SSH отключается; доступ сохраняется через консоль VM и по публичным ключам.
  * `devops` — выделенный пользователь автоматизации, создаваемый ролью `ssh`. Включен в группы `sudo` и `docker`. Настроен беспарольный sudo (`/etc/sudoers.d/devops`: `devops ALL=(ALL) NOPASSWD:ALL`, валидация через `visudo -cf`). Аутентификация осуществляется по ключу `~/.ssh/devops_vm.pub` (`mode: 0400/0700`).

### 10.3. Роли Ansible и их зоны ответственности
1. **`base`:**
   * Проверка целевой ОС (Ubuntu 24.04+ / amd64 / arm64).
   * Установка базовых системных пакетов и библиотек (`ca-certificates`, `curl`, `gnupg`, `openssh-server`, `ufw`, `python3-debian`, `python3-requests`, `qemu-user`, `qemu-user-binfmt`).
   * Подключение официального репозитория Docker через `deb822_repository` с проверкой GPG-ключа.
   * Установка пакетов Docker Engine (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`) и запуск сервиса `docker`.
2. **`ssh`:**
   * Создание пользователя `devops` и каталога `~/.ssh`.
   * Размещение открытого ключа `devops_vm.pub` в `authorized_keys`.
   * Настройка файла `/etc/sudoers.d/devops` с валидацией через `visudo`.
   * Ужесточение конфигурации SSH (`/etc/ssh/sshd_config.d/00-ansible-hardening.conf`): отключение паролей (`PasswordAuthentication no`), запрет входа root (`PermitRootLogin no`), ограничение пользователей (`AllowUsers student devops`), валидация через `sshd -t -f` и вызов обработчика `Restart SSH`.
   * Настройка межсетевого экрана UFW: разрешение портов `22/tcp` и `80/tcp`, блокировка всех остальных входящих соединений (`default deny incoming`), разрешение исходящих (`default allow outgoing`).
3. **`nginx`:**
   * Установка веб-сервера Nginx.
   * Развертывание обратного прокси-сервера из шаблона `devops-ci-app.conf.j2` в `/etc/nginx/sites-available/devops-ci-app`.
   * Отключение дефолтной конфигурации (`/etc/nginx/sites-enabled/default`) и активация сайта приложения через символическую ссылку.
   * Безопасное применение изменений: связка обработчиков `Validate Nginx before reload` (`nginx -t`) и `Reload Nginx` (`systemctl reload nginx`).
4. **`app`:**
   * Добавление пользователя `devops` в группу `docker`.
   * Загрузка проверенного OCI-образа из реестра GHCR по полному SHA коммита.
   * Запуск контейнера `devops-ci-app` с изоляцией порта `127.0.0.1:5000:5000` и политикой перезапуска `unless-stopped`.
   * Контроль работоспособности приложения через периодический опрос эндпоинта `/health` модулем `uri` с проверкой HTTP 200 и тела `{"status":"ok"}`.

### 10.4. Параметры приложения и контейнера
* **Реестр и образ:** `ghcr.io/karpx/devops-ci-app`
* **Фиксированный тег (полный SHA коммита из PR № 10):** `37e9e52bfbc20cf52010e2741dd6e6042ae109b9`
* **Имя контейнера:** `devops-ci-app`
* **Публикация портов:** `127.0.0.1:5000:5000` (порт 5000 привязан исключительно к loopback-интерфейсу хоста, прямой доступ из внешней сети исключен; входящий веб-трафик проходит через Nginx на порту 80).
* **Эндпоинты проверки доступности:**
  * Внутренний (прямой к приложению): `http://127.0.0.1:5000/health`
  * Внешний (через обратный прокси Nginx): `http://192.168.56.2/health`
  * Основной маршрут: `http://192.168.56.2/`

### 10.5. Регламентные команды управления
* **Проверка инвентаря и синтаксиса:**
  ```bash
  ansible-inventory --graph
  ansible-playbook site.yml --syntax-check
  ```
* **Первичный запуск (Bootstrap чистой виртуальной машины):**
  ```bash
  ansible-playbook -i inventory/bootstrap.ini site.yml --ask-pass --ask-become-pass
  ```
* **Регулярный запуск и развертывание (Prod, по ключу):**
  ```bash
  ansible-playbook site.yml
  ```
  *(При отсутствии дрейфа конфигурации повторный запуск возвращает `changed=0, failed=0`).*
* **Автоматический read-only аудит инфраструктуры:**
  ```bash
  ansible-playbook verify.yml
  ```
  *(Выполняет аудит фактических директив sshd, состояния контейнера Docker и ответа healthcheck без внесения изменений).*
