## MURU Executor — [ID: 2026-08-01-11]

**Репозиторий:** muru-backend-local  
**Путь:** `/Users/vasilii/Desktop/code /muru-backend-local`  
**Ветка:** `feat/settings-site-contacts`. НЕ merge/deploy.  
**Связь:** EPIC Settings → Часть **4B Admin**. Backend 4A **ACCEPT**.

### Цель
Вкладки «Доставка (СДЭК)» и «Оплата (ЮКасса / 54-ФЗ)». Секреты = статус-бейджи, не инпуты.

### API (готово)
База: `/api/crm/settings` (owner cookie)
- `GET /site` — полный DTO включая cdek*/yookassa* business fields (без секретов)
- `PUT /cdek` — body camelCase cdek-полей (full replace группы, как contacts)
- `PUT /yookassa` — `yookassaVatCode`, `yookassaVerifyIp`
- `GET /integrations-status` →  
  `{ cdekConfigured, yookassaConfigured, yookassaWebConfigured, yookassaShopId, yookassaWebShopId }`

Поля CDEK (сверить с `CdekSettingsInput` в site-settings.service.ts):  
`cdekEnv`, `cdekSenderCityCode`, `cdekSenderPostalCode`, `cdekSenderAddress`, `cdekSenderName`, `cdekSenderPhone`, `cdekTariffDoor`, `cdekTariffPvz`, `cdekDefaultWeightGrams`, `cdekDefaultLengthCm`, `cdekDefaultWidthCm`, `cdekDefaultHeightCm`

### Сделать
1. `settings-api.ts`: `updateCdekSettings`, `updateYookassaSettings`, `getIntegrationsStatus`
2. Types в `settings.ts`
3. `CdekSettingsPage.tsx`:
   - Блок статуса: «СДЭК ключи» ✓/✗ из `cdekConfigured`
   - Форма бизнес-полей; `cdekEnv` select test|production|пусто
   - Полный payload; toast; русские ошибки (в т.ч. 422 про production keys)
4. `YookassaSettingsPage.tsx`:
   - Статусы Mini App / Web + read-only shopId / webShopId
   - Редактируемые: vatCode, verifyIp (checkbox)
   - **Нет** инпутов secret/returnUrl
5. Routes `/settings/cdek`, `/settings/yookassa` под RequireOwner
6. SettingsHub: обе карточки → Link, убрать из soonItems

### НЕ трогать
- backend/ (кроме бага — лучше не)
- SF, Drive-скрипты
- Не показывать clientSecret/secretKey никогда

### DoD
- [ ] tsc -b зелёный
- [ ] Owner открывает обе страницы; save CDEK/YK
- [ ] Секреты только бейджи

### Отчёт
файлы, tsc, API paths использованные, подтверждение no secret inputs
