# Traefik 3.5 Reverse Proxy с SSL

Полная настройка Traefik v3.5 как обратного прокси для множества Docker контейнеров с автоматическим получением SSL сертификатов от Let's Encrypt.

## 📋 Структура проекта

```
proxy/
├── docker-compose.yml          # Основной compose файл Traefik
├── config/
│   └── traefik.yml            # Статическая конфигурация
├── letsencrypt/
│   └── acme.json              # Хранилище SSL сертификатов
└── example-site/              # Пример сайта
    ├── docker-compose.yml     # Compose для примера
    └── html/
        └── index.html
```

## 🚀 Быстрый старт

### 1. Настройка Traefik

## Traefik reverse-proxy — универсальное руководство

Этот репозиторий содержит конфигурацию Traefik (Docker + docker-compose) для использования как обратного прокси и автоматического получения TLS-сертификатов. README сделан максимально универсальным — с подсказками, какие поля и файлы менять, чтобы развернуть на любом сервере.

Цель:
- Быстрая настройка Traefik на любом Linux-сервере с Docker
- Простая инструкция по добавлению сайтов (Docker Compose)
- Подсказки для production (Let's Encrypt, firewall, бэкапы)

Файлы, на которые стоит обратить внимание:
- `docker-compose.yml` — главный Compose для Traefik
- `config/traefik.yml` — статическая конфигурация Traefik (entrypoints, providers и т.д.)
- `letsencrypt/` или `acme/` — место для хранения ACME данных (`acme.json`)

## Короткая «контракт»-инструкция (inputs / outputs)
- Вход: домены (A/AAAA), корректные volume-пути, email для ACME
- Выход: Traefik, предоставляющий HTTPS для сервисов, подключённых к Docker
- Ошибки/режимы: staging vs production ACME, блокирующий firewall, неверные labels

## 1 — Общие требования
- Docker Engine (версия >= 20.10) и docker-compose v2 (или встроенный `docker compose`)
- Доступ к портам 80 и 443 на сервере (проброшенные через firewall/облако)
- Для production: валидные DNS A/AAAA записи для всех доменов

Примечание по SELinux/AppArmor: при включённом SELinux могут потребоваться дополнительные разрешения для томов; в этом случае используйте relabel или разместите тома в разрешённых местах.

## 2 — Переменные и настройки, которые нужно адаптировать
- LETSENCRYPT_EMAIL — ваш email для Let's Encrypt
- TRAEFIK_NETWORK — имя внешней сети Docker (по умолчанию `traefik`)
- VOLUMES пути: проверьте, что пути к `letsencrypt`/`acme.json` и `config/` доступны и имеют корректные права
- Dashboard: если вы открываете dashboard в интернет — обязательно настройте BasicAuth и ограничьте доступ по IP

Пример минимальных ENV в `docker-compose.yml` (шаблон):

```env
LETSENCRYPT_EMAIL=you@example.com
TRAEFIK_NETWORK=traefik
```

## 3 — Как использовать на другом сервере (шаги)
1) Скопируйте репозиторий или файлы на сервер.
2) Убедитесь, что `docker` и `docker compose` установлены.
3) Отредактируйте переменные: email, пути томов, имя сети.
4) Создайте внешнюю сеть, если её нет:

```bash
docker network create traefik
```

5) Запустите Traefik:

```bash
docker compose up -d
```

6) Проверьте логи и статус:

```bash
docker compose ps
docker compose logs -f traefik
```

Если вы используете провайдер облака (DigitalOcean, AWS, GCP) — откройте 80/443 в соответствующей панели.

## 4 — Как добавить сайт (универсально)
Подключите сервис к внешней сети Traefik и добавьте метки (labels).

Обязательные минимальные labels:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.SERVICE.rule=Host(`example.com`)"
  - "traefik.http.routers.SERVICE.entrypoints=websecure"
  - "traefik.http.routers.SERVICE.tls=true"
  - "traefik.http.routers.SERVICE.tls.certresolver=letsencrypt"
  - "traefik.http.services.SERVICE.loadbalancer.server.port=80" # порт внутри контейнера
