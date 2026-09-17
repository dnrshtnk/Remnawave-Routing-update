# Remnawave Routing Updater

Безопасно синхронизирует deeplink из GitHub с Remnawave. Основной режим
обновляет заголовок `routing` только внутри выбранного Response Rule и не
перезаписывает остальные правила или заголовки.

## Защитные проверки

Перед каждым PATCH сервис:

1. загружает deeplink и декодирует Base64/JSON;
2. проверяет обязательные поля и HTTPS-хосты геобаз;
3. проверяет доступность `geoip.dat` и `geosite.dat`;
4. повторно читает актуальные настройки Remnawave;
5. требует увеличения `LastUpdated` и разрешает только явно заданные пары
   переименования профиля;
6. сохраняет полную резервную копию текущих настроек;
7. меняет только заголовок `routing`;
8. повторно читает настройки и проверяет результат.

По умолчанию выключен `DRY_RUN=false`: сервис только показывает планируемое
изменение.

## Установка

```bash
git clone https://github.com/dnrshtnk/Remnawave-Routing-update.git
cd Remnawave-Routing-update
cp .env.example .env
nano .env
mkdir -p backups
docker compose up -d --build
docker compose logs -f routing-updater
```

Пример для локальной панели Remnawave:

```env
REMNA_BASE_URL=http://remnawave:3000/api
REMNA_TOKEN=replace_with_api_token

# Настройки для правила Happ
RULE_1_NAME=Happ
RULE_1_URL=https://raw.githubusercontent.com/hydraponique/roscomvpn-routing/refs/heads/main/HAPP/DEFAULT.DEEPLINK

# Настройки для правила Incy
RULE_2_NAME=Incy
RULE_2_URL=https://raw.githubusercontent.com/hydraponique/roscomvpn-routing/refs/heads/main/INCY/DEFAULT.DEEPLINK


DRY_RUN=false
VALIDATE_GEO_URLS=true
ALLOW_PROFILE_RENAME=false
ALLOWED_PROFILE_RENAMES=RoscomVPN:swiftless-routing
CRON_SCHEDULE=30 4 * * *
TZ=UTC
```

Response Rule:

``` Remnawave - Response Rule
{
  "version": "1",
  "rules": [
    {
      "name": "Happ",
      "description": "Happ",
      "enabled": true,
      "operator": "AND",
      "conditions": [
        {
          "headerName": "user-agent",
          "operator": "STARTS_WITH",
          "value": "happ/",
          "caseSensitive": false
        }
      ],
      "responseType": "XRAY_JSON"
    },
    {
      "name": "Incy",
      "description": "Incy",
      "enabled": true,
      "operator": "AND",
      "conditions": [
        {
          "headerName": "user-agent",
          "operator": "STARTS_WITH",
          "value": "incy/",
          "caseSensitive": false
        }
      ],
      "responseType": "XRAY_JSON"
    },
    {
      "name": "Block others",
      "description": "Block others requests.",
      "enabled": false,
      "operator": "AND",
      "conditions": [],
      "responseType": "BLOCK"
    }
  ]
}
```

Контейнер подключается к существующей сети `remnawave-network`. Если панель
имеет другое имя контейнера или порт, измените `REMNA_BASE_URL`.

Перезапустить только updater:

   ```bash
   docker compose up -d --build routing-updater
   docker compose logs --tail=100 routing-updater
   ```

Production deeplink использует `happ://routing/onadd/`, чтобы профиль с новым
именем `swiftless-routing` стал активным после успешной загрузки геобаз. Общий
переключатель `ALLOW_PROFILE_RENAME=true` для этой миграции не нужен.

## Переменные окружения

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `REMNA_BASE_URL` | — | URL API, например `http://remnawave:3000/api` |
| `REMNA_TOKEN` | — | Bearer-токен Remnawave |
| `GITHUB_RAW_URL` | production deeplink | Источник профиля |
| `UPDATE_TARGET` | `response-rule` | `response-rule` либо совместимый режим `global` |
| `RESPONSE_RULE_NAME` | `Happ` | Точное имя изменяемого Response Rule |
| `DRY_RUN` | `true` | Запретить фактический PATCH |
| `VALIDATE_GEO_URLS` | `true` | Проверять обе базы перед обновлением |
| `ALLOW_PROFILE_RENAME` | `false` | Разрешить изменение поля `Name` |
| `ALLOWED_PROFILE_RENAMES` | пусто | Разрешённые точные пары `старое:новое`, через запятую |
| `ALLOWED_GEO_HOSTS` | jsDelivr, GitHub Raw, GitHub | Разрешённые хосты баз |
| `CRON_SCHEDULE` | пусто | Cron вместо интервального опроса |
| `CHECK_INTERVAL` | `21600` | Интервал без cron, минимум 60 секунд |
| `REMNA_SSL_VERIFY` | `true` | Проверка TLS внешнего API Remnawave |
| `BACKUP_DIR` | `/data/backups` | Каталог резервных копий |
| `REQUEST_TIMEOUT` | `30` | HTTP timeout в секундах |
| `SQUAD_N_UUID`, `SQUAD_N_URL` | пусто | Необязательные внешние сквады |


## Лицензия

MIT
