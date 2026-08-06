## MURU Executor — [ID: 2026-08-06-IMPORT-P2]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local` (работа в `admin/`)  
**Режим:** Plan mode → план → approve → код  
**Модель:** Auto OK (UI на готовых контрактах); сильнее — если рефакторишь layout вкладок  
**Канон:** IMPORT-001; контракты из ACCEPT Ф1 (ветка `feature/import-p1-products` @ `d54f53c`, отчёт `.tasks/2026-08-06-SUPERVISOR-REPORT-IMPORT-P1.md`)  
**Связь:** Фаза 2 — admin UI для clean product import. Backend уже есть — **не менять** API/миграции без крайней нужды.  
**Prod / staging / VPS — НЕ трогать.**

### Цель
В разделе «Товары» админки: UI «Импорт товаров» (чистый шаблон) — скачать шаблон → загрузить .xlsx → выбрать `mode` → предпросмотр → подтвердить commit → результат; плюс просмотр **лога импортов**. Legacy google «Импорт / Экспорт» **сохранить** (не ломать).

### Контекст UI (якоря)
- Нав каталога: `admin/src/components/catalog/CatalogLayout.tsx` — есть пункт «Импорт / Экспорт» → `/catalog/import-export`.
- Legacy страница: `admin/src/pages/catalog/ImportExportPage.tsx` + `downloadExport` / `importCatalog` в `catalog-api.ts` (`POST /import?dryRun=`).
- UI-kit: `FileDropzone`, `Table`, `Button`, `Card`, `PageHeader`, `useToast`, `useConfirm`, `useCatalogMetaContext` (`readOnly` → блокировать write).
- Роутинг: `admin/src/App.tsx`.

### Контракты backend (строго; envelope `{ success, data }`)
База: `/api/crm/catalog` (через существующий `apiFetch` / fetch с cookie как в catalog-api).

1. `GET /import/template` → blob xlsx (`muru-product-import-template.xlsx`).
2. `POST /import/products?dryRun=true&mode=new|upsert` multipart field **`file`** → preview  
   `data: { summary:{toCreate,toUpdate,errorRows,total}, rows:[{row,sku,action:'create'|'update'|'error', errors:[{field,message}]}] }`  
   **Всегда передавай `dryRun=true` явно** (без query бэк считает commit).
3. `POST /import/products?dryRun=false&mode=…` → commit + `data.importId`.
4. `GET /import/log` → `{ items:[{ id, createdAt, filename, mode, adminId, adminEmail, summary, durationMs }] }`.
5. `GET /import/log/:id` → то же + `errors:[{row,sku,field,message}]`.

Колонки шаблона (для подсказок в UI):  
`Артикул*`, `Наименование*`, `Стоимость, ₽*`, `Остаток*`, `Цвет`, `Размер`, `Описание`, `Бренд`, `Материал`, `Страна`, `Скидка %`.

### UX (обязательно)
- Секция/вкладка **«Импорт товаров»** (чистый шаблон), отдельно от legacy google import/export. Варианты OK:
  - расширить `ImportExportPage` двумя явными блоками (рекомендуется), **или**
  - новый route `/catalog/product-import` + пункт в `CatalogLayout`.
- Кнопка «Скачать шаблон».
- `FileDropzone` .xlsx only.
- Переключатель режима: **Только новые** (`new`) / **Создать или обновить** (`upsert`).
- Кнопка «Предпросмотр» → таблица строк: row#, SKU, action (create/update/error), ошибки.
- Summary chips/строка: toCreate / toUpdate / errorRows / total.
- Кнопка «Импортировать» активна только после успешного preview на **том же файле**; `useConfirm` перед commit.
- После commit — показать результат (+ importId) и обновить лог.
- Секция **«Лог импортов»**: список прогонов; клик → детали с ошибками (drawer/вторая таблица).
- `readOnly` (sheets mode): write-кнопки disabled + понятный текст (как на legacy странице).
- Soft-предупреждение при `upsert`: пустые ячейки на бэке могут обнулить скидку/описание/цвет/размер у существующего SKU — коротко в UI.

### API-хелперы
Добавить в `admin/src/lib/catalog-api.ts` (+ типы в `types/catalog.ts` при необходимости):
- `downloadProductImportTemplate()`
- `previewProductImport(file, mode)` / `commitProductImport(file, mode)`
- `listProductImportLogs()` / `getProductImportLog(id)`

Не менять сигнатуры legacy `importCatalog` / `downloadExport`.

### НЕ трогать
- Backend import services / migration 041 (уже ACCEPT), кроме если обнаружен блокер UI — тогда стоп и отчёт.
- google-sync / legacy `POST /import` flow.
- Storefront, платежи, auth.
- Prod/staging.

### Критерии готовности
- [ ] `cd admin && npm run build` (или `tsc -b`) чисто.
- [ ] Скачивается шаблон; preview показывает create/update/error; commit пишет и появляется в логе (ручной прогон против локального API на ветке Ф1).
- [ ] Legacy «Импорт / Экспорт» google по-прежнему открывается и не сломан визуально/API-вызовами.
- [ ] `readOnly` блокирует commit/preview write.
- [ ] `dryRun` всегда явный в query.

### Проверка
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/admin" && npm run build
# ручной: backend на :4000 с CATALOG_SOURCE=crm + mig 041; admin → Импорт товаров → template → preview → commit → лог
```

### Отчёт оркестратору
- Файлы, ветка/SHA; вывод build.
- Как устроена навигация (блок на ImportExport vs новый route).
- Скрин/описание preview→commit→лог.
- Риски / отклонения от контракта.
