# CV — портфолио Junior Linux SysAdmin

Статичная одностраничка (`public/index.html`, инлайн CSS/JS, ноль внешних запросов).
Отдаётся контейнером nginx за Traefik, обновляется из этого репозитория.

## Структура

```
.
├── public/
│   └── index.html      # единственный публично отдаваемый файл
├── docker-compose.yml  # деплой (nginx монтирует только public/, в вебрут не попадает)
└── README.md
```

## Деплой на сервере

```bash
git clone https://github.com/lemonke68/cv.git ~/cv
cd ~/cv && docker compose up -d
```

DNS-запись поддомена должна указывать на сервер. Если при первом старте Traefik отдал
self-signed cert (домен не успел зарезолвиться к ACME-challenge) — перезапустить прокси-контейнер.

Проверка:
```bash
curl -sk -o /dev/null -w "%{http_code}\n" -H "Host: cv.mango-kokos.ru" https://127.0.0.1/
# ожидаем 200
```

## Авто-обновление (pull-модель)

Правка на ноуте → `git push` → сервер сам подтягивает и страница обновляется.
Сервер ходит на GitHub сам, поэтому не нужно ни webhook'ов, ни входящих портов.
Монтируется папка `public/`, nginx отдаёт статику свежей на каждый запрос —
после `git pull` перезапуск контейнера не требуется.

`systemd`-таймер на сервере (`cv-pull.service` + `cv-pull.timer`):

```ini
# /etc/systemd/system/cv-pull.service
[Unit]
Description=Pull cv repo
After=network-online.target

[Service]
Type=oneshot
WorkingDirectory=%h/cv
ExecStart=/usr/bin/git pull --ff-only
```

```ini
# /etc/systemd/system/cv-pull.timer
[Unit]
Description=Pull cv repo every 3 min

[Timer]
OnBootSec=2min
OnUnitActiveSec=3min

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now cv-pull.timer
```

## Правки контента

- **Контакты** — заменить плейсхолдеры в шапке `public/index.html` (блок `<!-- TODO -->`).
- Поддомен меняется в `docker-compose.yml` (лейблы роутера).

## Примечания

- Схема отражает реальную топологию, вычищена от IP/путей/внутренних имён.
- k3s намеренно не показан как рабочий слой (был откатан на Docker Compose) — это в разделе кейсов.
- Секретов в репозитории нет. Но это конфиг для личного сервера, поэтому репозиторий
  лучше держать приватным (тогда клон на сервере — через read-only deploy key).
