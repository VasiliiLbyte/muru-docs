# MURU — Forward-Port Protocol

> ⚠️ **DEPRECATED с 2026-07-06.** Прод `/var/www/muru` теперь деплоится напрямую из канона `muru-backend-local` (cutover, `PROGRESS.md` → «Унификация бэкендов» U1-U4, `DEP-008`). `MURU_miniAPP` заморожен, форвард-порт между двумя репозиториями больше не нужен — есть только один репозиторий backend'а. Документ сохранён как историческая референс (полезен, если понадобится понять, как раньше синхронизировались два репо, или для аналогичной ситуации в будущем).

Синхронизация кода между **каноническим бэкендом** (`muru-backend-local`) и **прод-монорепо** (`MURU_miniAPP`) — использовалось до cutover 2026-07-06.

**Версия:** 2026-07-02 (последняя активная), deprecated 2026-07-06
**Связанные документы:** `[ORCHESTRATOR.md](ORCHESTRATOR.md)`, `[PROGRESS.md](PROGRESS.md)` (Pending deploy), `[DEPLOY.md](DEPLOY.md)`

---

## 1. Зачем это нужно

Сейчас два репозитория с общей историей, но разными ветками:


| Репо                 | Ветка                    | Роль                                  |
| -------------------- | ------------------------ | ------------------------------------- |
| `muru-backend-local` | `storefront-integration` | Канон: новые API, web-канал, CRM-ядро |
| `MURU_miniAPP`       | `master`                 | Прод на Beget VPS (`murushop.ru`)     |


Без протокола легко: пофиксить в одном репо и забыть зеркалировать в другом, или задеплоить на VPS то, чего нет в `MURU_miniAPP`.

---

## 2. Когда нужен Forward-Port


| Ситуация                                                                             | Направление                                              |
| ------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| Новая фича / фикс разработан в `muru-backend-local` и должна попасть в прод Mini App | **local → miniAPP**                                      |
| Срочный хотфикс сделан напрямую в `MURU_miniAPP` на VPS                              | **miniAPP → local** (сразу после коммита)                |
| Изменения затрагивают `backend/` **и** `frontend/` mini app                          | Forward-port **обоих** слоёв                             |
| Есть SQL-миграция для прод-БД                                                        | Forward-port код + запись в Pending deploy + `DEPLOY.md` |


## 3. Когда НЕ нужен

- Работа **только** в `muru-storefront` (витрина) — отдельный репо, отдельные промпты.
- Эксперименты в `muru-backend-local`, которые **ещё не готовы** к проду (web-канал до cutover).
- Обновление **только** `muru-docs` (PROGRESS, спеки).

---

## 4. Направления

### 4.1. local → miniAPP (основной поток)

```mermaid
flowchart LR
  A[muru-backend-local] -->|forward-port| B[MURU_miniAPP]
  B -->|git push| C[VPS deploy]
  C --> D[PROGRESS Pending deploy]
```



1. Фича verified в `muru-backend-local` (`tsc`, vitest, ручной сценарий).
2. Оркестратор выдаёт forward-port промпт (шаблон §7).
3. Исполнитель переносит **те же логические изменения** в `MURU_miniAPP`.
4. Verify в `MURU_miniAPP`.
5. Оркестратор обновляет `PROGRESS.md` → Pending deploy.
6. Василий: `git pull` + `deploy.sh` на VPS (см. `DEPLOY.md`).

### 4.2. miniAPP → local (хотфикс)

1. Хотфикс закоммичен в `MURU_miniAPP/master`.
2. **Немедленно** зеркалировать в `muru-backend-local` (ветка `storefront-integration`).
3. Иначе при следующей фиче в local — merge-конфликт или потеря хотфикса.

---

## 5. Чеклист Forward-Port

Перед закрытием задачи оркестратор проверяет:

- [ ] Список изменённых файлов в source-репо задокументирован в промпте
- [ ] Те же файлы (или эквивалентная логика) изменены в target-репо
- [ ] `tsc --noEmit` чисто в **обоих** репо (backend; + frontend если трогали mini app UI)
- [ ] Релевантные vitest прогнаны (payments, sync — если затронуты)
- [ ] Миграции: файл есть в обоих `backend/src/db/migrations/` (если применимо)
- [ ] `.env.example` обновлён в обоих репо (если новые переменные)
- [ ] `API_CONTRACT.md` обновлён (если менялся HTTP-контракт)
- [ ] Строка добавлена/обновлена в `PROGRESS.md` → **Pending deploy**
- [ ] Направление портирования указано в логе сессий

