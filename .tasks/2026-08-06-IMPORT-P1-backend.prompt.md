## MURU Executor — [ID: 2026-08-06-IMPORT-P1]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local` (работа в `backend/`)  
**Режим:** Plan mode → план → approve → код  
**Модель:** сильнее Auto (парсинг/валидация/новый модуль + миграция)  
**Канон:** `muru-docs` → `DECISIONS.md` **IMPORT-001**; бриф `.tasks/2026-08-06-IMPORT-GLOBAL-orchestrator-brief.md`  
**Связь:** PROGRESS → bulk-импорт товаров из шаблона `.xlsx`. **Только backend.** Admin-UI = Фаза 2 (не здесь).  
**Prod / staging / VPS — НЕ трогать.**

### Цель
Backend для импорта новых/обновляемых товаров из **чистого** `.xlsx`-шаблона: скачать шаблон → preview (`dryRun`) → commit с частичной записью + персистентный лог. Legacy google-формат и google-sync **не трогать**.

### Контекст (факты канона — не переисследовать с нуля)
- Роутер: `app.use('/api/crm/catalog', crmCatalogRouter)` + `requireCrmAuth()` на весь router.
- **Legacy** (оставить как есть): `POST /api/crm/catalog/import?dryRun=true` — google-формат (`crm-catalog-import.controller.ts` / `crm-catalog-import.service.ts` / `parseXlsxBufferToCatalog`). Переиспользовать **multer** (memoryStorage, xlsx-only) и паттерн dry-run, **не** формат колонок.
- Write-path (обязателен): `crm-catalog.service.ts` → `createCrmCatalogProduct` / `updateCrmCatalogProduct`. Прямых INSERT в обход — запрещено.
- Zod: `createCrmCatalogProductSchema` в `crm-catalog.schemas.ts`.
- Shipping-дефолты при create без веса/габаритов: `product-shipping-defaults.ts` → **3000 г / 22×12×18 см**.
- Экспорт/xlsx-пакет: смотри `crm-catalog-export.service.ts`.
- Sheet-map google (`crm-catalog-sheet-map.ts`) хранит страну как `'Страна'` — **не менять**. Для **нового** шаблона писать admin-канон: `specs['Страна производитель']` (как `ProductEditPage.tsx`: read fallback обоих, write канон).
- `catalog_sync_log` / `catalog-sync-history.service.ts` — **не reuse** (нет file/mode/structured errors, prune ~3 строк). Нужна **новая** таблица + миграция **041**.
- Последняя миграция = `040_…`. `discount_percent NUMERIC(5,2)` (PRICE-001: `price` = LIST).

### Колонки шаблона (ровно 11, RU-заголовки)
`Артикул*`, `Наименование*`, `Стоимость, ₽*`, `Остаток*`, `Цвет`, `Размер`, `Описание`, `Бренд`, `Материал`, `Страна`, `Скидка %`  
(`*` — обязательные.)

Маппинг:
- Артикул → `sku`; Наименование → `name`; Стоимость → `price` (LIST); Остаток → `inStock`; Скидка % → `discountPercent`.
- Цвет → `color` **и** `specs['Цвет']`; Размер → `size` **и** `specs['Размер']` (dual-write как admin).
- Бренд → `specs['Бренд']`; Материал → `specs['Материал']`; Страна → `specs['Страна производитель']`.
- Категории / подкатегории / фото / коллекции / вес / габариты — **не в шаблоне** (вес/габариты → shipping-дефолты; `categoryId` пустой).

### Парсинг чисел (обязательно)
RU: убрать пробелы разрядов и NBSP, `,` → `.`.  
Примеры: `"3 500,00"` → `3500.00`; остаток `"2,00"` → `2` (целое); скидка `"25"` / `"25,5"`.

### Эндпоинты (новые; под тем же CRM-guard)
Префикс: `/api/crm/catalog`. **Не ломать** существующий `POST /import`.

1. `GET /import/template` → `.xlsx`: заголовки (11 колонок) + 1–2 пример-строки + памятка (лист «Инструкция» или комментарии): обязательные поля, RU-числа, категории/фото — в админке.
2. `POST /import/products?dryRun=true&mode=new|upsert` (multipart field `file`) → **preview**: parse + validate; ответ  
   `{ summary: { toCreate, toUpdate, errorRows, total }, rows: [{ row, sku, action: 'create'|'update'|'error', errors: [{ field, message }] }] }`  
   **Без записи в БД и без записи в лог.**
