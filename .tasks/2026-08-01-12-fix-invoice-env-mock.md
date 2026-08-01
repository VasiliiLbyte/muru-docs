# MURU Executor — [ID: 2026-08-01-12]

**Репозиторий:** `muru-backend-local`  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `master` @ `f75656f` (или tip master)  
**Связь:** PROGRESS → DEP-055 / EPIC Settings; задача `.tasks/FIX-invoice-test-env-mock-runtime-config.md`  
**Режим:** Plan mode → approve → execute. **Деплой НЕ нужен.** Staging НЕ нужен.

### Цель
Починить красный BE-сьют: 4 падения в `invoice.service.test.ts` из‑за устаревшего мока `env` после введения `runtime-config` (эпик Settings). Прод-код не менять.

### Контекст (root cause)
Цепочка: `createInvoiceForCheckout` → `buildProviderData` → `getEffectiveConfig` → `buildEffectiveConfigFromRow` читает `env.cdek.env`.

Мок в тесте (строки ~11–16) не содержит `cdek` → `env.cdek === undefined` → TypeError.

Это **дефект тестового изолята**, не прод-баг (ручной QA DEP-055 GREEN).

Минимальный фикс — добавить в мок:

```ts
cdek: { env: 'test' },
```

`buildEffectiveConfigFromRow` дальше читает `env.cdek.*` / `env.yookassa.*`; доступ к отсутствующим полям на **существующем** объекте даёт `undefined` без throw. Поэтому одного `env` обычно достаточно.

**Если** после добавления `cdek.env` сьют всё ещё красный — дочитать `runtime-config.service.ts` (`buildEffectiveConfigFromRow`) и добавить в мок **ровно** недостающие поля с нейтральными значениями (ориентир: мок в `runtime-config.service.test.ts` или `provider-receipt.test.ts`). Не копировать весь реальный `env`.

Альтернатива только при упорных падениях (не предпочитать): `vi.mock('../runtime-config.service', …)` как в `provider-receipt.test.ts` — тоже только в этом тест-файле.

### Файлы
- **Трогать:** `backend/src/services/telegram/invoice.service.test.ts`
- **НЕ трогать:** любой рантайм (`.ts` без `.test`): `runtime-config.service.ts`, `provider-receipt.ts`, `invoice.service.ts`, `env.ts`, admin, SF, миграции.

### Критерии готовности (DoD)
- [ ] Diff **только** `invoice.service.test.ts`
- [ ] Ассерты теста (t.me normalize / reject invalid URL) **не** ослаблены и не удалены
- [ ] `npx vitest run src/services/telegram/invoice.service.test.ts` → 4/4 passed
- [ ] `npx vitest run` (полный сьют из `backend/`) → **0 failed** (ожидаемо ~562 passed | 1 skipped; число по факту)
- [ ] `npx tsc --noEmit` → зелёный
- [ ] Коммит в `master` (тестовая правка; **без** deploy). Сообщение в стиле репо, напр. `test(invoice): stub env.cdek for runtime-config chain`

### Проверка (из `backend/`)
```bash
cd "/Users/vasilii/Desktop/code /muru-backend-local/backend"
npx vitest run src/services/telegram/invoice.service.test.ts
npx vitest run
npx tsc --noEmit
git diff --stat
```

### Заметка (не в скоупе)
В stderr vitest может быть баннер `vestauth.com` — игнорировать, не устанавливать. Отдельно: `grep -rln vestauth node_modules/*/package.json` — опционально в отчёте.

### Отчёт оркестратору
1. Итоговые счётчики полного vitest (passed/failed/skipped) — факт.
2. Diff-stat (подтвердить: один файл).
3. SHA коммита на `master` (+ push, если попросили).
4. Если в мок ушло больше `cdek: { env: 'test' }` — список полей и почему.
5. Риски: нет (тест-only).

### Модель
Auto достаточен (1 файл, мок). Сильнее — по желанию.
