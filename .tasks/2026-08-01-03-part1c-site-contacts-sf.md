## MURU Executor — [ID: 2026-08-01-03]

**Репозиторий:** muru-storefront  
**Путь:** `/Users/vasilii/Desktop/code /muru-storefront`  
**Ветка:** feature от `main @ 459eb90` (напр. `feat/settings-site-contacts-sf`). НЕ мержить в main. НЕ деплоить.  
**Связь:** EPIC Settings → Часть **1C**. Backend 1A + Admin 1B **ACCEPT**. Публичный API: `GET /api/content/site-contacts`.

### Цель
Перевести контакты витрины со статики на API с fallback. **Вёрстку не менять.** Соцсети в UI не добавлять (полей соцсетей в текущих компонентах нет).

### Контекст — маппинг API → витрина

Публичный ответ бэка (camelCase):
```
contactPhoneDisplay, contactPhoneHref, contactEmail, contactAddress, contactHours,
contactMapLat, contactMapLng, contactMapZoom,
socialTelegram, socialWhatsapp, socialVk
```

Текущий `siteContacts` в `src/lib/site.ts`:
```
address, phoneDisplay, phoneHref, email, emailHref, hours,
coordinates: { lat, lng }, mapZoom
```

Маппинг при адаптере:
- `phoneDisplay` ← `contactPhoneDisplay`
- `phoneHref` ← `contactPhoneHref`
- `email` ← `contactEmail`
- `emailHref` ← если email непустой: `mailto:${email}`, иначе fallback/`""` (как сейчас `mailto:…`)
- `address` ← `contactAddress`
- `hours` ← `contactHours`
- `coordinates.lat/lng` ← `contactMapLat/Lng` (если null — брать из fallback)
- `mapZoom` ← `contactMapZoom` ?? fallback
- social* — можно держать в типе ответа, **не рендерить** в header/footer/contacts в этом промпте

Если API вернул null для поля, которое сейчас обязательно на UI — подставлять значение из `SITE_CONTACTS_FALLBACK` **по полю** (не ронять весь объект), либо целиком fallback при полном провале fetch. Предпочтение: per-field coalesce с fallback, чтобы частичные настройки из админки не стирали адрес.

### Файлы (ожидаемые)
- `src/lib/site.ts` — переименовать `siteContacts` → `SITE_CONTACTS_FALLBACK` (экспорт); тип `SiteContacts` (без `as const` lock на runtime object, либо отдельный mutable type)
- `src/lib/schemas/` — zod схема ответа `/content/site-contacts` (конверт success/data как у других content endpoints)
- `src/lib/api/content-backend.ts` или `endpoints.ts` — `fetchSiteContacts` / `getSiteContacts()` по образцу `getStaticPage` + `isContentBackendEnabled()`
- Потребители (prop-drill, без client-fetch):
  - `src/components/layout/footer.tsx` (уже async) — `await getSiteContacts()`
  - `src/components/layout/header.tsx` — сделать async RSC; phone из props/await; **`<MobileMenu contacts={…} />`**
  - `src/components/layout/mobile-menu.tsx` (`"use client"`) — убрать import `siteContacts`; принимать `contacts: SiteContacts` props
  - `src/components/contacts/contacts-details.tsx` — props или fetch в родителе
  - `src/components/contacts/contact-map.tsx` (`"use client"`) — props: lat/lng/zoom/address (не import static)
  - `src/components/contacts/contacts-page-content.tsx` / `app/company/contacts/page.tsx` — прокинуть
  - `src/lib/seo/jsonld.ts` — `organizationJsonLd(contacts?: SiteContacts)` с default fallback
  - `src/app/layout.tsx` — async: `const contacts = await getSiteContacts()`; передать в Header/Footer и jsonld; `export const revalidate = 300` на layout **или** cache `revalidate: 300` внутри `getSiteContacts` (как принято в репо для content fetch)

### НЕ трогать
- Вёрстку/CSS/классы (только data source + props)
- Checkout, catalog, banners, legal pages
- backend / admin
- Не добавлять иконки/ссылки соцсетей в footer
- Не раздувать `legalNav`

### Критерии готовности
- [ ] Без `NEXT_PUBLIC_API_BASE` — поведение идентично нынешнему (fallback)
- [ ] С API — контакты из `GET /api/content/site-contacts` (после сохранения в admin 1B)
- [ ] MobileMenu / ContactMap без собственного fetch
- [ ] Пустые/null поля не дают битых `tel:`/`mailto:` (coalesce или не рендерить пустой href — сохранить текущую логику ссылок)
- [ ] `npx tsc --noEmit` / `next build` (или принятый CI-скрипт) зелёный
- [ ] Существующие тесты 29/29; обновить снапшоты/моки если контакты зашиты в тестах layout

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-storefront"
npx tsc --noEmit
npm test   # или vitest — как в package.json
# Опционально: NEXT_PUBLIC_API_BASE=http://localhost:4000/api → header phone меняется после PUT в admin
```

### Отчёт оркестратору
- Ветка/SHA, список файлов
- tsc + tests
- Как сделан revalidate/cache
- Подтверждение: вёрстка не менялась; client-компоненты получают props
- Риски

**STOP-1** после этого промпта: оркестратор verify 1A+1B+1C → ACCEPT Части 1 → Part 2.
