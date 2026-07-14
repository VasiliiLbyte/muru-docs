# CUTOVER: Google Sheets → CRM (владелец каталога)

**Статус:** подготовлен 2026-07-13. Прод: `master @ 1392cb7` (DEP-017 deployed). Cutover — по окну.
**Решение:** каталог переводится на CRM как единственный источник записи. Sheets после
переключения — **read-only навсегда** (жёсткая заморозка, дубль-записи нет — см.
PROGRESS.md, архитектурное решение фазы D).
**Даунтайм не нужен:** чтение каналов (mini app, витрина) идёт из PostgreSQL и не меняется.
Флип `CATALOG_SOURCE` меняет только владельца записи. Откат = вернуть флаг.
**Триггер боевой валидации:** заказчик добавляет ~30 новых товаров — вносятся уже через CRM.

**Env на проде (важно):** PM2-процесс стартует с `cwd=/var/www/muru`; `backend/src/utils/env.ts`
берёт **первый** существующий файл из `envCandidatePaths` → **`/var/www/muru/.env`**.
Файл `/var/www/muru/backend/.env` процессом **не читается**. После деплоя проверять
`pm2 logs`: строка `[env] loaded: /var/www/muru/.env`.

---

## 1. Пре-чеклист (до назначения окна)

| # | Пункт | Статус |
|---|---|---|
| 1 | Миграция 019 (`subcategory`, `subcategory_slug`) в git и применена на prod | ✅ 2026-07-13 (DEP-016) |
| 2 | nginx `location /img/` на `murushop.ru` включён и проверен ещё в sheets-режиме (200, кэш) | ✅ 2026-07-13 (DEP-016) |
| 3 | Прод `.env`: `CDEK_SENDER_ADDRESS` — проверить кавычки (TD-003) | ☐ |
| 4 | Прод `.env`: нет дубликатов `CATALOG_SOURCE` (иначе выигрывает неожиданная строка) | ☐ |
| 5 | Staging `/var/www/muru-staging` синхронизирован с `master @ 0877d6d` (TD-002) | ☐ |
| 6 | Договорённость с клиентом о freeze таблицы: дата/время; после — Sheets только чтение | ☐ |
| 7 | Бэкап прод-БД непосредственно перед окном (`pg_dump muru_db`) | ☐ в окне |
| 8 | Два `.env` на VPS сведены: канон — `/var/www/muru/.env`; `backend/.env` → `.unused` | ☐ |

---

## 2. Окно переключения (~1–2 ч, вечер)

Все команды — на VPS, Василий по этому ранбуку. Прод не останавливается.

### 2.0. Режим обслуживания mini app (до флипа)
В **`/var/www/muru/.env`** (корневой, см. §2.4):
```
MINIAPP_MAINTENANCE=true
```
- Покупатели видят заглушку «Магазин скоро откроется».
- Админы из `ADMIN_TELEGRAM_IDS` проходят (каталог, TG-админка, sync).
- Снять флаг — после зелёного смоука §3 (`MINIAPP_MAINTENANCE=false` + `pm2 reload --update-env`),
  либо оставить до конца боевой валидации §4 — по решению Василия.

### 2.1. Финальный sync из Sheets
- В TG-админке запустить ручной sync каталога, дождаться успеха.
- Убедиться: `catalog_sync_log` — последняя запись success, количество товаров ожидаемое.

### 2.2. Выключить sync-расписание
- В TG-админке: Настройки sync → выключить расписание (enabled=false).
- Подстраховка кодом: в `crm`-режиме `runSyncSchedulerTick` сам скипает
  (`[sync-scheduler] skipped: catalog source is crm`) — но флаг выключаем явно.

### 2.3. Бэкап БД
```bash
pg_dump -Fc muru_db > /root/backups/muru_db_pre_cutover_$(date +%F_%H%M).dump
```

### 2.4. Консолидация env + флип флагов
**Канонический файл env — `/var/www/muru/.env`** (НЕ `backend/.env`).

1. Сравнить оба файла и перенести актуальные значения в корневой:
   ```bash
   diff <(sort /var/www/muru/.env) <(sort /var/www/muru/backend/.env)
   grep -c 'CATALOG_SOURCE' /var/www/muru/.env   # должно быть ровно 1
   ```