3. `POST /import/products?dryRun=false&mode=new|upsert` → **commit**: валидные строки через write-path; невалидные пропустить и вернуть; **записать прогон в лог**; тот же формат + `importId`.
4. `GET /import/log` → список прогонов (кто, когда, имя файла, mode, counts, свод ошибок). `GET /import/log/:id` — детали (включить).

Query: `dryRun` — строка `"true"` / `"false"` (как legacy: `req.query.dryRun === 'true'`). `mode` обязателен: `new` | `upsert` (иначе 400).

### Валидация (частичный импорт)
- Нет обязательной колонки в заголовках → **400** на весь файл.
- Пустой файл / нет data-строк → 400.
- Построчно (ошибка строки не роняет остальные):
  - `sku`: обязателен, trim; дубль внутри файла → ошибка **обеих** строк; `mode=new` + SKU уже в БД → ошибка строки «SKU уже существует»; `mode=upsert` + существует → `action: update`.
  - `name`: обязателен.
  - `price`: обязателен, число ≥ 0.
  - `inStock`: обязателен, целое ≥ 0.
  - `discountPercent`: опц., 0–100.
  - color/size/description/бренд/материал/страна: опц., trim.
- Лишние колонки — игнорировать.

### Лог (миграция 041)
Новая таблица (имя на усмотрение, напр. `catalog_product_import_log`):  
`id`, кто (`admin` id/email как принято в CRM), `created_at`, `filename`, `mode`, counts (`toCreate`/`toUpdate`/`errorRows`/`total` или эквивалент), structured errors (JSONB), опц. длительность.  
+ `.down.sql`. Сервис list/get + запись только на commit (`dryRun=false`).

### Файлы (ожидаемые)
- `backend/src/services/crm-catalog-product-import.service.ts` (+ vitest) — парсер/валидатор/orchestration
- `backend/src/services/crm-catalog-import-template.service.ts` (+ vitest) — генерация шаблона
- `backend/src/services/crm-catalog-product-import-log.service.ts` (или рядом) — лог
- `backend/src/db/migrations/041_catalog_product_import_log.sql` + `.down.sql`
- Дописать handlers в `crm-catalog-import.controller.ts` **без** поломки google `POST /import`
- Роуты в `crm-catalog.routes.ts` (порядок: `/import/template`, `/import/products`, `/import/log/:id`, `/import/log` — до/отдельно от `POST /import`)
- Тесты vitest

### НЕ трогать
- google-sync / `parseXlsxBufferToCatalog` / формат legacy import
- `crm-catalog-sheet-map.ts` (google mapping)
- платежи / CDEK / auth / settings
- storefront, **admin-UI** (Фаза 2)
- prod/staging, БД на VPS (миграцию применить только локально при необходимости для тестов)

### Критерии готовности
- [ ] `cd backend && npm run build && npm test` — чисто; новые тесты зелёные
- [ ] Тесты: RU-числа (`3 500,00`, `2,00`); нет обяз. колонки → 400; дубль SKU в файле; `mode=new` существующий → error; `mode=upsert` → update; partial commit; discount 0–100; dual-write Цвет/Размер + Бренд/Материал/`Страна производитель`
- [ ] `GET /import/template` — валидный xlsx, 11 колонок
- [ ] `dryRun=true` ничего не пишет (ни products, ни log); `dryRun=false` пишет валидные + log + `importId`
- [ ] Только через write-path
- [ ] `POST /api/crm/catalog/import` (legacy) без регрессии (хотя бы smoke/существующие тесты)

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/backend" && npm run build && npm test
```

### Отчёт оркестратору (обязателен)
1. Изменённые/добавленные файлы + SHA/ветка если есть.
2. Вывод `build` + `vitest` (числа passed/failed).
3. **Контракты** preview / commit / log list / log detail (точные JSON-формы) — для Фазы 2.
4. Список колонок + spec-маппинг как в коде.
5. Имя таблицы лога + суть миграции 041.
6. Риски / что осталось (admin-UI = Ф2).
