# STOP-4 — Staging-first checklist (EPIC Settings Часть 4)

**CLOSED 2026-08-01** — DEP-055 deployed (BE `f75656f`, SF `4795b25`). Ниже — исторический чеклист.

Код 4A+4B **ACCEPT**. Merge в `master` / прод-деплой — **только после** зелёного staging.

## Prefetch (ветки)
- BE+admin: `feat/settings-site-contacts` (не merge)
- SF: `feat/settings-site-contacts-sf` (части 1C/2C/3C; можно мержить отдельно от 4 — не зависит от 036)

## Staging backend (`api-staging.murushop.ru`)
1. Закоммитить feature (без Drive-скриптов `backend/scripts/scan-drive-*`).
2. Deploy на staging по `DEPLOY.md` §2a (`deploy-staging.sh` / checkout feature или cherry-pick).
3. Миграции на `muru_staging`: **035** (если ещё нет) → **036**.
4. Smoke:
   - `GET /api/health` 200
   - `GET /api/content/site-contacts` 200
   - `GET /api/content/requisites` 200
   - Owner CRM: save CDEK (не production без ключей) / YK vat
   - `GET /api/crm/settings/integrations-status` — только булевы + shopIds
   - CDEK: `POST /api/cdek/web/calculate` (или calc path) с тестовым городом — цена OK
   - YooKassa web: тестовый платёж до return URL (как e2e 2026-07-03)
5. Mini App smoke на staging/prod не ломать (канал telegram).

## Откат
- down 036 на staging при регрессии; или revert deploy SHA.

## После зелёного staging
- FF/merge `feat/settings-site-contacts` → `master` (Vasilii gate)
- Prod: migrate 035+036 → `deploy.sh`
- SF merge `feat/settings-site-contacts-sf` → `main` + VPS rebuild (можно раньше 4-го деплоя)
- Docs: `DECISIONS.md` ADR secrets-A + content_pages legal; `API_CONTRACT.md` public endpoints; `PROGRESS` DEP-xxx

## НЕ делать сейчас
- Прод без staging smoke
- Перенос секретов в БД
- Forward-port в замороженный MURU_miniAPP до решения (канон = muru-backend-local)