2. Убедиться, что в корневом `.env` есть актуальные `TELEGRAM_BOT_TOKEN`,
   `TELEGRAM_PROVIDER_TOKEN`, `YOOKASSA_SHOP_ID`, `YOOKASSA_SECRET_KEY`, `ADMIN_JWT_SECRET`.
3. Переименовать лишний файл (чтобы резолвер не подхватил его при смене cwd):
   ```bash
   mv /var/www/muru/backend/.env /var/www/muru/backend/.env.unused
   ```
4. В **`/var/www/muru/.env`** выставить:
   ```
   CATALOG_SOURCE=crm
   ENABLE_SHEETS_STOCK_WRITE=false
   ```
   Проверить: каждая переменная — одна строка, без дубликатов (п.4 пре-чеклиста).

### 2.5. Рестарт
```bash
cd /var/www/muru && pm2 reload ecosystem.config.js --update-env && pm2 save
pm2 logs muru-backend --lines 50
# Ожидание: [env] loaded: /var/www/muru/.env
#           crm-режим / skipped sync — без env-ошибок
pm2 env $(pm2 id muru-backend | head -1) | grep CATALOG_SOURCE
# Ожидание: CATALOG_SOURCE=crm
```

---

## 3. Смоук после флипа (в том же окне)

| # | Проверка | Ожидание |
|---|---|---|
| 0 | `pm2 logs muru-backend` — строка `[env] loaded:` | указывает на **`/var/www/muru/.env`** |
| 1 | `curl -sS http://127.0.0.1:4000/api/health` | 200 |
| 2 | Mini App (админ): каталог → тестовый заказ до инвойса | OK; **`invoiceUrl` начинается с `https://t.me/$`** |
| 3 | Витрина `web.murushop.ru`: каталог, карточка, корзина | OK |
| 4 | CRM `/admin/catalog`: meta показывает **не** read-only (crm-режим, запись разрешена) | OK |
| 5 | CRM: создать тестовый товар с фото → виден в mini app И на витрине | OK |
| 6 | `/img/<fileId>` для фото нового товара | 200, из кэша при повторе |
| 7 | CRM: архивировать тестовый товар → исчез из обоих каналов | OK |
| 8 | TG-админка: ручной sync недоступен/заблокирован в crm-режиме (423 или скрыт) | OK |
| 9 | Снять `MINIAPP_MAINTENANCE=false` (если был включён в §2.0) + reload; покупатель видит каталог | OK |

Если п.5–7 не проходят — откат (см. §5), разбор на staging.

## 4. Боевая валидация (первые дни)

- Клиент вносит ~30 новых товаров **только через CRM**.
- Ежедневно: выборочная проверка новых товаров в обоих каналах + `/img/` для их фото.
- Sheets никто не редактирует (договорённость п.6 пре-чеклиста); записи в Sheets из
  кода нет (`ENABLE_SHEETS_STOCK_WRITE=false`).

## 5. Откат (окно решения — 48 ч)

Критерии отката: товары из CRM не появляются в каналах; массовые ошибки записи;
битые изображения, не решаемые точечно.

```bash
# 1. Вернуть в /var/www/muru/.env: CATALOG_SOURCE=xlsx  (legacy-алиас sheets), ENABLE_SHEETS_STOCK_WRITE=false
# 2. pm2 reload ecosystem.config.js --update-env
# 3. В TG-админке включить расписание sync обратно
# 4. Товары, созданные через CRM за это время, руками перенести в Sheets ДО первого sync
#    (иначе sync-upsert их не тронет, но и не обновит; сверить по catalog_sync_log)
```

После 48 ч стабильной работы откат считается закрытой опцией; Sheets переводится
в архивный режим (доступ у клиента остаётся, но как справочный).

## 6. Пост-стабилизация

- Чистка категорий-сирот («не используется») через CRM (safe delete из D7 с guard'ами).
- ~~TD-005: герметичность `sync-scheduler.test.ts`~~ ✅ закрыто D8.1 (`1e439bf`).
- Обновить PROGRESS.md (DEP-строка cutover) и DEPLOY.md (§ sync больше не операция прод-каталога).
