# Алгоритм развертывания сайта на vps-сервере

Стек:
1. frontend - Django templates (HTML, CSS, JavaScript)
2. backend - python3.11, Django
3. database - PostgreSQL, Redis
4. webserver - Nginx + Gunicorn

## 1. Стартовая настройка VPS
<i>Процесс аренды сервера не описываю, т.к. у разных провайдеров может отичаться.<br>
Также подразумевается, что доменное имя уже пробретено и ns-сервера настроены на наш сервер.<br>
VPS под управлением Ubuntu server 22.04.<br>
Следующие шаги предполагают, что выполнен первоначальный вход на сервер под root user.</i>

### 1. Обновление ОС:
```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```
### 2. Создание нового пользователя (назову его dude) и сразу добавление в группу sudo:
```bash
adduser dude
usermod -aG sudo dude
```
### 3. Переключение на нового пользователя. С этого момента все команды выполняются от имени dude:
```bash
su - dude
```
### 4. Создание нового ssh-ключа типа ED25519 на <strong>локальной машине</strong>:
```bash
ssh-keygen -t ed25519 -C "key for dude vps" -f dude_key
touch ~/.ssh/config && chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/dude_key.pub && chmod 400 ~/.ssh/dude_key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/dude_key
xclip -sel c < dude_key.pub
```
### 5. Подключение SSH на <b>удаленном сервере</b>:
вставляем публичный ключ из буфера обмена:
```bash
sudo nano /home/dude/.ssh/authorized_keys
```
### 6. Отключение входа по паролю и запрет подключения для root user:
```bash
sudo nano /etc/ssh/sshd_config
...
PermitRootLogin no
...
PasswordAuthentication no
...
```
### 7. Перезагрузка службы ssh:
```bash
sudo systemctl restart sshd 
```
### 8. ОПЦИОНАЛЬНО. Подключение GitHub репозитория к серверу:<br>
копируем приватный ключ с локальной машины:
```bash
scp /home/alex/.ssh/<my_github_key> dude@<server_ip>:/home/dude/.ssh
```
добавляем ключ в ssh агент на сервере:
```bash
eval "$(ssh-agent -s)"
ssh-add /home/dude/.ssh/<my_github_key>
```
клонируем проект из репозитория на сервер в папку /home/dude/projects:
```bash
mkdir /home/dude/projects
cd /home/dude/projects
git clone <my_repository_clone_link>
```
### 9. Копирование файлов проекта с <b>локальной машины</b> на сервер:
```bash
rsync -Praz /home/alex/my_project dude@<server_ip>:/home/dude/projects/my_project
```

## 2. Стартовая настройка веб-сервера
<i>NOTE: ufw лучше настраивать до того, как отключать вход на сервер по паролю. Но у меня не заблокировался вход на сервер.</i>

### 1. Установка тайм-зоны, установка nginx и ufw:
```bash
timedatectl set-timezone Europe/Moscow
sudo apt install nginx
sudo apt install ufw
```
проверяю наличие настроек nginx у файервола:
```bash
sudo nano /etc/ufw/applications.d/nginx
```
содержимое файла:
```
[Nginx HTTP]
title=Web Server (Nginx, HTTP)
description=Small, but very powerful and efficient web server
ports=80/tcp

[Nginx HTTPS]
title=Web Server (Nginx, HTTPS)
description=Small, but very powerful and efficient web server
ports=443/tcp

[Nginx Full]
title=Web Server (Nginx, HTTP + HTTPS)
description=Small, but very powerful and efficient web server
ports=80,443/tcp
```
включаю файерволл:
```bash
sudo ufw enable
```
проверяю доступные приложения:
```bash
sudo ufw app list
```
```
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH
```
проверяю статус файервола:
```bash
sudo ufw status
```
```
Status: active
```
добавляю Nginx и SSH в доступные соединения:
```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'
```
проверяю статус файервола после добавления приложений:
```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```
Такой вывод корректный. UFW разрешает входящие соединения OpenSSH на стандартном порту 22 и Nginx на 80, 443 портах<br>

## 3. Стартовая настройка django-проекта
### 1. Установка виртуального окружения
<i>В качестве пакетного менеджера использую poetry</i>

