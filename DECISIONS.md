# MURU — Accepted decisions

Living log of locked product/tech decisions. Append-only; do not rewrite history.

---

## URL-001 (2026-07-29) — 12 canonical-conflict PDP URLs from muru.ru

**Status:** Accepted limitation (Option A). Backend slug rename (Option B) deferred / out of URL D2–D6 scope.

**Problem:** 12 muru.ru product URLs use stale Bitrix `CODE` slugs that coincide with the **canonical** `final_slug` of a *different* CRM SKU. The S0 redirect generator correctly refused these rules (`source is a canonical URL that must serve 200`). On the storefront those paths therefore return **200 for the wrong product**.

| muru.ru path | muru.ru SKU | storefront owner SKU |
|---|---|---|
| `/catalog/vazy-i-aksessuary/vazy-i-kuvshiny/keramicheskaya-vaza-glina/` | MU0061 | MU0038 |
| `/catalog/vazy-i-aksessuary/vazy-i-kuvshiny/vaza-keramicheskaya/` | MU0204 | MU0168 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskiy-salatnik-zhivaya-forma/` | MU0032 | MU0013 |
| `/catalog/kukhnya-i-stolovaya/posuda/supovoe-blyudo-keramicheskoe-zhivaya-forma/` | MU0033 | MU0014 |
| `/catalog/kukhnya-i-stolovaya/posuda/salatnik-keramicheskiy-zhivaya-forma/` | MU0034 | MU0015 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskoe-blyudo-s-ruchkami-zhivaya-forma/` | MU0035 | MU0016 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskoe-servirovochnoe-blyudo-s-ruchkami-zhivaya-forma/` | MU0036 | MU0017 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskoe-servirovochnoe-blyudo-glubina/` | MU0037 | MU0018 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskaya-kruzhka-mudzhi/` | MU0040 | MU0021 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskoe-servirovochnoe-blyudo-mudzhi/` | MU0041 | MU0022 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskaya-chaynaya-para-mudzhi/` | MU0042 | MU0023 |
| `/catalog/kukhnya-i-stolovaya/posuda/keramicheskaya-tarelka-mudzhi/` | MU0043 | MU0024 |

**Why Option A:** Fixing without shadowing live PDP requires renaming owner SKUs’ slugs (S1 / backend), which is out of the D2–D6 storefront-only scope. On muru.ru the CODE already mismatches the visible title; after Bitrix decommission the conflict set shrinks to zero SEO value.

**Follow-up (optional SEO epic):** backend slug dedup for the 12 owner SKUs + add 301 from old muru.ru path → correct product.

---

## M7-V4 (2026-07-29) — «Избранное» в мобильной шапке — не дефект

**Status:** Closed as non-defect.

**False positive:** скан по видимому тексту не находил «Избранное» на 393px.

**Fact:** `header-actions.tsx` — ссылка/кнопка Избранное есть; на мобиле скрыта только подпись, `aria-label`/`href` на месте (подтверждено STOP-1 tab-order: `a:Избранное`).

---

## M7-V7 (2026-07-29) — отдельная кнопка сортировки на мобиле — не дефект

**Status:** Closed as non-defect.

Мобильный ряд в `catalog-toolbar.tsx` уже содержит кнопку с текущим значением сортировки + «Фильтры». Отдельной задачи на добавление сортировки нет.

---

## M7-4 (2026-07-29) — блок «Новинки» на главной только mobile

**Status:** Accepted.

Секция `HomeProductRail` обёрнута `lg:hidden`. Причина: десктопная главная под `scroll-snap-type: y mandatory` (≥1024px). Десктопный блок новинок → **DEP-045** / backlog.

Допущение исполнителя: если `newArrival:true` даёт &lt;4 items — fallback на `sort:"new"` (чтобы T3 ≥4). Пустой ответ → `null`.

---

## M7-3 (2026-07-29) — h1 листинга на мобиле → `text-h2`

**Status:** Accepted.

Крошки/h1: `mb-3 pt-4` / `text-h2` на мобиле; `lg:mb-6 lg:pt-8` / `lg:text-display` без изменений десктопа. Новых токенов нет.

---

## M7-T6 (2026-07-29) — метрика T6 после переноса тулбара

**Status:** Accepted reinterpretation (надзорный конфликт разрешён числами).

**Проблема:** до M7 тулбар был *под* гридом → gap header→card = только служебный блок (**165.6px**). После M7-2 тулбар *над* гридом (~**85px**) + сжатие V3 → **raw gap 233.8** &gt; baseline, при этом **effective (raw − toolbarH) = 148.8 ≤ 165.6**.

Полностью уложить raw ≤165 при sticky-тулбаре сверху нельзя без обнуления отступов (запрещено надзором). T6 в e2e измеряет **effective** gap (V3-guard на крошки+h1). Raw фиксируется в отчётах/DECISIONS.

---

## M8-V1-REVERSAL (2026-07-30) — откат M7-1 ATC-иконки

**Status:** Accepted.

**Проблема:** вводная «у оригинала нет кнопки корзины в карточке» опровергнута живым скрином устройства (2026-07-30) + curl mobile-UA STOP-1: `button.btn.to_cart…Добавить в корзину`.

**Решение:** M8-1 вернул полноширинную полосу на всех вьюпортах по E2 (36px / 14px / brand). Иконка ShoppingBag удалена.

**Урок:** парность сверять по живому muru.ru с mobile UA / устройства; единичное снятие шаблона без кнопки не считается доказательством отсутствия.

---

## M8-5 (2026-07-30) — позиция рейла «Новинки»

**Status:** Accepted.

Если среди CMS-баннеров title матчит `/новинк/i` — `HomeProductRail` рендерится сразу после этого баннера; заголовок рейла `sr-only` (баннер = шапка секции), ссылка «Все новинки» видима. Иначе — после первого баннера, h2 видимый.

---

## URL-002 (2026-07-29) — D1 “wrong targets” was a false positive

Slug-token Jaccard is **invalid** as a redirect correctness metric when Bitrix CODE was not regenerated after renames. Identity = article `MU####` (or explicit product id). Confirmed: pairing in `redirects_preview.csv` is SKU-correct; live muru.ru HTML article matches prod API by-slug on sampled pairs.

