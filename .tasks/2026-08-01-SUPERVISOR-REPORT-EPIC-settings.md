# Глобальный отчёт → надзорный оркестратор

**Epic:** Settings — CDEK · ЮKassa · Документы · Контакты · Реквизиты  
**Исполнитель (working orchestrator):** Cursor / muru-docs  
**Заказчик / gate:** Vasilii  
**Дата закрытия:** 2026-08-01  
**Вердикт:** **ACCEPT + DEPLOYED + OPERATOR QA GREEN** (DEP-055)

Скопируй блок ниже во внешний чат надзора как self-contained brief.

---

```text
# SUPERVISOR BRIEF — EPIC Settings CLOSED (DEP-055)

## 1. Вердикт (одна строка)
EPIC Settings (контакты, юр. документы, реквизиты, CDEK/YK non-secret settings, runtime-config) реализован, прошёл STOP-гейты, задеплоен на staging→prod, ручной smoke заказчика GREEN. Код закрыт. Следующий крупный трек — RF CDN cutover (не этот epic).

## 2. Scope vs НЕ-цели (соблюдено)
Сделано:
- Admin SPA Settings hub: Contacts, Documents, Requisites, CDEK, YooKassa (owner-only).
- Backend: site_settings singleton + public content endpoints + CRM settings API + runtime-config (DB ?? env).
- Storefront: contacts/requisites/legal/help pages читают API с per-field static fallback.
- Secrets variant A: CDEK/YK secrets остаются в env; в API/admin JSON не отдаются.

НЕ трогали (по ТЗ):
- Mini-App legacy admin (frontend/src/admin)
- Перенос секретов в БД
- Phase W / auth / catalog / orders redesign
- Вёрстка SF «с нуля» — только источник данных
- Forward-port в замороженный MURU_miniAPP

## 3. Архитектура (locked decisions)
| ID | Решение | Где |
|----|---------|-----|
| SET-001 | Secrets = env only; non-secrets = site_settings + runtime-config | DECISIONS.md |
| SET-002 | Legal docs = content_pages + LEGAL_DOC_SLUGS (не новая таблица) | DECISIONS.md |
| Owner-gate | Все CRM settings — как /crm/users (RequireOwner + BE guard) | код |
| Additive API | Public GET /api/content/site-contacts, /requisites | API_CONTRACT.md |

Миграции:
- 035_site_settings.sql — singleton contacts+requisites (+ social stubs)
- 036_settings_cdek_yookassa.sql — non-secret CDEK/YK columns

## 4. Delivery map (части → STOP)
| Часть | Содержание | Результат |
|-------|------------|-----------|
| 1A/1B/1C | Contacts BE + admin + SF | ACCEPT |
| 2A/2B/2C | LEGAL_DOC_SLUGS + Documents admin + SF /help|/legal | ACCEPT |
| 3A/3B/3C | Requisites BE + admin + SF table | ACCEPT |
| 4A/4B | runtime-config + CDEK/YK admin | ACCEPT |
| STOP-4 | Staging-first → merge → prod | CLOSED |

Task files: muru-docs/.tasks/2026-08-01-01 … 11, EPIC-settings-*.md, STOP4-staging-checklist.md (CLOSED).

## 5. Git tips (канон)
| Репо | Ветка | SHA | VPS path |
|------|-------|-----|----------|
| muru-backend-local | master | f75656f | /var/www/muru |
| muru-storefront | main | 4795b25 | /var/www/muru-storefront |

Feature branches (historical, merged):
- feat/settings-site-contacts → master (FF)
- feat/settings-site-contacts-sf → main (FF 459eb90..4795b25)

Не коммитить / не деплоить: backend/scripts/scan-drive-* (Drive audit junk).

## 6. Deploy evidence (staging → prod)
Staging (/var/www/muru-staging, muru_staging):
- migrate 035 → 036
- deploy-staging.sh
- smoke: health, site-contacts, requisites, CRM 401, CDEK calc validation path

Prod (/var/www/muru, muru_db):
- git pull master → f75656f
- psql 035 + 036 (CREATE/INSERT/ALTER OK)
- bash deploy.sh (backend+miniAPP frontend+admin)
- muru-backend PM2 ↺ 12→13, online
- curl health 200; site-contacts JSON; requisites JSON; CRM /settings/site → 401

Storefront:
- Mac: FF merge feat → main, push 4795b25
- VPS: checkout main, pull, npm ci --include=dev, next build 48/48, omit=dev, pm2 restart ↺140
- Build-time 404 на content/pages/{delivery,consent,…} и пустые collections — ожидаемый CMS gap + static fallback (не регрессия runtime-config)

## 7. Инцидент на деплое (закрыт, урок)
На VPS в /var/www/muru ошибочно выполнены команды merge SF с Mac-путями и `git checkout main` (у GitHub muru-backend-local ветка main = устаревший канон ≠ master).
Исправление: `git checkout master && pull` → HEAD f75656f до повторного deploy.sh.
Урок надзору: на Beget канон backend = **master**; SF merge только локально в muru-storefront, на VPS только pull+build.

## 8. Operator QA (Vasilii, 2026-08-01) — GREEN
Проверено руками:
- Admin Settings: contacts / documents / requisites / CDEK / YooKassa — save+reload OK
- Storefront: contacts, requisites, help/legal routes, header/footer phone
- Web checkout: CDEK calc + YooKassa test card
- Mini App: каталог/корзина без регрессии

API spot-checks post-deploy:
- GET https://murushop.ru/api/content/site-contacts → 200
- GET https://murushop.ru/api/content/requisites → 200
- GET https://murushop.ru/api/crm/settings/site (no cookie) → 401

## 9. Docs sync (muru-docs)
- PROGRESS.md: epic CLOSED, tip SHAs, DEP-055 row, session log, operator QA GREEN
- API_CONTRACT.md: § site-contacts / requisites (v 2026-08-01)
- DECISIONS.md: SET-001, SET-002
- .tasks/STOP4 checklist: CLOSED

## 10. Residual / next (НЕ блокеры epic)
Контент (операционка, не код):
- Наполнить legal slug'и в admin Documents (иначе SF остаётся на static fallback для delivery/refund/terms/consent)
- Поддерживать актуальные контакты/реквизиты в Settings

Hub «скоро» осталось:
- SEO
- Уведомления

Параллельный активный трек платформы (не Settings):
- RF-доступность / YC CDN cutover web.murushop.ru (DNS CNAME pending; см. PROGRESS RF-001…005)
- YC баланс < мин. пакет — флаг

## 11. Definition of Done checklist (надзор)
- [x] Части 1–4 ACCEPT (orchestrator verify)
- [x] STOP-4 staging-first
- [x] Merge master/main
- [x] Prod migrate 035+036
- [x] Prod BE + admin + SF online tips
- [x] API smoke
- [x] Operator manual QA GREEN
- [x] Docs (PROGRESS / API_CONTRACT / DECISIONS)
- [ ] Optional: создать CMS legal pages (контент)
- [ ] Optional: CDN cutover (другой epic)

## 12. Рекомендация надзору
Закрыть EPIC Settings как DONE. Не открывать регресс-тикеты по build-time CMS 404 при fallback. Следующий приоритет оркестрации — RF CDN cutover gate (деньги YC + DNS + тест заказчика из РФ без VPN), либо SEO/Уведомления в Settings hub по продуктовому приоритету.
```

---

## Краткий индекс для working orchestrator

| Артефакт | Путь |
|-----------|------|
| Этот отчёт | `.tasks/2026-08-01-SUPERVISOR-REPORT-EPIC-settings.md` |
| Epic ТЗ | `.tasks/EPIC-settings-cdek-yookassa-docs-contacts.md` |
| Progress | `PROGRESS.md` → DEP-055 / session 2026-08-01 |
| Decisions | `DECISIONS.md` SET-001, SET-002 |
| Contract | `API_CONTRACT.md` site settings public |