```bash
cd /home/dude/projects/my_project
sudo apt install python3.11
sudo apt install python3.11-venv
python3.11 -m venv venv
source ./venv/bin/activate
pip install --upgrade pip
pip install poetry
poetry install
deactivate
```
### 2. Установка и настройка БД PostgreSQL
```bash
sudo apt install postgresql postgresql-contrib
sudo -i -u postgres
  psql
    \l
    \du
    CREATE DATABASE projectdb;
    CREATE USER projectuser WITH PASSWORD 'projectpassword';
    GRANT ALL PRIVILEGES ON DATABASE projectdb TO projectuser;
    ALTER DATABASE projectdb OWNER TO projectuser;
    ALTER USER postgres WITH PASSWORD 'newpostgrespassword';
    \l
    \du
    \q
  exit
```
проверяем, что вход в БД под новым пользователем работает:
```bash
psql -h localhost projectdb projectuser;
\q
```
### 3. Установка и настройка Redis
```bash
sudo apt install redis-server
```
проверяю, что redis слушает только локальные соединения:
```bash
sudo nano /etc/redis/redis.conf
```
ожидаемая настройка bind:
```
...
bind 127.0.0.1 ::1
...
```
устанавливаю пароль для подключения к redis:
```
...
requirepass myredispassword
...
```
перезапускаю службу Redis:
```bash
sudo systemctl restart redis
```
### 4. Применение миграций django-проекта и создание суперпользователя:
```bash
cd /home/dude/projects/my_project
source ./venv/bin/activate
cd ./my_project
poetry run python manage.py migrate
poetry run python manage.py createsuperuser
deactivate
```

## 4. Настройка веб-сервера под django-проект
### 1. Gunicorn
<i>Сам gunicorn установлен в виртуальном окружении джанго-проекта</i><br>

создаем службу:
```bash
sudo nano /etc/systemd/system/my_project.service
```
```
[Unit]
Description=my_project daemon
Requires=my_project.socket
After=network.target

[Service]
User=dude
WorkingDirectory=/home/dude/projects/my_project/my_project/
ExecStart=/home/dude/projects/my_project/venv/bin/gunicorn\
--workers 3\
--bind unix:/run/my_project.sock my_project.wsgi:application\
--access-logfile /home/dude/projects/my_project/my_project/logs/service/access.log\
--error-logfile /home/dude/projects/my_project/my_project/logs/service/error.log\

[Install]
WantedBy=multi-user.target
```
создаем сокет:
```bash
sudo nano /etc/systemd/system/my_project.socket
```
```
[Unit]
Description=my_project socket

[Socket]
ListenStream=/run/my_project.sock

[Install]
WantedBy=sockets.target
```
проверка файла my_project.serice на наличие ошибок (пустой вывод - OK):
```bash
sudo systemd-analyze verify my_project.service
```
включение службы:
```bash
sudo systemctl enable my_project
sudo systemctl start my_project
```
проверка создания сокета:
```bash
file /run/my_project.sock
```
корректный вывод:
```
/run/my_project.sock: socket
```

### 2. SSL Certbot
1. Создаю ssl-ключ:
    ```bash
    sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
    ```

2. Создаю конфиг ssl-параметров:
    ```bash
    sudo nano /etc/nginx/snippets/ssl-params.conf
    ```
    ```plaintext
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    add_header Strict-Transport-Security "max-age=63072000" always;
    ```

3. Устанавливаю snap, cerbot
    ```bash
    sudo apt install snapd
    sudo snap install core
    sudo snap install --classic certbot
    sudo snap refresh
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    ```

4. Устанавливаю сертификат. В ходе установки указать почту, на которую будут приходить уведомления об истечении срока действия, указать домены через пробел:
    ```bash
    sudo certbot certonly --nginx
    ```
5. Меняю владельца папки с сертификатами:
    ```bash
    sudo chown dude /etc/letsencrypt/live/
    ```
### 2. Nginx
1. Меняю пользователя в основном конфиге nginx:
    ```bash
    sudo nano /etc/nginx/nginx.conf
    ```
    ```
    user dude
    ...
    ```
2. Отключаю дефолтный конфиг в sites-enabled:
    ```bash
    sudo rm /etc/nginx/sites-enabled/default
    ```

