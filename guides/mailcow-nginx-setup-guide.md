# Гайд по установке Mailcow с Nginx в качестве Reverse Proxy (Linux Ubuntu)

**Цель:** Запустить Mailcow, чтобы его веб-интерфейс был доступен по `mail.your-domain.tld` через наш основной Nginx, а почтовые службы работали на стандартных портах.

**Предварительные Условия:**

1. **Сервер:** Ubuntu с минимум 4GB RAM (рекомендуется 6-8GB) и достаточными ресурсами CPU.
2. **Docker и Docker Compose:** Установлены и работают.
3. **Git:** Установлен.
4. **rDNS (PTR-запись):** Для IP-адреса нашего сервера настроена PTR-запись, указывающая на `mail.your-domain.tld` (настраивается у хостинг-провайдера).
5. **Wildcard SSL-сертификат:** Для `*.your-domain.tld` у нас уже есть. (либо в Шаге 2) не отказывайтесь от Let's Encrypt)

---

В этом руководстве вместо конкретного имени домена и названия компании мы будем использовать следующие заглушки:

* **`your-domain.tld`**: Это ваш основной домен. Например, если ваш сайт `mycompany.com`, то здесь вы будете использовать `mycompany.com`. Пожалуйста, замените `your-domain.tld` на ваш реальный домен во всех инструкциях ниже.
* **`mail.your-domain.tld`**: Это полное доменное имя (FQDN) вашего почтового сервера. Обычно это субдомен вашего основного домена, например, `mail.mycompany.com`. Замените `mail.your-domain.tld` на FQDN, который вы будете использовать для почтового сервера.
* **`dmarc-reports@your-domain.tld`**: Это пример адреса электронной почты для получения DMARC-отчетов. Замените `your-domain.tld` на ваш домен (например, `dmarc-reports@mycompany.com`).

---

## **Шаг 1: Подготовка Системы**