---

## RF-001 (2026-07-31) — Диагноз: RF-недоступность = ТСПУ/DPI, меры вероятностные

**Status:** Accepted (рабочая гипотеза, подтверждена косвенно; прямого пруфа ТСПУ не бывает).

Симптом: murushop.ru/web.murushop.ru не грузятся из RF (мобильные операторы, без VPN); из Латвии — грузятся. Origin: Beget VPS `109.172.38.194` (AS198610); Beget под санкциями ЕС с 23.07.2026. Диагноз: вмешательство DPI/ТСПУ на уровне TLS-handshake/SNI к данному AS/IP. Следствие: **все меры (h2, TLS1.2-only, CDN edge) вероятностные**, универсального решения нет; каждая мера валидируется ТОЛЬКО живым тестом заказчика из RF без VPN. Отрицательный локальный тест из Латвии (Browser 1) ничего не говорит о RF-симптоме — только о протоколе.

---

## RF-002 (2026-07-31) — TLS1.3 off на origin (TLS1.2-only), обратимо

**Status:** Accepted. **Дата пересмотра: после стабильного зелёного edge-теста (или 2026-09-01, что раньше).**

`/etc/letsencrypt/options-ssl-nginx.conf` → только TLSv1.2; бэкап `options-ssl-nginx.conf.bak-20260731-060529`. Мотив: паттерны TLS1.3 handshake хуже проходят ТСПУ (вероятностная мера, как h2). Цена: минус forward secrecy улучшения/скорость 1.3 для всех клиентов. Откат: восстановить `.bak` → `nginx -t` → reload.

---

## RF-003 (2026-07-31) — Выбор Yandex Cloud CDN как edge

**Status:** Accepted.

YC CDN (инфраструктура на сети Gcore, RF-POP присутствуют) выбран edge-слоем перед origin. Мотивы: RF-присутствие; LE-cert через Certificate Manager с авто-renew по CNAME; полная DNS-обратимость cutover; низкая цена (мин. пакет 150₽/мес). Риск: пройдёт ли ТСПУ по SNI edge-хостов `*.yccdn.ru` — неизвестно до теста заказчика (см. RF-001). Ресурс: `bc8rqhgfke2xmg6c73li`, edge `f69e224d86aa8887.topology.gslb.yccdn.ru`.

