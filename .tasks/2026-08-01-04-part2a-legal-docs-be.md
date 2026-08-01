## MURU Executor — [ID: 2026-08-01-04]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** продолжать `feat/settings-site-contacts` (1A+1B на ней) **или** новая `feat/settings-legal-docs` от того же tip с 1A. НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **2A Backend**. STOP-1 (Часть 1) **ACCEPT** оркестратором.

### Цель
Разрешить CRM upsert/get для юридических slug'ов через отдельный `LEGAL_DOC_SLUGS`, общий код с `upsertFixedPage`. Миграций не нужно (`content_pages` уже есть).

### Зафиксированное решение
- `LEGAL_DOC_SLUGS = ['privacy','offer','delivery','refund','terms','consent'] as const`
- **Не** тащить `requisites` в legal (Часть 3 = site_settings)
- `FIXED_PAGE_SLUGS` (help/contacts/company/vacancy/partners) оставить как есть для контентных страниц
- Allowlist для `assertFixedPageSlug` / get+upsert by-slug = **union** FIXED ∪ LEGAL (можно переименовать хелпер в `assertEditablePageSlug`, но публичный API функции `upsertFixedPage` сохранить)
- Дефолтные заголовки для legal (RU):
  - privacy → «Политика обработки персональных данных»
  - offer → «Публичная оферта»
  - delivery → «Доставка»
  - refund → «Возврат»
  - terms → «Условия обслуживания»
  - consent → «Согласие на обработку персональных данных»

### Файлы
- `backend/src/services/content.service.ts` — LEGAL_DOC_SLUGS, titles, assert allowlist
- `backend/src/services/content.service.test.ts` — сейчас `upsertFixedPage('privacy')` **отклоняется** → должен проходить; добавить кейсы для 1–2 новых slug'ов; невидимая страница → public 404 (если ещё нет — добавить)
- `backend/src/routes/content.routes.test.ts` — сейчас CRM PUT `privacy` ожидает **400** → обновить на 200/успех; invalid slug вне обоих списков по-прежнему 400
- Admin / SF — **не** в этом промпте (2B/2C)

### НЕ трогать
- `site_settings`, CDEK/YK, payments
- Спец-апы company/vacancy/partners (sections)
- Drive-скрипты
- Публичный `getPublicPageBySlug` — уже slug-agnostic; только убедиться, что sanitize на upsert сохраняется

### Критерии готовности
- [ ] `upsertFixedPage('privacy'|…|'consent')` OK; неизвестный slug → 400
- [ ] `getCrmPageBySlug` для legal slug после upsert OK
- [ ] Public get: visible → 200; `is_visible=false` → 404
- [ ] HTML sanitize на body при upsert (существующий путь)
- [ ] `tsc --noEmit` + vitest content.service + content.routes зелёные
- [ ] Тест «rejects privacy» заменён / инвертирован осознанно

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/backend"
npx tsc --noEmit
npx vitest run src/services/content.service.test.ts src/routes/content.routes.test.ts
```

### Отчёт оркестратору
- ветка, файлы, какие тесты поменялись (было 400 на privacy → стало OK)
- tsc/vitest
- подтверждение: requisites не в LEGAL_DOC_SLUGS
