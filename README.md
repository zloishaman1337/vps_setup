<h1 align="center">Настройка VPS с нуля + Marzban + VLESS TCP REALITY + nginx steal_oneself + WireGuard с веб-интерфейсом</h2>

Данный гайд не предусматривает открытия портов, но если что вот список портов, необходимых для открытия:
```bash
22
8000
51821
443
1443
8443
```

### Требования:
- VPS
- Ubuntu 22.04 и выше
- Доступ по SSH, если еще нет
- Привязка двух доменов к айпи VPS
	- example.ru - fallback
	- www.example.ru - VLESS Трафик
### Зависимости:
`docker fail2ban mc net-tools cron socat curl wget iperf3 nginx libnginx-mod-stream`

## Настройка сервера
Апдейт системы, если еще не делали
```bash
sudo apt update && sudo apt upgrade -y
```
### Установка нужного ПО
Может и не очень нужного, однако я его ставлю в большинстве случаев
```bash
sudo apt install -y fail2ban mc net-tools cron socat curl wget iperf3
```
### Включение bbr 
Это поможет увеличить пропускную способность сетевого интерфейса
```bash
sudo echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
sudo echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sudo sysctl -p
```
### Настройка fail2ban
Утилита для автоматического бана хостов, которые превысили количество попыток входа по SSH (количество устанавливается параметром maxretry)
- Откроем файл для редактирования
```bash
sudo nano /etc/fail2ban/jail.conf
```
- Внесем изменения
```bash
[sshd]
#mode   = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
maxretry = 4
bantime = 1d
logencoding = UTF-8
filter = sshd
```
- Сохраняем файл CTRL+C ENTER CTRL+X
- Включим автозагрузку, запустим и проверим работу
```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```
![fail2ban](https://github.com/user-attachments/assets/57899ed3-4f7c-4256-9fb2-4eeec5a1b10d)

### Установка docker
```bash
sudo bash <(wget -qO- https://get.docker.com) @ -o get-docker.sh
```
Если вы под рутом, то лучше делать без sudo, может не найти директории.
Проверяем:
```bash
docker ps -a
```
Если в выводе есть пустая таблица докера - победа.
### Установка Marzban
- Запустим установочный скрипт
```bash
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
```
- Создадим суперпользователя для администрирования системы
```bash
marzban cli admin create --sudo
```
### Настройка SSL для Marzban
Здесь используем домен fallback, проверить привязку можно на dnschecker
- Скачаем утилиту для сертификатов и введем свой EMAIL
```bash
sudo curl https://get.acme.sh | sh -s email=ВАШ_EMAIL
```
- Создадим директорию для сертификатов
```bash
sudo mkdir -p /var/lib/marzban/certs/
```
- Получаем сертификаты (Вместо example.ru вводим свой домен)
```bash
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt --issue --standalone \
-d example.ru \
--key-file /var/lib/marzban/certs/key.pem \
--fullchain-file /var/lib/marzban/certs/fullchain.pem
```
- Проверяем сертификаты
```bash
~/.acme.sh/acme.sh --list
```
- Вносим данные по сертификатам в конфиг:
```bash
sudo nano /opt/marzban/.env
```
- Внести эти данные в файл .env
```bash
UVICORN_PORT = 8000
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/fullchain.pem"
UVICORN_SSL_KEYFILE = "/var/lib/marzban/certs/key.pem"
XRAY_SUBSCRIPTION_URL_PREFIX = https://example.ru
```
- Сохраняем файл CTRL+C ENTER CTRL+X
### Переключение marzban на dev
Как минимум для наличия русского языка и корректной работы телеграм бота
- Редактируем файл докера
```bash
sudo nano /opt/marzban/docker-compose.yml
```
- Заменяем marzban:latest на marzban:dev
- Сохраняем файл CTRL+C ENTER CTRL+X
- Обновляем Marzban для подтягивания новой версии
```bash
marzban update 
```
Теперь можно попасть на марзбан по ссылке https://твой_домен.com:8000/dashboard
![marzban](https://github.com/user-attachments/assets/4a7e3d9b-508f-4576-80ad-7493d3a0ce08)
### Поднимаем nginx для fallback и маскировки
Ставим зависимости
```bash
apt install nginx libnginx-mod-stream -y
```
Проверяем установку модуля stram
```bash
nano /etc/nginx/modules-enabled/50-mod-stream.conf
```
Редачим конфигурацию
```bash
nano /etc/nginx/nginx.conf
```

```nginx
stream {
    map $ssl_preread_server_name $backend {
        www.example.ru reality;
        default web;
    }

    upstream reality {
        server 127.0.0.1:1443;
    }

    upstream web {
        server 127.0.0.1:8443;
    }

    server {
        listen 443;
        proxy_pass $backend;
        ssl_preread on;
    }
}

```
Создаем сайт
```bash
nano /etc/nginx/sites-available/skufee
```

```nginx
server {
    listen 127.0.0.1:8443 ssl http2;
    server_name _;

    ssl_certificate /var/lib/marzban/certs/fullchain.pem;
    ssl_certificate_key /var/lib/marzban/certs/key.pem;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

```
Делаем заглушку
```bash
echo "Hello, world" > /var/www/html/index.html
```
Применяем изменения и запускаем
```bash
ln -s /etc/nginx/sites-available/skufee /etc/nginx/sites-enabled/
nginx -t
systemctl enable nginx --now
systemctl status nginx
```
Проверяем
```bash
curl -v https://example.ru
curl -v https://www.example.ru
```
Должно возвращать по сути одно и тоже и получать хендшейки
### Настройка протокола VLESS
Заходим в настройки и заменяем конфиг следующим образом (добавляется секция VLESS TCP REALITY, можно из старого все убрать и скопировать отсюда):
```json
{
  "log": {
    "loglevel": "warning"
  },
  "routing": {
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "BLOCK",
        "type": "field"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "VLESS TCP REALITY",
      "listen": "127.0.0.1",
      "port": 1443,
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "tcpSettings": {},
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "127.0.0.1:8443",
          "xver": 0,
          "serverNames": [
            "www.example.ru"
          ],
          "privateKey": "ключ",
          "shortIds": [
            "айди"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    },
    {
      "tag": "Shadowsocks TCP",
      "listen": "0.0.0.0",
      "port": 1080,
      "protocol": "shadowsocks",
      "settings": {
        "clients": [],
        "network": "tcp,udp"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "DIRECT"
    },
    {
      "protocol": "blackhole",
      "tag": "BLOCK"
    }
  ]
}
```
Для того, чтобы сгенерировать ключ (параметр `privateKey`) нужно воспользоваться командой
```bash
docker exec marzban_marzban_1 xray x25519
```
Контейнер Marzban'a может создаться с именем marzban-marzban-1, в таком случае команда будет:
```bash
docker exec marzban-marzban-1 xray x25519
```
Отсюда нас интересует значение, собственно, PrivateKey. Копируем его и вставляем в конфиг. Далее генерируем айди параметр `shortIds`
```bash
openssl rand -hex 8
```
Также копируем его и вставляем в параметр `shortIds`
![vless](https://github.com/user-attachments/assets/5721705c-b4fd-4c53-81d6-f7f9c52bc85f)

После этого нажать сохранить и перезагрузить ядро и перезагрузить страницу.
Теперь можно добавить пользователя, выбрать протокол VLESS, указать имя пользователя, развернуть на три точки настройки и указать Flow xtls-rprx-vision.
После создания, в основном меню можно будет скопировать ссылку на подписку и ссылку конфигурации (типа vless//).

### Добавление телеграм бота
- Создаем телеграм бота в @BotFather (при создании будет сообщение с АПИ)
- Получаем свой айди в @userinfobot
- Редактируем конфигурационный файл
```bash
sudo nano /opt/marzban/.env
```
- Раскомментируем TELEGRAM_API_TOKEN и TELEGRAM_ADMIN_ID
- Копируем API из сообщения о создании бота и свой айди
- Сохраняем файл CTRL+C ENTER CTRL+X
- Перезапускаем Marzban
```bash
marzban restart
```
- Открываем бота (по ссылке из сообщения ботфазера) и запускаем.
Теперь можно создавать пользователей и мониторить их через телеграм бота. Также отсюда можно сразу копировать ссылки для подключений.

## Установка WireGuard с веб-интерфейсом wg-easy.
- Создаем директорию в удобном виде
```bash
sudo mkdir /opt/wg-easy
```
- Создаем внутри файл docker-compose.yml
```bash
sudo nano /opt/wg-easy/docker-compose.yml
```
- Вставляем следующий код:
```yml
volumes:
  etc_wireguard:

services:
  wg-easy:
    environment:
      # Change Language:
      # (Supports: en, ua, ru, tr, no, pl, fr, de, ca, es, ko, vi, nl, is, pt, chs, cht, it, th, hi, ja, si)
      - LANG=ru
      # ⚠️ Required:
      # Change this to your host's public address
      - WG_HOST=raspberrypi.local # ваш айпи или ваше доменное имя

      # Optional:
      - PASSWORD_HASH=$$2y$$10$$hBCoykrB95WSzuV4fafBzOHWKu9sbyVa34GJr8VV5R/pIelfEMYyG # Хэш вашего пароля, генерацию рассмотрим чуть позже
      # - PORT=51821
      # - WG_PORT=51820
      # - WG_CONFIG_PORT=92820
      # - WG_DEFAULT_ADDRESS=10.8.0.x
      # - WG_DEFAULT_DNS=1.1.1.1
      # - WG_MTU=1420
      # - WG_ALLOWED_IPS=192.168.15.0/24, 10.0.1.0/24
      # - WG_PERSISTENT_KEEPALIVE=25
      # - WG_PRE_UP=echo "Pre Up" > /etc/wireguard/pre-up.txt
      # - WG_POST_UP=echo "Post Up" > /etc/wireguard/post-up.txt
      # - WG_PRE_DOWN=echo "Pre Down" > /etc/wireguard/pre-down.txt
      # - WG_POST_DOWN=echo "Post Down" > /etc/wireguard/post-down.txt
      # - UI_TRAFFIC_STATS=true
      # - UI_CHART_TYPE=0 # (0 Charts disabled, 1 # Line chart, 2 # Area chart, 3 # Bar chart)
      # - WG_ENABLE_ONE_TIME_LINKS=true
      # - UI_ENABLE_SORT_CLIENTS=true
      # - WG_ENABLE_EXPIRES_TIME=true
      # - ENABLE_PROMETHEUS_METRICS=false
      # - PROMETHEUS_METRICS_PASSWORD=$$2a$$12$$vkvKpeEAHD78gasyawIod.1leBMKg8sBwKW.pQyNsq78bXV3INf2G # (needs double $$, hash of 'prometheus_password'; see "How_to_generate_an_bcrypt_hash.md" for generate the hash)

    image: ghcr.io/wg-easy/wg-easy
    container_name: wg-easy
    volumes:
      - etc_wireguard:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
      # - NET_RAW # ⚠️ Uncomment if using Podman
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```
Обратите внимание на параметры WG_HOST и PASSWORD_HASH. WG_HOST мы меняем на свой IP адрес или доменное имя, а PASSWORD_HASH мы сгенерируем чуть позже. Он почему-то не генерируется, если контейнер не запущен.
- Сохраняем файл CTRL+C ENTER CTRL+X
- Запускаем наш контейнер
```bash
docker compose up --detach
```
- Теперь пришло время генерации хэша. Для этого используем команду (заменив YOUR_PASSWORD на ваш пароль):
```bash
docker run --rm -it ghcr.io/wg-easy/wg-easy wgpw 'YOUR_PASSWORD'
```
- В результате вы получите нечто вот такое:
```bash
PASSWORD_HASH='$2b$12$coPqCsPtcFO.Ab99xylBNOW4.Iu7OOA2/ZIboHN6/oyxca3MWo7fW'
```
- Копируем все что в кавычках и открываем снова на редактирование docker-compose.yml
```bash
sudo nano /opt/wg-easy/docker-compose.yml
```
- Обязательно после знака = у нас должен остаться один $ (то есть суммарно 2) и после него копируем наш хэш:
```bash
- PASSWORD_HASH=$$2y$$10$$hBCoykrB95WSzuV4fafBzOHWKu9sbyVa34GJr8VV5R/pIelfEMYyG
```
- Сохраняем файл CTRL+C ENTER CTRL+X
- И теперь снова запускаем контейнер
```bash
docker compose up --detach
```
- Проверяем корректность запуска (здесь же вы увидите запущенный Marzban)
```bash
docker ps -a
```
![docker](https://github.com/user-attachments/assets/fba9fda3-1876-47d9-8aa7-c414fcfcf20e)
Теперь у нас доступен веб-интерфейс WireGuard по адресу http://ваш_айпи_или_домен:51821
Заходим, добавляем клиента, скачиваем конфиг и радуемся жизни.
