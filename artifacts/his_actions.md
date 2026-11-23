# David Crozier — Что он делал и что увидел

## Контекст

David Crozier — клиент со стороны ClaimsBridge (`dcrozier@claimsbridge.com`). Работал с PoV-системой в роли конечного пользователя — оценивал пригодность продукта для реального использования в обработке медицинских страховых заявок.

---

## Хронология активности

### Сессия 1 — 23 января 2026 (первое знакомство)

**Время:** 18:51–20:12 UTC (≈1.5 часа)

Загрузил **9 файлов** — базовый тестовый набор из знакомых форм ClaimsBridge:

| Файл | Страниц | Статус |
|------|---------|--------|
| 0009.tif | 1 | Completed |
| APA_251216_HC_S-0314.tif | 1 | Completed |
| BPAD_251215_HC_A-0009.tif | 1 | Completed |
| CENT_251211_HCM_E-00000007.tif | 11 | Completed |
| CVH_251215_HC_A-0009.tif | 1 | Completed |
| EHN_251201_HC_D-0105.tif | 1 | Completed |
| FLU_251205_HCM_A-00000001.tif | 2 | Completed |
| IAG_251211_HC_A-0001.tif | 1 | Completed |
| TCN_251212_HC_A-0012.tif | 1 | Completed |
| YUZU_251210_HCM_A-00000001.tif | 2 | Completed |
| YUZU_251210_HCM_A-00000006.tif | 2 | Completed |

Цель: ознакомиться с системой, посмотреть как работает базовое распознавание.

---

### Сессия 2 — 13 февраля 2026 (массовый тест)

**Время:** 20:01–21:10 UTC (≈1 час 10 минут)

Загрузил **47 файлов** от ~25 различных пейеров — полноценное стресс-тестирование на реальных production-данных:

Пейеры: APA, ARR, ASH, BPAD, CENT, CVH, EHN, FLU, HPI, IAG, IDA, LGH, QCP, QUE, RCFH, REC, RUB, SAI, SDH, SHN, TCN, TPC, UHPT, UPC, YUZU

Примечательно:
- Несколько файлов загружены **дважды подряд** (например, `shn_260211_hcm_a-00000001.tif` в 20:44 и 20:45) — явное сравнение результатов при повторном запуске
- Самые сложные файлы: `shn_260211_hcm_a-00000001.tif` (18 стр.), `uhpt_260212_hcm_a-00000003.tif` (11 стр.)
- Один файл застрял в статусе **`OCR Prompting`** — `YUZU_251210_HCM_A-00000006.tif`

---

### Сессия 3 — 17 февраля 2026 (целевая проверка)

**Время:** 13:45 UTC

Загрузил **1 файл**: `HCFA_RED_DROP2.tif` (1 стр., 15 628 байт)

Целенаправленная проверка конкретного документа после выводов из предыдущей сессии.

---

## Итоговые метрики

| Параметр | Значение |
|---|---|
| Всего сессий | 3 (+ 15 login-сессий итого) |
| Загружено файлов | 58 (уникальных — ~35) |
| Обработано страниц | ~120+ |
| Дней активности | 23 янв, 13 фев, 17 фев |
| Токены (gpt-4.1-mini) | 264K input + 566K output |
| Модель | gpt-4.1-mini |

---

## Что он увидел: прямой результат его тестирования

Дэвид обнаружил, что **чистый Azure DI OCR** (без LLM post-processing) даёт приемлемые результаты, а добавление LLM post-processing ухудшает их. Его вывод: Azure Document Intelligence + LLM = хуже, чем Azure Document Intelligence без LLM.

### Почему это происходит

Azure DI читает текст точно. LLM в текущей реализации получает те же инструкции "Извлеки поля заново", что и в Direct режиме — в итоге LLM принимает низкоуверенные OCR-поля как достоверные данные и копирует их ошибки вместо их исправления.

### Что Azure DI вернул для `apa_260212_hc_i-0109.tif` (сырые OCR поля)

| Azure DI поле | Значение | Confidence |
|---|---|---|
| sender | "AMERICAN PLAN ADMINISTRATORS PO BOX 4 !! ARNOLD, KD 21012" | 0.06 |
| patients-address | "1520 N ST" | 0.52 |
| insured-name | "GNE" | 0.30 |
| diagnosis | "ON IT IS" | 0.004 |
| billing-provider-info | "(866) 559 55- RASTRT EXECTICIT :R.HULUSI CONEJLIAN.8 PO BOX 844770" | 0.04 |

### Что LLM сделал с этими данными (OCR Result Processing)

LLM получил эти поля с промптом "Извлеки поля из CMS-1500" и воспринял их как достоверный источник:

| Поле результата | Что LLM вернул | Откуда ошибка |
|---|---|---|
| patient.last_name | "ARNOLD" ❌ | Взял из поля `sender` (там был адрес пейера) |
| patient.first_name | "KD" ❌ | Продолжение той же строки из `sender` |
| patient.city | "SUFF ELL" ❌ | OCR прочитал "SHEFFIELD" как "SUFF ELL" — LLM скопировал |
| patient.sex | null ❌ | LLM не нашёл в загрязнённых данных |
| insured.last_name | "GNE" ❌ | Скопировал обрезанное `insured-name = "GNE"` |
| insured.city | "SAME" ❌ | Скопировал метку формы буквально |
| insurance_plan_name | "AMERICAN PIAN ADMINISTRATORS" ❌ | OCR опечатка PLAN→PIAN, LLM скопировал |
| provider.name | "CTR STROPHER RUMPLE" ❌ | OCR разбил "DR CHRISTOPHER" → "CTR STROPHER", LLM не исправил |

Правильные значения: patient = OBERA QUEEN, city = SHEFFIELD, provider = DR CHRISTOPHER RUMPLE.

**Вывод Дэвида был абсолютно правильным**: LLM post-processing с текущим промптом ухудшает результаты OCR вместо их улучшения.
