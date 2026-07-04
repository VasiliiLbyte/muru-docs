# MURU — API Contract (backend ↔ storefront)

Единый контракт между **каноническим бэкендом** (`muru-backend-local`, порт `4000`) и **витриной** (`muru-storefront`).

**Версия:** 2026-07-04  
**Источник правды в коде:** `muru-backend-local/backend/src`, `muru-storefront/src/lib/api`, `muru-storefront/src/lib/schemas`

При изменении эндпоинтов или полей — обновлять этот файл **и** `PROGRESS.md` (лог сессии).

---

## 1. Общие правила

### Base URL

| Клиент | Env | Пример |
|---|---|---|
| Storefront (payments, CDEK) | `NEXT_PUBLIC_API_BASE` | `http://localhost:4000/api` |
| Storefront (каталог) | `NEXT_PUBLIC_CATALOG_API_BASE` | `http://localhost:4000/api` |

Оба могут указывать на один хост. Каталог включается **только** при заданном `NEXT_PUBLIC_CATALOG_API_BASE` (`catalog-backend.ts`).

### Envelope (все JSON-ответы бэкенда)

```json
// Успех
{ "success": true, "data": <T>, "error": null }

// Ошибка
{ "success": false, "data": null, "error": { "message": "...", "code": "VALIDATION|UNAUTHORIZED|NOT_FOUND|UPSTREAM", "details": ... } }
```

Реализация: `muru-backend-local/backend/src/utils/api-response.ts`  
Клиент витрины: `apiEnvelopeFetch()` в `muru-storefront/src/lib/api/client.ts`

### CORS

Бэкенд разрешает origin из allowlist (`index.ts`):

- Прод: `murushop.ru`, `murushop.online`, `web.murushop.ru`, `muru-blue.vercel.app` + `ALLOWED_ORIGINS`
- Dev: `localhost:3000`, `localhost:5173`, `localhost:4173`

Перед go-live витрины на `muru.ru` — добавить домен в `ALLOWED_ORIGINS` на VPS.

### Каналы заказов

| `channel` | Auth | Промокоды | `telegram_user_id` |
|---|---|---|---|
| `telegram` | JWT (initData) | ✅ | обязателен |
| `web` | нет (гостевой) | ❌ | `null` |

Миграция: `014_web_identity.sql`

---

## 2. Каталог (публичный, без auth)

Префикс: `/api/catalog`

| Метод | Путь | Storefront | Rate limit | Описание |
|---|---|---|---|---|
| `GET` | `/tree?subcategories=1` | `fetchCatalogTree()` | — | Дерево категорий (2 уровня) |
| `GET` | `/products` | `fetchCatalogProducts()` | — | Список товаров |
| `GET` | `/products/:sku` | `fetchCatalogProductBySku(sku)` | — | Карточка по SKU (uppercase на бэке) |
| `POST` | `/restock-notify` | — | 5/min IP | Уведомление о поступлении (Telegram) |

### Query `/products` (опционально)

`category`, `categorySlug`, `subcategory`, `subcategorySlug`, `q`, `color`, `size`, `priceMax`, `debug=1`

### Маппинг backend → storefront

Адаптер: `muru-storefront/src/lib/api/catalog-backend.ts` → `adaptProduct()`, `adaptTree()`

| Backend поле | Storefront `Product` |
|---|---|
| `sku` | `id`, `sku`, `slug` |
| `name` | `title` |
| `price`, `discountPercent` | `price`, `oldPrice`, `isOnSale` |
| `inStock` (>0) | `inStock: boolean` |
| `imageUrls[]` | `images[].url` |
| `category`, `subcategorySlug` | `categorySlugs[]` |

### Гидрация корзины по SKU (закрыто 2026-07-02)

| Storefront | Backend | Статус |
|---|---|---|
| `getProductBySku()` → `fetchCatalogProductBySku(sku)` | `GET /api/catalog/products/:sku` | ✅ При `NEXT_PUBLIC_API_BASE` (или `NEXT_PUBLIC_CATALOG_API_BASE`) каталог-бэкенд включён; SKU нормализуется `trim().toUpperCase()`. MSW `GET /products/by-sku/:sku` — только dev-fallback без API_BASE. |

Контент (collections, lookbooks, static pages) — **не API**, статика в `muru-storefront/src/lib/content/`.

---

## 3. Веб-оплата (гостевой чекаут)

Префикс: `/api/payments`  
Auth: **не требуется**  
Rate limit: create `10/min`, status `30/min` (по IP)

### `POST /payments/web/create`

Создаёт платёж ЮKassa, сумма пересчитывается на сервере (`computeTrustedPricing`).

**Request body** (strict, без `promoCode`):

```typescript
{
  items: Array<{ sku: string; quantity: number; color?: string; size?: string }>  // min 1
  deliveryMode: "delivery" | "pickup"
  deliveryOption: string | null
  deliveryEta: string | null
  address: string          // обязателен при deliveryMode === "delivery"
  comment: string
  birthDate: string | null
  recipientName: string    // min 2
  recipientPhone: string   // min 10
  email: string            // обязателен, в чек 54-ФЗ
  cdekTariffCode: number | null
  cdekCityCode: number | null
  cdekCityName: string | null
  cdekPvzCode: string | null
  cdekPvzAddress: string | null
}
```

