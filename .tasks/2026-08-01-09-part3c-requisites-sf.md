## MURU Executor — [ID: 2026-08-01-09]

**Репозиторий:** muru-storefront  
**Путь:** `/Users/vasilii/Desktop/code /muru-storefront`  
**Ветка:** `feat/settings-site-contacts-sf` (1C+2C). НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **3C**. BE/Admin 3A+3B **ACCEPT**. Public: `GET /api/content/requisites`.

### Цель
Таблица реквизитов на `/company/requisites/` из API с fallback на текущие плейсхолдеры. Вёрстку таблицы не менять (только data source).

### API → UI маппинг

Публичный DTO (все nullable strings):
`reqFullName, reqShortName, reqInn, reqOgrnip, reqLegalAddress, reqActualAddress, reqPhone, reqEmail, reqSite, reqBankDetails`

Текущие labels в `requisites-table.tsx` (сохранить порядок/тексты labels):
| label | API field |
|---|---|
| Полное наименование | reqFullName |
| Сокращённое наименование | reqShortName |
| ИНН | reqInn |
| ОГРНИП | reqOgrnip |
| Юридический адрес | reqLegalAddress |
| Фактический адрес | reqActualAddress |
| Телефон, факс | reqPhone |
| Электронная почта | reqEmail |
| Сайт | reqSite |
| Банковские реквизиты | reqBankDetails |

Per-field coalesce с `REQUISITES_FALLBACK` (текущий хардкод `REQUISITES` → fallback values), как у site-contacts. Пустой/null → fallback по полю.

### Сделать
1. Zod schema `PublicRequisites` в `src/lib/schemas/`
2. `fetchRequisites` в content-backend (`/content/requisites`, `revalidate: 300`)
3. `getRequisites()` в endpoints — `cache` + `isContentBackendEnabled` + adapt to `{ label, value }[]` или typed object
4. `RequisitesTable` — async server component **или** принимать `rows` props от page; page уже async — предпочтительно: `page.tsx` await `getRequisites()`, передать в table
5. Fallback идентичен нынешним placeholder values (без реальных ПДн клиента — уже плейсхолдеры)

### НЕ трогать
- CMS `getStaticPage("requisites")` для title/SEO shell страницы
- Layout/header/footer contacts (1C)
- legalNav / help pages
- backend/admin
- Дизайн dl/grid таблицы

### DoD
- [ ] Без API — визуально как сейчас
- [ ] С API — values из `/api/content/requisites` (после save в admin)
- [ ] `tsc --noEmit` + `npm test` зелёные

### Отчёт
файлы, tsc/tests, маппинг, подтверждение: вёрстка labels/классов без изменений