---

## 6. Сравнение diff (ветки расходятся)

Ветки **не идентичны**: `storefront-integration` содержит web-канал, которого ещё нет в `MURU_miniAPP/master`.

**Правило:** переносить **конкретное изменение задачи**, а не `git merge` вслепую.

### Полезные команды

```bash
# Список файлов в коммите-источнике
git -C "/Users/vasilii/Desktop/code /muru-backend-local" show --name-only HEAD

# Diff конкретного файла между репо (ручное сравнение)
diff "/Users/vasilii/Desktop/code /muru-backend-local/backend/src/..." \
     "/Users/vasilii/Desktop/code /MURU_miniAPP/backend/src/..."
```

### Типичные ловушки


| Ловушка                                        | Как избежать                                                |
| ---------------------------------------------- | ----------------------------------------------------------- |
| Скопировали web-only код в miniAPP до cutover  | Forward-port только то, что нужно **телеграм-проду сейчас** |
| Забыли frontend mini app при backend-изменении | Проверить `frontend/src/` в обоих монорепо                  |
| Разные пути импорта после рефактора            | Сверять по смыслу, не построчно вслепую                     |
| Миграция применена только на local БД          | Pending deploy + `psql` на VPS перед деплоем кода           |


---

## 7. Шаблон executor-промпта (Forward-Port)

Копировать в Plan mode. ID промпта: `YYYY-MM-DD-NN-fp`.

```markdown
## MURU Forward-Port — [ID: 2026-07-02-01-fp]

**Направление:** muru-backend-local → MURU_miniAPP
**Исходная задача:** PROGRESS.md → [раздел / промпт ID]

### Фаза A — уже сделано (source)
Репо: muru-backend-local, ветка storefront-integration
Коммит/файлы:
- backend/src/routes/orders.routes.ts — удалён POST /create
- frontend/src/lib/api.ts — удалён createOrder
Проверки source: tsc чисто (be+fe)

### Фаза B — зеркалирование (target)
Репо: MURU_miniAPP
Путь: /Users/vasilii/Desktop/code /MURU_miniAPP
Ветка: master

### Цель
Перенести те же изменения, что в фазе A. Не добавлять web-only код из storefront-integration.

### Файлы target (ожидаемые)
- [те же пути, что в фазе A]

### НЕ трогать
- web payment routes (/payments/web/*)
- миграция 014 (если не в scope этой задачи)

### Критерии готовности
- [ ] Логика идентична фазе A
- [ ] cd MURU_miniAPP/backend && npx tsc --noEmit — чисто
- [ ] cd MURU_miniAPP/frontend && npx tsc --noEmit — чисто (если frontend менялся)

### Отчёт оркестратору
- Diff-список файлов в target
- Вывод tsc
- Нужна ли запись/обновление Pending deploy (DEP-xxx)
```

Для направления **miniAPP → local** — поменять фазы A/B местами.

---

## 8. Пример (reference): удаление POST /orders/create

**Задача:** убрать небезопасный эндпоинт, принимавший клиентские цены.


|             | muru-backend-local | MURU_miniAPP                 |
| ----------- | ------------------ | ---------------------------- |
| Статус кода | ✅ verified         | ✅ verified                   |
| VPS         | —                  | ⏳ Pending deploy (`DEP-001`) |


**Изменения:**

- Backend: удалены роут и `createOrderHandler`
- Frontend: удалены мёртвые `createOrder`, `submitOrder` в CartContext
- **Не тронуты:** `draftPayloadSchema`, сервисный `createOrder` (нужны для draft/fulfill)

**Pending deploy:** `DEP-001` — `git pull` + `deploy.sh` на `/var/www/muru`.

---

## 9. Связь с Pending deploy

Любой forward-port, затрагивающий прод:

1. Оркестратор добавляет/обновляет строку в `PROGRESS.md` → **Ожидает деплоя**
2. После успешного VPS-деплоя Василий сообщает оркестратору → статус `deployed`, строка архивируется или помечается ✅

---

## Changelog


| Дата       | Изменение               |
| ---------- | ----------------------- |
| 2026-07-02 | Первая версия протокола |
| 2026-07-06 | Deprecated — cutover прода на канон, `MURU_miniAPP` заморожен, форвард-порт больше не применяется |