---

## RF-004 (2026-07-31) — Порядок: web-first, muru.ru потом

**Status:** Accepted.

Сначала cutover `web.murushop.ru` за edge (низкий риск, DNS-откат за минуты) → тест заказчика. Cutover `muru.ru` — ТОЛЬКО после зелёного теста и ТОЛЬКО за валидированный edge (`www.muru.ru`→CNAME edge, apex 301→www), НЕ на голый Beget-IP: иначе замена рабочего Bitrix на сайт с тем же RF-симптомом.

---

## RF-005 (2026-07-31) — Split origin/apex = Фаза 2

**Status:** Accepted (scope-гейт).

Фаза 1: за CDN только `web`. Apex `murushop.ru` (API `/api/*`, `/img/*`, uploads), `www`, `api-staging` остаются на прямых A-записях — их дубль-A НЕ трогать при cutover web. Фаза 2 (перевод apex/API за edge) — отдельное решение; обязательное условие: пересмотр W-SEC trusted-proxy модели с учётом нового CDN-хопа в XFF-цепочке.

---

## SET-001 (2026-08-01) — Settings secrets: variant A (env only)

**Status:** Accepted. Deployed DEP-055 (`f75656f`).

Несекретные поля CDEK/YooKassa (from_location, package defaults, VAT/tax system, shopId **read-only display**) живут в `site_settings` + `runtime-config.service` = `DB ?? env`. **Секреты** (`CDEK_CLIENT_SECRET`, `YOOKASSA_SECRET_KEY`, …) — **только env**; никогда в API JSON / admin forms. Перенос секретов в БД — отдельное решение (не этот epic).

---

## SET-002 (2026-08-01) — Legal docs via `content_pages` + `LEGAL_DOC_SLUGS`

**Status:** Accepted. Deployed DEP-055.

Юридические страницы (`privacy|offer|delivery|refund|terms|consent`) — не отдельная таблица; allowlist `LEGAL_DOC_SLUGS` с тем же upsert/get, что fixed pages. Admin: Settings → Документы → FixedPageEdit. Storefront routes `/help/*`, `/legal/*` читают CMS с static fallback.


---

## PRICE-001 (2026-08-02) — Канон цены: price = LIST, sale вычисляется

**Status:** Accepted. Deployed (BE master a215bfb + mig `039_price_list_restore.sql` [21 SKU], SF adaptProduct list→sale). Восстановлено в доках 2026-08-03 после doc-drift.

**Симптом:** checkout UI ~1440+420, ЮKassa считала 1572. **Root cause** (подтверждён payments snapshot): `discount_percent=20` применялся к уже «sale»-цене, лежавшей в `products.price` (1440×0.8=1152)+420=1572.

**Канон:**
- `products.price` = LIST (полная цена без скидки).
- `sale = round(list × (1 − d/100), 2)`; оплата — по той же формуле.
- Admin: поле «Цена без скидки» + preview «со скидкой».
- Storefront и Mini App отображают sale.
- Пример: MU0182 price=1800, discount=20 → sale=1440.

**Soft:** миграция 039 НЕ идемпотентна — повторно не гонять. Deploy order (урок инцидента): BE tip + миграция → deploy.sh → SF (иначе SF раньше BE даёт кратковременный неверный показ ~1152).

---

## STK-001 (2026-08-02) — Механизм склада: списание при оплате + единый журнал движений

**Status:** Accepted. Deployed DEP-056. Восстановлено в доках 2026-08-03 после doc-drift (аппенд был не закоммичен и снесён `reset --hard`).

**Момент списания (реш. 1):** сток списывается в момент успешной оплаты — заказ и создаётся при оплате (`completeOrderAfterPayment → createOrder`, декремент в той же транзакции). Резерва до оплаты нет. Идемпотентно по построению (createOrder на оплаченный заказ — один раз, привязка платежа атомарна).

