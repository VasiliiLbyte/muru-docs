## MURU Executor — [ID: 2026-08-01-10]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `feat/settings-site-contacts` (или `feat/settings-cdek-yookassa` от tip с 1–3). НЕ merge в master. НЕ деплоить на прод.  
**Связь:** EPIC Settings → Часть **4A Backend**. STOP-3 **ACCEPT**. Staging-first для деплоя — после кода, отдельным гейтом Vasilii.

### Цель
Несекретные CDEK/YooKassa параметры в `site_settings` + runtime-resolver `DB ?? env`. Секреты **только** из env, никогда в API-ответах.

### Секреты — НЕ в БД, НЕ в ответах
`CDEK_CLIENT_ID/SECRET/WEBHOOK_SECRET`, `YOOKASSA_SECRET_KEY`, `YOOKASSA_WEB_SECRET_KEY`, `YOOKASSA_RETURN_URL`, `YOOKASSA_WEB_RETURN_URL`.  
Shop IDs — **не секрет**, но **не писать в БД**: read-only из env в `integrations-status`.

### Миграция `036_settings_cdek_yookassa.sql` (+down)
ALTER `site_settings` ADD (все nullable кроме где логично default):
- CDEK: `cdek_env TEXT` CHECK IN ('test','production') или null, `cdek_sender_city_code INT`, `cdek_sender_postal_code TEXT`, `cdek_sender_address TEXT`, `cdek_sender_name TEXT`, `cdek_sender_phone TEXT`, `cdek_tariff_door INT`, `cdek_tariff_pvz INT`, `cdek_default_weight_grams INT`, `cdek_default_length_cm INT`, `cdek_default_width_cm INT`, `cdek_default_height_cm INT`
- YooKassa: `yookassa_vat_code INT`, `yookassa_verify_ip BOOLEAN`
- Обновить `schema.sql`

### Сервис / резолвер
1. Расширить `site-settings.service.ts`: `updateCdekSettings`, `updateYookassaSettings` (zod `.strict()`, UPDATE только своей группы) + тесты.
2. Новый `runtime-config.service.ts` (или рядом):
   - `getEffectiveCdekBusinessConfig()` / packaging dims / yookassa vat+verifyIp
   - Правило: **непустое DB-значение ?? env/constants**
   - Секреты (`clientId/secret/webhookSecret`, YK secrets, returnUrls) — **всегда** `env.*`
   - Тест: DB overrides env; secrets never from DB object
3. **Call sites (минимально):**
   - `cdek/calc.service.ts` — senderCityCode, tariffs, dims defaults
   - `cdek/orders.service.ts` — **убрать module-level `normalizedSenderPhone`**; читать phone через resolver на вызов; sender address/name/postal/city/tariffs
   - `cdek/packaging.service.ts` — default dims/weight из resolver
   - `pricing.service.ts` — allowedTariffCodes из resolver
   - `yookassa/receipt.ts` (или где vatCode) — vat + при необходимости verifyIp consumers
   - Не мутировать `env`; startup guards в `env.ts` оставить
4. При `updateCdekSettings`: если `cdek_env === 'production'` и в env нет clientId/secret — **не** `process.exit`; вернуть `HttpError` 422 с русским текстом «Для режима production нужны серверные ключи CDEK (CDEK_CLIENT_ID/SECRET в env)».

### CRM routes (owner, расширить crm-settings)
- `PUT /site/cdek`, `PUT /site/yookassa`
- `GET /integrations-status` → только булевы + read-only shopIds:
  ```
  {
    cdekConfigured: boolean,      // clientId+secret non-empty
    cdekWebhookConfigured: boolean,
    yookassaConfigured: boolean,  // shop+secret
    yookassaWebConfigured: boolean,
    yookassaShopId: string | null,     // non-secret display
    yookassaWebShopId: string | null
  }
  ```
  **Никогда** не включать secretKey/clientSecret/webhookSecret/returnUrl values.

### Тесты (обязательно)
- Resolver priority DB > env; secrets только env
- PUT cdek production without secrets → 422
- Serialisation guard: JSON ответов settings/integrations **не содержит** известных secret env values (если заданы в test mock — assert not.toContain)
- 401/403 на новых PUT

### НЕ трогать
- Admin SPA (4B отдельно), SF, Mini-App admin
- Перенос секретов в БД
- Drive-скрипты

### DoD
- 036 up/down локально OK
- tsc + vitest зелёные (затронутые + regression smoke)
- Явно в отчёте: «секреты не сериализуются»

### Проверка
```bash
cd backend
# psql -f 036 … down … up
npx tsc --noEmit
npx vitest run src/services/site-settings.service.test.ts src/services/runtime-config.service.test.ts src/routes/crm-settings.routes.test.ts
# плюс точечно calc/orders тесты если есть
```

### Отчёт оркестратору
ветка, файлы, 036 up/down, список call-sites, тесты, подтверждение no-secrets, риски для staging smoke

**Модель:** сильная (payments/delivery). Plan mode обязателен.
