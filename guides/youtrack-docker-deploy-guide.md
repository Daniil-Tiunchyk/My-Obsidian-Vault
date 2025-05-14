# Установка YouTrack на сервер с Docker (Linux Ubuntu)

Этот гайд описывает процесс установки YouTrack с использованием Docker и автоматическим перезапуском контейнера.

## 1. Подключение к серверу

Подключитесь к вашему серверу по SSH:

```bash
ssh root@ip_adress
```

Замените `ip_adress` на IP-адрес вашего сервера. При первом подключении подтвердите добавление хоста, введя `yes`.

## 2. Обновление системы

Обновите пакеты операционной системы до актуальных версий:

```bash
sudo apt update && sudo apt upgrade -y
```

## 3. Установка Docker

Для работы YouTrack в контейнере необходим Docker.

### Обновление списка пакетов и установка зависимостей

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
```

### Добавление GPG-ключа и репозитория Docker

Добавьте официальный GPG-ключ Docker:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Добавьте репозиторий Docker:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Установка Docker Engine

Обновите список пакетов и установите Docker:

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### Проверка установки Docker

Убедитесь, что Docker запущен и работает:

```bash
sudo systemctl status docker
```

(Нажмите `q` для выхода)

### (Рекомендуется) Настройка прав для Docker

Чтобы выполнять команды `docker` без `sudo`, добавьте вашего пользователя в группу `docker`. Потребуется перелогиниться или выполнить `newgrp docker`.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 4. Создание каталогов для YouTrack

Создадим директории для данных, конфигурации, логов и бэкапов YouTrack в `/opt/youtrack` и установим необходимые права доступа (UID/GID `13001:13001` используется пользователем внутри контейнера YouTrack):

```bash
BASE_DIR="/opt/youtrack"
for i in data logs conf backups; do sudo mkdir -p -m 750 $BASE_DIR/$i && sudo chown -R 13001:13001 $BASE_DIR/$i ; done
```

Проверьте создание и права:

```bash
ls -ld $BASE_DIR/*
```

Вывод должен показать, что владельцем каталогов является пользователь `13001`.

## 5. Загрузка образа YouTrack

Актуальные версии образов доступны на [Docker Hub](https://hub.docker.com/r/jetbrains/youtrack/tags). Загрузим указанную версию (в моём случае это 2025.1.74704):

```bash
docker pull jetbrains/youtrack:2025.1.74704
```

## 6. Запуск контейнера YouTrack

Запустим контейнер YouTrack в фоновом режиме с автоматическим перезапуском, привязкой к локальному порту `8080` и монтированием подготовленных каталогов:

```bash
docker run -d \
  --name youtrack \
  -v /opt/youtrack/data:/opt/youtrack/data \
  -v /opt/youtrack/conf:/opt/youtrack/conf \
  -v /opt/youtrack/logs:/opt/youtrack/logs \
  -v /opt/youtrack/backups:/opt/youtrack/backups \
  -p 127.0.0.1:8080:8080 \
  --restart always \
  jetbrains/youtrack:2025.1.74704
```

Ключевые параметры команды:

* `-d`: фоновый (detached) режим.
* `--name youtrack`: имя контейнера для удобства управления.
* `-v /opt/youtrack/...:/opt/youtrack/...`: монтирование томов для сохранения данных YouTrack на хост-машине.
* `-p 127.0.0.1:8080:8080`: привязка порта `8080` контейнера к порту `8080` на `localhost` сервера. Это означает, что YouTrack будет доступен только с самого сервера.
* `--restart always`: автоматический перезапуск контейнера при сбоях или после перезагрузки сервера.

## 7. Доступ к YouTrack и первоначальная настройка

YouTrack теперь работает на вашем сервере и слушает порт `8080` только на локальном интерфейсе (`127.0.0.1`). Для доступа к веб-интерфейсу используйте один из следующих методов:

### Вариант А: SSH-туннелирование (для начальной настройки)

На вашем локальном компьютере выполните:

```bash
ssh -L 8080:127.0.0.1:8080 root@ip_adress
```

Замените `ip_adress` на IP вашего сервера. Затем откройте в браузере `http://localhost:8080/`. Этот туннель будет активен, пока открыта SSH-сессия.

### Вариант Б: Использование обратного прокси (Nginx, Apache)

Это предпочтительный способ для постоянного доступа, позволяющий использовать доменное имя и HTTPS. Пример базовой конфигурации Nginx (файл `/etc/nginx/sites-available/youtrack`):

```nginx
server {
    listen 80;
    server_name youtrack.yourdomain.com; # Замените на ваш домен или IP-адрес

    location / {
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8080;
        client_max_body_size 10M; # Увеличьте при необходимости
    }
}
```

Активируйте конфигурацию и перезапустите Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/youtrack /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

После настройки DNS для `youtrack.yourdomain.com` вы сможете получить доступ по этому адресу. Рекомендуется также настроить HTTPS с помощью Let's Encrypt.

После первого входа в YouTrack следуйте инструкциям мастера настройки.

## Полезные команды Docker

* Просмотр логов контейнера: `docker logs youtrack`
* Остановка контейнера: `docker stop youtrack`
* Запуск остановленного контейнера: `docker start youtrack`
* Удаление контейнера (данные в `/opt/youtrack` сохранятся): `docker rm youtrack` (контейнер должен быть остановлен)