```

Заменяйте `SERVICE` на уникальное имя. Для нескольких доменов используйте `Host(`a.com`) || Host(`b.com`)`.

## 5 — TLS / Let's Encrypt
- Для тестирования используйте staging ACME (в `traefik.yml`): он не даёт rate-limit проблем.
- Для production удалите staging CA и используйте `https://acme-v02.api.letsencrypt.org/directory`.
- Убедитесь, что `acme.json` защищён (chmod 600).

Пример: дать права и создать файл:

```bash
mkdir -p ./letsencrypt
touch ./letsencrypt/acme.json
chmod 600 ./letsencrypt/acme.json
```

## 6 — Локальная разработка и тестирование
- Для локального тестирования можно добавить записи в `hosts` на вашей машине:

```
YOUR_SERVER_IP example.com
```

Note: Let's Encrypt не будет работать для `localhost` или IP; используйте staging или mkcert для локальных тестов.

## 7 — Production чеклист (универсальный)
- Настроить и проверить DNS
- Проверить открытые порты 80/443
- Установить правильный email для ACME
- Резервная копия `acme.json`
- Настроить резервный доступ к dashboard (BasicAuth + IP allowlist)

## 8 — Частые проблемы и советы
- ACME не получает сертификат: проверьте DNS, доступность порта 80 и отсутствие прокси перед сервером.
- Контейнер не доступен: убедитесь, что сервис присоединён к сети `traefik` и label с правильным портом.
- Too many redirects: проверьте редиректы на уровне приложения и middlewares в Traefik.

## 9 — Примеры (коротко)

- Добавление nginx сайта:

```yaml
services:
  web:
    image: nginx:alpine
    networks:
      - traefik
    volumes:
      - ./html:/usr/share/nginx/html:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`example.com`)"
      - "traefik.http.routers.web.entrypoints=websecure"
      - "traefik.http.routers.web.tls=true"
      - "traefik.http.routers.web.tls.certresolver=letsencrypt"
      - "traefik.http.services.web.loadbalancer.server.port=80"

networks:
  traefik:
    external: true
```

## 10 — Как адаптировать для разных серверов
- Публичный VPS: убедитесь, что провайдер не блокирует 80/443 (иногда нужно опустить cloud firewall).
- Сервер в облаке: настройте security group/Network ACL для 80/443.
- Локальная машина или NAT: пробросьте порты с роутера или используйте tunneling (ngrok, cloudflared) для тестирования.

## 11 — Валидация изменений
- После редактирования `docker-compose.yml` и `config/traefik.yml` всегда перезапускайте Traefik и смотрите логи:

```bash
docker compose up -d
docker compose logs -f traefik
```

## 12 — Доп. ресурсы
- Traefik docs: https://doc.traefik.io/traefik/
- Docker provider: https://doc.traefik.io/traefik/providers/docker/

---

Если хотите, могу дополнительно:

Готов продолжать — скажите, какие дополнения нужны.

## Пример systemd unit (готовый файл)
Ниже — простой systemd unit для управления Traefik через `docker compose`. Файл добавлён в репозиторий как `systemd/traefik.service`.

Перед использованием:
- Проверьте путь в `WorkingDirectory` (по умолчанию `/home/debian/proxy`) и при необходимости измените.
- Убедитесь, что путь к `docker` и `docker compose` совпадает с системой (`/usr/bin/docker` обычно корректно).

Содержимое примера `systemd/traefik.service`:

```
[Unit]
Description=Traefik (Docker Compose)
Documentation=https://doc.traefik.io/traefik/
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/debian/proxy
Environment=COMPOSE_PROJECT_NAME=traefik
ExecStart=/usr/bin/docker compose -f docker-compose.yml up -d
ExecStop=/usr/bin/docker compose -f docker-compose.yml down --remove-orphans
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Как включить и запустить:

```bash
# скопируйте или отредактируйте unit в /etc/systemd/system/traefik.service
sudo cp systemd/traefik.service /etc/systemd/system/traefik.service

# перечитать конфигурацию systemd
sudo systemctl daemon-reload

# включить при загрузке и запустить
sudo systemctl enable --now traefik.service

# посмотреть статус и логи
sudo systemctl status traefik.service
sudo journalctl -u traefik.service -f
```

Советы:
- Если вы часто редактируете `docker-compose.yml`, после изменений используйте `sudo systemctl restart traefik.service`.
- Для безопасного использования в production: убедитесь, что `acme.json` и конфиги имеют правильные права (например, chmod 600) и что доступ к dashboard ограничен.