**Оверселл (реш. 2):** между валидацией наличия и оплатой сток не резервируется; `GREATEST(0, …)` не даёт минус. Принято как есть при текущем трафике; резерв — будущая доработка.

**Журнал `stock_movements` + `applyStockDelta`:** таблица (product_id + снапшот sku/name, delta знаковое, type sale/return/adjustment, reason, order_id, stock_before/after, actor). Единый хелпер `applyStockDelta(client,…)` внутри транзакции; переведены все точки: декремент в createOrder, orphan-возврат, ручная правка стока. Импорт/синх НЕ логируется (реш. 3).

**Возврат и статусы (реш. 4):** terminal = {Отменён, Возврат}. Централизованный `settleOrderStockOnStatusChange` на переход в terminal из нетерминала — из всех путей (cancel, PATCH, рефанд-вебхук), в одной транзакции; идемпотентность по журналу (нет повторного `return` по order_id). Исправлена ловушка: раньше PATCH-статус на «Возврат» сток не возвращал.

**Авто-рефанд (реш. 4):** вебхук ЮKassa `refund.succeeded` — полный рефанд → статус «Возврат» + возврат стока + сигнал в бот; частичный → только сигнал. Реализовано в DEP-056 (`refund-webhook.service.ts`).

**Админка:** страница «Склад → Движения» (фильтры товар/тип/даты, пагинация).

---

## YK-001 (2026-08-04) — Mini App YooKassa open latency: ACCEPTABLE / deferred

**Status:** Accepted (operator gate close). Phase B instrumentation **not** shipped.

**Symptom:** Mini App pay succeeds but transition to YooKassa felt slower than web.

**Phase A evidence (prod 2026-08-04):**
- Handler: `POST /api/payments/invoice` → 200 (not `/create` / `/web/create`).
- Stopwatch click → native sheet: **1.51 / 1.69 / 1.92 s** (≈1.7 s p50); YK UI may still load after sheet (client).
- H5: MainButton has no progress copy; can feel like a double-press.
- Web stopwatch skipped (operator: muru.ru already feels fast).
- nginx access has no `$request_time` stage split; code has no `[pay-timing]` logs.

**Static path:** `openInvoice` only after `await createInvoice` → server `computeTrustedPricing` (+CDEK if delivery) → `createInvoiceLink` (Telegram). Sequential external RTTs expected.

**Decision:** Treat current ~1.5–2 s as **ACCEPTABLE** for now. No payment-logic change. No Phase B deploy.

**Reopen if latency regresses / complaints return:**
1. `.tasks/2026-08-04-YK-MA-LATENCY-PHASE-A-RUNBOOK.md`
2. After go: `.tasks/2026-08-04-YK-MA-LATENCY-PHASE-B.prompt.md` (`[pay-timing]`, then revert)
3. Soft UX candidate (low risk): MainButton progress while `isSubmitting` (F1) — separate prompt, not part of this close.

---

## AUTH-001 (2026-08-04) — Переход на phone-first passwordless (SMS OTP через sms.ru)

**Status:** Accepted (решения Василия 2026-08-04). Реализация — эпик backend-first, staging-first. Спека: [`.tasks/2026-08-04-AUTH-PHONE-OTP-EPIC.md`](.tasks/2026-08-04-AUTH-PHONE-OTP-EPIC.md).

**Продуктовые решения:**
1. **Delivery:** SMS-код (наш OTP через sms.ru `sms/send`), UX как Ozon/WB/Lamoda. `callcheck` (звонок) — возможное удешевление позже, не сейчас.
2. **Model:** Passwordless — только OTP. Пароль убираем (`password_hash` → nullable/удаляется, forgot/reset/login-by-password снимаются).
3. **Identity:** `phone` = основной логин (UNIQUE, NOT NULL, нормализованный RU). JWT/refresh модель без изменений (payload `customerId`).
4. **Email:**
   - Web-**checkout**: email **обязателен** (на него уходит подтверждение заказа).
   - Гостевой заказ (без явной регистрации): собираем email + подтверждаем телефон по OTP → **де-факто создаётся кабинет** (phone verified, email сохранён, заказ привязан).
   - **Mini App:** обязателен только телефон; email не требуется (identity через Telegram, путь `telegramAuthHandler` не трогаем).
   - На уровне БД `email` — nullable (обязательность email на checkout — прикладная, не constraint).