3. Создаю конфиг для джанго-проекта:
    ```bash
    sudo nano /etc/nginx/sites-available/my_project
    ```
    <i>вместо my_domain.ru используется заранее приобретенный и настроенный (ns-сервера) домен;</i><br>
    <i>поскольку проект предполагает воспроизведение видео на сайте, в конфиге приведены настройки отдачи .mp4 файлов</i>
    ```plaintext
    server {
      listen 80;
      listen [::]:80;

      server_name my_domain.ru www.my_domain.ru;
      return 301 https://www.my_domain.ru$request_uri;

      access_log /var/log/nginx/access.log;
    }

    server {
      listen 443 ssl http2;
      listen [::]:443 ssl http2;#

      server_name www.my_domain.ru;
      return 301 https://my_domain.ru$request_uri;

      access_log /var/log/nginx/access.log;

      ssl_certificate /etc/letsencrypt/live/my_domain.ru/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/my_domain.ru/privkey.pem;
      ssl_trusted_certificate /etc/letsencrypt/live/my_domain.ru/chain.pem;

      include snippets/ssl-params.conf;
    }

    server {
      listen 443 ssl http2;
      listen [::]:443 ssl http2;

      server_name my_domain.ru;

      access_log /var/log/nginx/access.log;

      ssl_certificate /etc/letsencrypt/live/my_domain.ru/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/my_domain.ru/privkey.pem;
      ssl_trusted_certificate /etc/letsencrypt/live/my_domain.ru/chain.pem;

      include snippets/ssl-params.conf;

      location ~ ^/static/video/(.*\.mp4)$ {
          root /home/dude/projects/my_project/my_project;

          sendfile on;
          tcp_nopush on;
          tcp_nodelay on;
          keepalive_requests 100;
          keepalive_timeout 65;

          # Включаем кэширование файла
          open_file_cache max=1000 inactive=20s;
          open_file_cache_valid 30s;
          open_file_cache_min_uses 2;
          open_file_cache_errors on;

          # Настраиваем сжатие
          gzip on;
          gzip_comp_level 2;
          gzip_min_length 1000;
          gzip_types video/mp4;

          # Дополнительные настройки безопасности и производительности
          client_max_body_size 10m;
          client_body_buffer_size 128k;
          client_body_in_file_only off;
          client_body_temp_path /home/dude/.cache/nginx_temp;

          # Настройки буферизации
          output_buffers 1 32k;
          postpone_output 1460;

          # Настройки логирования
          access_log /var/log/nginx/mp4_access.log;
          error_log /var/log/nginx/mp4_error.log;
      }

      location /static/ {
          root /home/dude/projects/my_project/my_project;
      }

      location /media/ {
          alias /home/dude/projects/my_project/my_project/media/;
      }

      location / {
          proxy_set_header Host $http_host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;

          proxy_pass http://unix:/run/my_project.sock;
      }
    }
    ```
4. Создаю симлинк на конфиг проекта в sites-enabled:
    ```bash
    sudo ln -s /etc/nginx/sites-available/my_project /etc/nginx/sites-enabled/
    ```

5. Меняю владельца файлов логов nginx:
    ```bash
    sudo chown dude /var/log/nginx/access.log
    sudo chown dude /var/log/nginx/error.log
    ```

6. Перезагрузка веб-сервера для применения всех изменений:<br>
    Проверка конигурации nginx:
    ```bash
    sudo nginx -t
    ```
    Перезапуск nginx:
    ```bash
    sudo systemctl restart nginx
    ```
    Проверка статуса nginx:
    ```bash
    sudo systemctl status nginx
    ```

## 5. Финальные проверки / запуски служб:
- проверка конфигурации nginx
  ```bash
  sudo nginx -t
  ```

- проверка конфигурации gunicorn
  ```bash
  systemd-analyze verify my_project.service
  ```

- запускаем службу my_project и создаем socket:
  ```bash
  sudo systemctl enable my_project
  sudo systemctl start my_project
  ```
- для отключения выполняем команды:
  ```bash
  sudo systemctl disable my_project
  sudo systemctl stop my_project
  ```

- для проверки создания сокета, необходимо ввести команду:
  ```bash
  file /run/my_project.sock
  ```
  такой вывод считается правильным: /run/my_project.sock: socket

- если внес какие-либо изменения в файл my_project.service или .socket, необходимо выполнить команду:
  ```bash
  sudo systemctl daemon-reload
  ```

- при изменении шаблонов, перезапускается сервис my_project:
  ```bash
  sudo systemctl restart my_project
  ```

- общая перезагрузка:
  ```bash
  sudo systemctl restart my_project && sudo systemctl daemon-reload && sudo systemctl restart nginx
  ```
