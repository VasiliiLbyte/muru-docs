## MURU Executor — [ID: 2026-08-01-06]

**Репозиторий:** muru-storefront  
**Путь:** `/Users/vasilii/Desktop/code /muru-storefront`  
**Ветка:** продолжать `feat/settings-site-contacts-sf` (1C) **или** новая от `main @ 459eb90` / tip с 1C. НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **2C**. Backend 2A + Admin 2B **ACCEPT**. Страницы тянут `getStaticPage(slug)` (уже умеет CMS + static fallback).

### Цель
Роуты для delivery/refund/terms/consent + DEFS fallback + прошить 3 stub-тайла на `/help/`. `legalNav` (privacy+offer) **не** раздувать. Соцсети/контакты 1C не ломать.

### Сделать

1. **Страницы** (копия паттерна `app/legal/privacy/page.tsx`):
   - `app/help/delivery/page.tsx` — slug `delivery`, path `/help/delivery/`
   - `app/help/refund/page.tsx` — slug `refund`, path `/help/refund/`
   - `app/help/terms/page.tsx` — slug `terms`, path `/help/terms/`
   - `app/legal/consent/page.tsx` — slug `consent`, path `/legal/consent/`
   - Каждая: `revalidate = 300`, `generateMetadata`, `ContentShell` + `StaticProse`, `getStaticPage("<slug>")`
   - Breadcrumbs: для help-* желательно Home → Клиентам (`/help/`) → текущая; для consent — как privacy (через `contentBreadcrumbs`). Можно добавить хелпер `helpCrumb()` рядом с `companyCrumb` если удобно.

2. **Fallback** `src/lib/content/pages.ts` DEFS — добавить:
   - delivery, refund, terms, consent  
   - titles согласовать с admin/BE где возможно (delivery/refund/terms/consent как в LEGAL_DOC_SECTION_META; privacy title в DEFS уже длиннее BE — **не менять** существующий privacy/offer body)
   - body: нейтральный lorem через дефолт DEFS (как у company), без копипасты LEGAL_* кроме уже существующих

3. **Help tiles** `app/help/page.tsx`:
   - «Доставка» → `/help/delivery/`
   - «Возврат» → `/help/refund/`
   - «Условия обслуживания» → `/help/terms/`
   - Остальные (`Отзывы`, `Корпоративные подарки`, `Подарочные карты`) оставить `href: "#"`

### НЕ трогать
- `legalNav` в `site.ts` (только privacy+offer)
- Вёрстку help/legal сверх замены href и новых page.tsx
- Checkout consent checkbox wiring (только роут страницы)
- backend / admin
- requisites (Часть 3)

### Критерии готовности
- [ ] Четыре URL отдают 200 (с fallback без API и с API если страница visible в CMS)
- [ ] Help tiles 3 ссылки рабочие
- [ ] `npx tsc --noEmit` + `npm test` зелёные
- [ ] 1C contacts props/getSiteContacts не регрессировали

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-storefront"
npx tsc --noEmit
npm test
# опционально next build / curl localhost pages
```

### Отчёт оркестратору
- ветка, файлы, tsc/tests
- подтверждение: legalNav не тронут; только 3 help stubs прошиты