Zod: `webSnapshotSchema` в `payments.controller.ts`  
Storefront: `WebCheckoutSchema` в `src/lib/schemas/order.ts`

**Response `data`:**

```typescript
{ paymentId: string; confirmationUrl: string }
```

**Поведение:**
- `channel: "web"`, `telegramUserId: null`, `promoCode: null`
- При `delivery` — СДЭК пересчитывается на сервере, тариф валидируется
- `in_stock` проверяется до создания платежа (race при параллельной оплате возможен)
- ЮKassa credentials: `YOOKASSA_WEB_SHOP_ID/SECRET_KEY` (fallback на основные в dev)

### `GET /payments/web/:paymentId/status`

**Response `data`:**

```typescript
{ status: string; orderId: number | null }
```

Фильтр: только `channel = 'web'`.  
Storefront: поллинг на `/checkout/return/` (3с × 40 попыток).

### Webhook (общий)

`POST /yookassa-webhook` — до CORS, IP-guard. Оба магазина (telegram + web) → один endpoint.

---

## 4. СДЭК (веб-чекаут)

Префикс: `/api/cdek`

| Метод | Путь | Auth | Rate limit | Storefront |
|---|---|---|---|---|
| `GET` | `/health` | нет | — | — |
| `GET` | `/cities?q=` | нет | 60/min IP | `getCdekCities()` |
| `GET` | `/tariff-list` | нет | 30/min IP | — |
| `GET` | `/address-suggest?q=&city=` | нет | 60/min IP | `getCdekAddressSuggestions()` |
| `GET` | `/pickup-points?cityCode=` | нет | 30/min IP | `getCdekPvz()` |
| `POST` | `/web/calculate` | нет | 20/min IP | `calculateCdekWeb()` |
| `POST` | `/calculate` | JWT | 20/min user/IP | Mini App only |

### `POST /cdek/web/calculate`

**Request:**

```typescript
{
  toCityCode: number
  items: Array<{ sku: string; quantity: number }>
}
```

**Response `data`:**

```typescript
{
  door: { tariffCode, deliverySum, periodMin, periodMax } | null
  pvz:  { tariffCode, deliverySum, periodMin, periodMax } | null
  errors: string[]
}
```

Storefront schema: `CdekCalcResultSchema`

---

## 5. Telegram-канал (справка, не storefront)

| Метод | Путь | Auth |
|---|---|---|
| `POST` | `/api/auth/telegram` | initData → JWT |
| `POST` | `/api/payments/create` | JWT |
| `POST` | `/api/payments/invoice` | JWT (native TG pay) |
| `GET` | `/api/payments/:paymentId/status` | JWT |
| `POST` | `/api/cdek/calculate` | JWT |

**Удалено (небезопасно):** `POST /api/orders/create` — клиентские цены не принимаются. Заказ создаётся после оплаты через webhook / `fulfillPaidPayment`.

---

## 6. Env-матрица (локальный e2e)

### Backend (`muru-backend-local/.env`)

```env
PORT=4000
DATABASE_URL=...
CDEK_ENV=production          # или test
CDEK_CLIENT_ID=...
CDEK_CLIENT_SECRET=...
DADATA_API_KEY=...
YOOKASSA_SHOP_ID=...
YOOKASSA_SECRET_KEY=...
YOOKASSA_WEB_RETURN_URL=http://localhost:3000/checkout/return
# Опционально отдельный магазин для web:
# YOOKASSA_WEB_SHOP_ID=...
# YOOKASSA_WEB_SECRET_KEY=...
```

### Storefront (`muru-storefront/.env.local`)

```env
NEXT_PUBLIC_API_BASE=http://localhost:4000/api
NEXT_PUBLIC_CATALOG_API_BASE=http://localhost:4000/api
# NEXT_PUBLIC_API_MOCKING=   # не включать при e2e с бэком
```

---

## 7. Чеклист совместимости при изменениях

- [ ] Envelope `{ success, data, error }` не менять без обновления `apiEnvelopeFetch`
- [ ] Новый публичный эндпоинт → CORS + rate limit
- [ ] Изменение snapshot → синхрон `webSnapshotSchema` + `WebCheckoutSchema`
- [ ] Новый домен витрины → `ALLOWED_ORIGINS` на бэке
- [ ] Миграция БД → `DEPLOY.md` + запись в `PROGRESS.md`

---

## 8. Changelog

| Дата | Изменение |
|---|---|
| 2026-07-02 | Первая версия: catalog, web payments, CDEK web, envelope, env matrix |
| 2026-07-02 | Зафиксирован блокер `getProductBySku` vs `/catalog/products/:sku` |
| 2026-07-02 | Блокер закрыт: fallback `CATALOG_API_BASE` ← `API_BASE`, uppercase SKU, `getProductBySku` → catalog fetch |