1. **Обновление и установка зависимостей:**

    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl docker-compose
    ```

2. **Настройка Firewall (ufw):**

    ```bash
    sudo ufw allow 25/tcp   # SMTP
    sudo ufw allow 143/tcp  # IMAP
    sudo ufw allow 465/tcp  # SMTPS (SMTP over SSL)
    sudo ufw allow 587/tcp  # SMTP Submission (STARTTLS)
    sudo ufw allow 993/tcp  # IMAPS (IMAP over SSL)
    sudo ufw enable # Если еще не включен
    sudo ufw status
    ```

    Порты 80 и 443 уже должны быть открыты для нашего Nginx.

## **Шаг 2: Начальная Установка Mailcow**

1. **Клонирование репозитория:**

    ```bash
    cd /opt # или другая удобная директория
    git clone https://github.com/mailcow/mailcow-dockerized
    cd mailcow-dockerized
    ```

2. **Генерация конфигурации:**

    ```bash
    ./generate_config.sh
    ```

    * При запросе **Mail server hostname (FQDN)** введите: `mail.your-domain.tld`.

**Шаг 3: Настройка `mailcow.conf`**

Откройте `mailcow.conf` для редактирования: `nano mailcow.conf`.
Убедитесь или измените следующие параметры:

* `MAILCOW_HOSTNAME=mail.your-domain.tld`
* **Порты для внутреннего Nginx Mailcow (чтобы не конфликтовать с хостовым):**

    ```ini
    HTTP_PORT=8088       # Пример, любой свободный порт
    HTTPS_PORT=8448      # Пример, любой свободный порт
    HTTP_BIND=127.0.0.1  # Слушать только на localhost
    HTTPS_BIND=127.0.0.1 # Слушать только на localhost
    ```

* **Пропуск Let's Encrypt для UI (SSL-терминация на нашем Nginx):**

    ```ini
    SKIP_LETS_ENCRYPT=y
    ```

Сохраните файл.

## **Шаг 4: SSL-Сертификаты для Служб Mailcow**

Даже с `SKIP_LETS_ENCRYPT=y`, Postfix и Dovecot внутри Mailcow нуждаются в SSL-сертификатах.

1. **Создайте директорию (если ее нет):**

    ```bash
    mkdir -p data/assets/ssl
    ```

2. **Скопируйте ваш wildcard сертификат:**
    Замените `/ПУТЬ/К/ВАШЕМУ/` на актуальные пути к вашим файлам сертификата.

    ```bash
    sudo cp /ПУТЬ/К/ВАШЕМУ/fullchain.pem data/assets/ssl/cert.pem
    sudo cp /ПУТЬ/К/ВАШЕМУ/privkey.pem data/assets/ssl/key.pem
    ```

3. **Установите права (рекомендуется):**

    ```bash
    sudo chown -R $(id -u):$(id -g) data/assets/ssl/ # Или пользователя, от которого запускаются контейнеры, если отличается
    sudo chmod 600 data/assets/ssl/key.pem
    sudo chmod 644 data/assets/ssl/cert.pem
    ```

## **Шаг 5: Запуск Mailcow**

1. **Загрузка образов:**

    ```bash
    docker-compose pull
    ```

2. **Первый запуск:**

    ```bash
    docker-compose up -d
    ```

3. **Проверка состояния:**

    ```bash
    docker-compose ps
    ```

    Все контейнеры должны быть `Up` или `healthy`.

## **Шаг 6: Настройка Nginx Хоста (Reverse Proxy)**

Создайте файл конфигурации для Nginx, например, `/etc/nginx/sites-available/mail.your-domain.tld`:

```bash
sudo nano /etc/nginx/sites-available/mail.your-domain.tld
```

Вставьте следующую базовую конфигурацию, заменив `/ПУТЬ/К/ВАШЕМУ/` на пути к вашим сертификатам и порт `8088` (если вы выбрали другой в `mailcow.conf`):

```nginx
server {
    listen 80;
    server_name mail.your-domain.tld;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name mail.your-domain.tld;

    ssl_certificate /ПУТЬ/К/ВАШЕМУ/fullchain.pem;
    ssl_certificate_key /ПУТЬ/К/ВАШЕМУ/privkey.pem;

    # Рекомендуемые SSL параметры (если нет глобальных)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    client_max_body_size 100M; # Для больших вложений

    # Проксирование на Mailcow Nginx (HTTP)
    location / {
        proxy_pass http://127.0.0.1:8088; # Используйте HTTP_PORT из mailcow.conf
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade"; # Для WebSocket
    }

    # Microsoft ActiveSync (если используется)
    location /Microsoft-Server-ActiveSync {
        proxy_pass http://127.0.0.1:8088/Microsoft-Server-ActiveSync;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 75;
        proxy_send_timeout 3650;
        proxy_read_timeout 3650;
        proxy_buffering off;
    }

    # Autodiscover/Autoconfig и другие .well-known
    location ~ ^/\.well-known/(carddav|caldav|autoconfig/mail/config-v1\.1\.xml|mta-sts\.txt) {
        proxy_pass http://127.0.0.1:8088$request_uri;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/mail.your-domain.tld.access.log;
    error_log /var/log/nginx/mail.your-domain.tld.error.log;
}
```

Активируйте сайт и перезагрузите Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/mail.your-domain.tld /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## **Шаг 7: DNS Конфигурация**

Войдите в веб-интерфейс Mailcow (`https://mail.your-domain.tld`). Логин по умолчанию: `admin`, пароль: `moohoo`. **Сразу смените пароль!**

Перейдите в `E-Mail -> Конфигурация -> Домены`. Если домен `your-domain.tld` еще не добавлен, добавьте его. Затем нажмите на кнопку **DNS** рядом с вашим доменом. Mailcow покажет список рекомендуемых DNS-записей. Вам необходимо создать/обновить следующие записи у вашего DNS-провайдера, используя значения, **предоставленные Mailcow UI**:

1. **`A` запись:**
    * Имя: `mail` (для `mail.your-domain.tld`)
    * Значение: `IP-АДРЕС_ВАШЕГО_СЕРВЕРА`
2. **`MX` запись:**
    * Имя: `@` (или `your-domain.tld.`)
    * Значение: `mail.your-domain.tld.`
3. **`PTR` запись (rDNS):** (Настраивается у хостинг-провайдера, должна уже быть сделана)
    * `IP-АДРЕС_ВАШЕГО_СЕРВЕРА` -> `mail.your-domain.tld`
4. **`TXT` запись (SPF):**
    * Имя: `@` (или `your-domain.tld.`)
    * Значение: `v=spf1 a:mail.your-domain.tld -all` (Используйте это или то, что предложит Mailcow UI)
5. **`TXT` запись (DMARC):**
    * Имя: `_dmarc` (для `_dmarc.your-domain.tld`)
    * Значение: `v=DMARC1; p=none; rua=mailto:dmarc-reports@your-domain.tld; ruf=mailto:dmarc-reports@your-domain.tld; fo=1; adkim=r; aspf=r; pct=100; rf=afrf; ri=86400; sp=reject` (Можно сделать по предлагаемому Mailcow гайду или настроить проще, например, `v=DMARC1; p=none; rua=mailto:dmarc-reports@your-domain.tld;`).
6. **`TXT` запись (DKIM):**
    * Имя: `dkim._domainkey` (или другое имя селектора, если Mailcow его изменит)
    * Значение: *Копируется из Mailcow UI (секция ARC/DKIM keys)*
7. **`CNAME` запись (autodiscover):**
    * Имя: `autodiscover`
    * Значение: `mail.your-domain.tld` (или как указано в Mailcow UI)
8. **`CNAME` запись (autoconfig):**
    * Имя: `autoconfig`
    * Значение: `mail.your-domain.tld` (или как указано в Mailcow UI)
9. **`SRV` запись (_autodiscover):**
    * Имя: `_autodiscover._tcp`
    * Значение: *Копируется из Mailcow UI (например: `0 0 443 mail.your-domain.tld`)*
10. **`TLSA` записи (DANE):**
    * Имена: например, `_25._tcp.mail`, `_465._tcp.mail`, `_587._tcp.mail` (для FQDN `mail.your-domain.tld`, эти имена будут `_25._tcp.mail.your-domain.tld`, и т.д. Mailcow UI покажет точные имена)
    * Значения: *Копируются из Mailcow UI*.
    * **Примечание:** TLSA-записи для DANE обеспечивают дополнительную безопасность при установке TLS-соединений. Однако их эффективность полностью зависит от **DNSSEC**. Если DNS-провайдер не поддерживает DNSSEC для домена, эти записи не будут иметь защитного эффекта для внешних систем.

**Ключевой момент:** Для DKIM, SRV, TLSA и, возможно, других специфичных записей **всегда используйте значения, которые генерирует и показывает Mailcow UI** в разделе DNS для вашего домена.

## **Шаг 8: Финальная Проверка и Тестирование**

1. **DNS Распространение:** Подождите некоторое время. Проверьте DNS-записи с помощью `mxtoolbox.com` или аналогичных сервисов.
2. **Создание Ящика:** В Mailcow UI: `Почтовые ящики -> Добавить ящик`.
3. **Настройка Клиента:** Используйте данные, предоставленные Mailcow (обычно `mail.your-domain.tld` для IMAP/SMTP серверов, SSL/TLS порты 993/465, STARTTLS порты 143/587).
4. **Тест Отправки/Получения:** Отправьте письма на внешние адреса (Gmail, Outlook и т.д.) и с них на ваш новый ящик. Проверьте папку "Спам".
5. **Mail-Tester:** Отправьте письмо на адрес, сгенерированный `mail-tester.com`, и проанализируйте отчет.

## **Шаг 9: Обслуживание**

1. **Бэкапы:** Настройте регулярные бэкапы с помощью скрипта Mailcow:

    ```bash
    cd /opt/mailcow-dockerized/helper-scripts # Убедитесь, что вы в правильной директории
    ./backup_and_restore.sh backup all --delete-days 7 # Пример: бэкап всего, хранить 7 дней
    ```

    Копируйте бэкапы на внешнее хранилище.

2. **Обновления:** Регулярно обновляйте Mailcow:

    ```bash
    cd /opt/mailcow-dockerized # Убедитесь, что вы в правильной директории
    ./update.sh
    # Прочитайте changelog перед обновлением, особенно если есть предупреждения
    # docker-compose pull # Иногда требуется перед update.sh
    # docker-compose up -d # После update.sh
    ```

3. **Мониторинг:** Следите за логами (`docker-compose logs -f ИМЯ_КОНТЕЙНЕРА`), ресурсами сервера.

4. **DMARC:** После периода мониторинга (`p=none`), если отчеты (`rua`, `ruf`) показывают, что все легитимные письма проходят SPF/DKIM, постепенно ужесточайте политику DMARC до `p=quarantine`, а затем до `p=reject`.

Удачи!
