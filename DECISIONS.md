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
