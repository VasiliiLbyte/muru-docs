## MURU Executor — [ID: 2026-08-01-05]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `feat/settings-site-contacts` (2A ACCEPT). НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **2B Admin SPA**. Backend allowlist уже: `LEGAL_DOC_SLUGS = privacy|offer|delivery|refund|terms|consent`.

### Цель
Хаб «Юридические документы» + редактирование через существующий `FixedPageEditPage` (tiptap). Новый редактор не писать.

### Контекст BE (уже есть)
- `PUT/GET /api/crm/content/pages/by-slug/:slug` для legal slug'ов
- `requisites` по-прежнему 400 — не добавлять в admin types
- Default title privacy на бэке: **«Политика конфиденциальности»** — в admin `defaultTitle`/`SECTION_META` выровнять с BE (не выдумывать другой)

### Файлы (ожидаемые)
- `admin/src/types/content.ts` — расширить типы:
  - `LegalDocSlug = 'privacy'|'offer'|'delivery'|'refund'|'terms'|'consent'`
  - `FixedPageSlug` оставить контентные **или** ввести `EditableFixedPageSlug = FixedPageSlug | LegalDocSlug` для редактора
  - `PageBySlug` / `upsertPageBySlug` должны принимать legal slug'и (`content-api.ts`)
- `admin/src/pages/content/FixedPageEditPage.tsx` — расширить `SECTION_META` для 6 legal (title, hint с URL на витрине, defaultTitle = как на BE):
  - privacy → `/legal/privacy/`
  - offer → `/legal/offer/`
  - delivery → `/help/delivery/`
  - refund → `/help/refund/`
  - terms → `/help/terms/`
  - consent → `/legal/consent/`
  - Для legal: hint про rich-text на витрине; hero-upload можно оставить (как у help) — не усложнять
- `admin/src/pages/settings/DocumentsSettingsPage.tsx` — список Link'ов на каждый документ (Card/section-links-grid как SettingsHub)
- `admin/src/App.tsx`:
  - внутри `settings` (уже RequireOwner):  
    `documents` → DocumentsSettingsPage  
    `documents/:slug` → страница-обёртка, которая валидирует LegalDocSlug и рендерит `<FixedPageEditPage section={slug} />` (или 6 явных Route)
  - Альтернатива ок: роуты под `/content/privacy` и т.д. **плюс** хаб в settings со ссылками туда — главное, чтобы owner доходил до редактора
- `SettingsHubPage.tsx` — «Юридические документы» → `<Link to="/settings/documents">`, убрать из `soonItems`

### НЕ трогать
- `backend/` (кроме случайного импорта — не надо)
- ContactsSettingsPage / site_settings
- Mini-App frontend
- Drive-скрипты
- SF (это 2C)

### Критерии готовности
- [ ] Owner: Настройки → Юридические документы → каждый из 6 открывается, load/save через by-slug API
- [ ] Тумблер видимости + SEO + body работают (как у help)
- [ ] Manager не видит (RequireOwner)
- [ ] `cd admin && npx tsc -b` зелёный
- [ ] requisites нет в типах/хабе документов

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/admin"
npx tsc -b --pretty false
# Ручной: backend :4000 + admin → сохранить privacy → GET by-slug OK
```

### Отчёт оркестратору
- файлы, tsc, smoke (если был)
- куда повесили роуты (settings vs content)
- подтверждение: 6 slug'ов, без requisites