5. **Миграция:** боевая клиентская база практически пуста → **данные не мигрируем**; схему пересобираем под phone-first (031 расширяем миграцией, не переписываем историю).

**Провайдер sms.ru — предпосылки (operator, внешний lead-time, стартовать параллельно с кодом):**
- Внести реквизиты; создать **буквенного отправителя** (напр. `MURU`); **согласовать у операторов** (дни) — без него `sms/send` = ошибка **221**.
- Поднять **дневной лимит** (сейчас 0/10) под боевой трафик; пополнить баланс.
- Секреты в env: `SMSRU_API_ID`, `SMSRU_SENDER`, `SMSRU_TEST_MODE` (dev/staging = `test=1`, без списания/без реального отправителя). Ключ в чат/доки открытым текстом не писать (SET-001).

**Антифрод:** свой rate-limit по phone+IP + cooldown resend + лимит попыток ввода; передавать `ip` конечного пользователя в sms.ru; переиспользовать SmartCaptcha (порог как в W-SEC). Встроенные лимиты sms.ru (233/505/etc) — как второй эшелон.

**Reopen/следующее:** декомпозиция и статус фаз — в спеке эпика и PROGRESS.

---

## IMPORT-001 (2026-08-06) — Bulk-импорт товаров из шаблона (.xlsx) в админке

**Status:** Accepted (решения Василия 2026-08-06). **P1+P2 ACCEPT** @ `feature/import-p2-admin` `a9b40ac` (BE `d54f53c` + mig 041). Deploy **DEP-062 STOP** staging-first. Reports: P1/P2 в `.tasks/`.

**Что:** новая вкладка «Импорт» в разделе «Товары» админки: скачать шаблон .xlsx → заполнить → загрузить → предпросмотр → подтвердить → импорт; плюс лог импортов. Отдельный **чистый** шаблон, НЕ legacy google-формат (тот остаётся для google-sync).

**Колонки шаблона (только «база»; категории/подкатегории/фото/коллекции — из админки, НЕ из файла):**
| Колонка | Поле | Правило |
|---|---|---|
| Артикул* | `sku` | обязателен, уникален |
| Наименование* | `name` | обязателен |
| Стоимость, ₽* | `price` | LIST-цена (PRICE-001), RU-число `3 500,00` |
| Остаток* | `in_stock` | целое ≥ 0 (RU `2,00`→2) |
| Цвет | `color` | + дублировать в `specs['Цвет']` (как admin) |
| Размер | `size` | ярлык; + `specs['Размер']` |
| Описание | `description` | |
| Бренд | `specs['Бренд']` | |
| Материал | `specs['Материал']` | |
| Страна | `specs['Страна производитель']` | канон admin-ключ |
| Скидка % | `discount_percent` | 0–100 |

Вес/габариты в шаблон **не входят** → дефолты (3000г / 22×12×18), правятся в админке.

**Решения по логике:**
1. **Дубли SKU:** режим на выбор при импорте — `mode = new` (дубль → ошибка строки, существующий товар не трогаем) или `mode = upsert` (обновить поля существующего).
2. **Ошибки:** частичный импорт — валидные строки пишутся, невалидные → в лог с `row#` + причиной. Не атомарно на уровне файла.
3. **Процесс:** предпросмотр (`dryRun`) → подтверждение → импорт. Preview = импорт с `dryRun=true` (без записи).
4. **Лог импортов:** персистентный (кто/когда/файл/режим/counts/ошибки), отдельная вкладка.

**Технические инварианты:**
- RU-числа: нормализовать пробел-разряды и запятую-десятичный до записи.
- Формат файла — `.xlsx` (переиспользовать multer + xlsx-инфру; маппинг колонок — на базе `crm-catalog-sheet-map.ts`).
- Импорт использует существующий write-path создания/апдейта товара (валидации, транзакции), не пишет в БД в обход.
- Импортированные товары — без категории → категоризация/фото потом в админке.
- Не задевать google-sync legacy импорт и его формат.

