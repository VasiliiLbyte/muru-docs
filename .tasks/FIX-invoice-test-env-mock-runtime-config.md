# FIX — invoice.service.test.ts: устаревший env-мок ломает сьют после runtime-config

**Target:** `muru-backend-local` (**site: backend**, только тест-файл)
**Base:** `master @ f75656f`
**Staging:** НЕ требуется — прод-код (`.ts` рантайма) НЕ меняется, правится только `*.test.ts`. Деплой не нужен.
**Приоритет:** закрыть до формального DONE эпика Settings (DEP-055) — сейчас master с красным сьютом.

---

## Проблема (root cause, уже локализован надзором)

Полный BE-сьют красный: `Test Files 1 failed | 84 passed (85)`, `Tests 4 failed | 558 passed | 1 skipped`. Все 4 падения — в `src/services/telegram/invoice.service.test.ts`, одна первопричина:

```
TypeError: Cannot read properties of undefined (reading 'env')
 ❯ buildEffectiveConfigFromRow src/services/runtime-config.service.ts:116:67
     dbEnv === 'production' || dbEnv === 'test' ? dbEnv : env.cdek.env   // ← env.cdek === undefined
 ❯ getEffectiveConfig            runtime-config.service.ts:153
 ❯ buildProviderData             yookassa/provider-receipt.ts:10
 ❯ createInvoiceForCheckout      telegram/invoice.service.ts:63
 ❯ invoice.service.test.ts:67
```

Эпик Settings добавил цепочку `createInvoiceForCheckout → buildProviderData → getEffectiveConfig → buildEffectiveConfigFromRow`, которая читает `env.cdek.env`. Но мок `../../utils/env` в `invoice.service.test.ts` (строки 11–16) урезан и НЕ содержит ветку `cdek`:

```ts
vi.mock('../../utils/env', () => ({
  env: {
    payments: { nativeEnabled: true },
    telegramProviderToken: 'test-provider-token',
    yookassa: { vatCode: 1 },
  },
}))
```

→ `env.cdek` = `undefined` → `undefined.env` кидает TypeError.

**Это дефект тестового изолята, НЕ прод-баг.** На проде `env.cdek` заполнен из реального env — поэтому checkout и ручной QA заказчика прошли зелёными честно. Прод-код корректен. Чинить надо только мок.

## Что сделать (одна правка, минимально-инвазивно)

В `src/services/telegram/invoice.service.test.ts`, в объект мока `env` (строки 11–16) добавить ветку `cdek` с минимально достаточным `env`-полем. В этом тесте `deliveryMode: 'pickup'` и все `cdek*`-поля входа = `null`, поэтому реальные значения габаритов/отправителя не важны — важно лишь, чтобы `env.cdek.env` не был `undefined`. Достаточно:

```ts
vi.mock('../../utils/env', () => ({
  env: {
    payments: { nativeEnabled: true },
    telegramProviderToken: 'test-provider-token',
    yookassa: { vatCode: 1 },
    cdek: { env: 'test' },
  },
}))
```

**Если** прогон после этого покажет, что `buildEffectiveConfigFromRow`/`buildProviderData` читает ещё какие-то `env.cdek.*` или `env.yookassa.*` поля за пределами `env` (напр. `env.cdek.env` уже хватает, но всплывёт `env.yookassa.secretKey` и т.п.) — дочитать `runtime-config.service.ts` (`buildEffectiveConfigFromRow`) и `yookassa/provider-receipt.ts` (`buildProviderData`) и добавить в мок РОВНО те поля, к которым идёт обращение в этой цепочке, с нейтральными тест-значениями. Не раздувать мок сверх необходимого, не копировать весь реальный `env`.

## Ограничения
- Трогать ТОЛЬКО `src/services/telegram/invoice.service.test.ts` (при крайней необходимости — мок env в этом же файле). НЕ менять рантайм-код (`runtime-config.service.ts`, `provider-receipt.ts`, `invoice.service.ts`, `env.ts`) — он корректен, менять его = маскировать неверным «фиксом» рабочую логику.
- НЕ ослаблять и не удалять сами ассерты теста (проверки нормализации t.me URL, reject невалидных URL) — они валидны, чинится только окружение мока.
- Не добавлять новые зависимости.

## DoD
- `npx vitest run src/services/telegram/invoice.service.test.ts` → 4/4 passed.
- `npx vitest run` (полный сьют) → 0 failed (ожидаемо ~562 passed | 1 skipped; число уточнить по факту, главное — 0 красных).
- `npx tsc --noEmit` → зелёный.
- Diff содержит ТОЛЬКО изменения в тест-файле invoice.
- Коммит в `master` (тестовая правка; деплой НЕ требуется — прод-артефакт не меняется).

## Вернуть в отчёте
- Итоговые счётчики полного сьюта (passed/failed/skipped) — фактический вывод vitest.
- Diff-стат (подтвердить: затронут только `invoice.service.test.ts`).
- SHA коммита в master.
- Если пришлось добавить в мок больше одного поля `cdek: { env: 'test' }` — перечислить какие и почему (к каким `env.*` шло обращение в цепочке).

## Заметка (не блокер, отдельно от фикса)
В stderr прогона vitest замечен посторонний баннер `⌁ auth for agents [www.vestauth.com]` — похоже, dev-зависимость печатает промо. НЕ переходить по домену, ничего по нему не устанавливать. При случае найти источник: `grep -rln vestauth node_modules/*/package.json`. К этому фиксу отношения не имеет.
