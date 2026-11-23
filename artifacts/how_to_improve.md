# Как улучшить систему: конкретный план

## Приоритеты изменений

Изменения ранжированы по ROI: максимальный эффект при минимальных усилиях.

---

## Приоритет 1 (КРИТИЧНО): Переписать промпт LLM для OCR режима

**Трудозатраты:** 2–4 часа (новый промпт + тестирование)
**Ожидаемый эффект:** OCR+LLM начнёт давать лучше результаты, чем чистый OCR

### Текущая проблема

Промпты `Direct Processing` и `OCR Result Processing` имеют **идентичные инструкции**: "Extract ONLY the defined fields". LLM в OCR-режиме переизвлекает всё заново вместо того чтобы корректировать OCR-ошибки. Результат: LLM копирует мусорные поля OCR и деградирует качество.

### Новая роль LLM: Linguistic Corrector

LLM должен получить чёткую роль: **не извлекать поля заново, а применить лингвистические знания там, где OCR ошибся**.

Обновить промпт `OCR Result Processing` в таблице `ocrdemo_prompts`:

```
You are a medical claim form data corrector. Your job is NOT to re-extract data —
Azure DI OCR has already extracted the fields. Your job is to APPLY YOUR GENERAL
LANGUAGE KNOWLEDGE to fix predictable OCR mistakes.

TASK 1 — COPY HIGH-CONFIDENCE FIELDS UNCHANGED (Confidence >= 0.85):
  Do NOT modify these values. OCR is correct. Copy exactly as provided.

TASK 2 — LINGUISTIC CORRECTION for LOW-CONFIDENCE FIELDS (Confidence < 0.85):
  Apply your language knowledge to fix OCR artifacts:

  Names (people and organizations):
    - Fix missing spaces: "AndreyKerchin" → "Andrey Kerchin"
    - Fix OCR character confusion: rn→m, cl→d, 0→O, l→I, li→h
    - Example: "MANCHASTER MEMCRIA HOSFT" → "MANCHESTER MEMORIAL HOSPITAL"
    - Example: "CTR STROPHER RUMPLE" → "DR. CHRISTOPHER RUMPLE"
  Addresses:
    - Fix garbled street names when the correction is obvious
    - Example: "71 HAYKES TEIN" → "71 HAYES STREET"
    - Example: "SUFF ELL" → "SHEFFIELD"
  Dates:
    - Remove OCR noise: "09 25 1963M." → "1963-09-25"
    - Normalize to YYYY-MM-DD when unambiguous

TASK 3 — CHECKBOXES:
  OCR tokens ":selected:" and ":unselected:" are unreliable.
  Determine checkbox state from image only.

TASK 4 — "SAME" RESOLUTION:
  If insured address = "SAME", copy the patient address values.

DO NOT TOUCH — copy exactly as OCR provided, regardless of confidence:
  - NPI, EIN, Tax ID, CPT codes, ICD-10 codes, policy numbers, member IDs
  - ANY field that is a code or numeric identifier
  - ANY name/word where you are less than 95% confident in the correction

MISSING FIELDS: return null. Do not invent values.

OCR OUTPUT (with confidence scores):
{{Fields}}

{{UseImage}}

Return valid JSON following the schema.
```

### Как внедрить

1. Обновить запись в `ocrdemo_prompts` где `key_name = 'OCR Result Processing'`
2. Тест на 5–10 формах из TestClaims/ — сравнить с текущим результатом
3. После подтверждения качества — деплой

---

## Приоритет 2 (КРИТИЧНО): Temperature = 0 + переход на GPT-5-mini

**Трудозатраты:** 15 минут
**Ожидаемый эффект:** Детерминированные результаты + лучшая модель за меньшие деньги

Temperature:
```sql
UPDATE agents 
SET content = jsonb_set(content, '{temperature,Value}', '"0"')
WHERE agent_id IN (7, 16, 18, 19, 21);
```

Модель: заменить deployment GPT-4.1-mini → GPT-5-mini в Azure AI Foundry. GPT-4.1-mini retires не ранее октября 2026, но GPT-5-mini уже GA, лучше и дешевле ($0.25 vs $0.40 за 1M input tokens).

---

## Приоритет 3 (ВАЖНО): Исправить баг с {{OCR}} плейсхолдером

**Трудозатраты:** 30 минут
**Ожидаемый эффект:** LLM начнёт получать coordinate-based текст для пространственного понимания формы

### Текущее состояние

В агенте `OcrDemoProcessPageCreatePrompt` (agent_id=18):
- Код строит `promptOcr` — список OCR текста с координатами (sorted by Y, then X)
- Шаблон `OCR Result Processing` содержит только `{{Fields}}` и `{{UseImage}}`
- `{{OCR}}` в шаблоне нет → coordinate data молча выбрасывается

### Вариант A: Добавить `{{OCR}}` в шаблон

Добавить в промпт раздел пространственного контекста перед `{{Fields}}`:

```
SPATIAL LAYOUT (text sorted by position on page — use to understand form structure):
{{OCR}}

OCR OUTPUT (with confidence scores):
{{Fields}}
```

### Вариант B: Упростить

Если coordinate text не несёт ценности после исправления промпта (Приоритет 1), убрать построение `promptOcr` из агента 18 и упростить код.

---

## Приоритет 4 (ВАЖНО): Разделить поля по confidence в агенте 18

**Трудозатраты:** 2–4 часа
**Ожидаемый эффект:** LLM точно знает какие поля надёжны, а какие нет

Изменить `OcrDemoProcessPageCreatePrompt` (agent_id=18) — разделить `promptFields` по порогу confidence:

```csharp
var highConf = textFieldsTextArray.Where(f => f.Confidence >= 0.85).ToList();
var lowConf = textFieldsTextArray.Where(f => f.Confidence < 0.85).ToList();

promptFields.AppendLine("### HIGH CONFIDENCE (copy unchanged, do NOT modify):");
foreach(var f in highConf) 
    promptFields.AppendLine($"[{f.Key}] = {f.Value} (Confidence: {f.Confidence:F2})");

promptFields.AppendLine("\n### LOW CONFIDENCE (apply linguistic correction):");
foreach(var f in lowConf) 
    promptFields.AppendLine($"[{f.Key}] = {f.Value} (Confidence: {f.Confidence:F2}) <- verify/correct");
```

---

## Приоритет 5 (ЖЕЛАТЕЛЬНО): Перейти на GPT-5-mini (Azure AI Foundry GA)

**Трудозатраты:** 30 минут (смена deployment name в конфиге)
**Ожидаемый эффект:** +15–25% к точности на плохих сканах, снижение стоимости на ~37%

`OcrDemoImageToText` (agent_id=16) и `OcrDemoExecuteLlmPrompt` (agent_id=7) используют `llm_type=2` — это GPT-4.1-mini, модель на пути к retirement (не ранее октября 2026).

### Рекомендуемые модели (Azure AI Foundry, GA, апрель 2026)

| Сценарий | Модель | Vision | Structured Outputs | $/1M input / output |
|---|---|---|---|---|
| **Primary (80% страниц)** | **GPT-5-mini** | Да | Да (JSON Schema) | $0.25 / $2.00 |
| **Escalation (conf < 0.70)** | **GPT-5** | Да | Да | $1.25 / $10.00 |
| **Checkboxes / signatures** | **o4-mini** | Да | Да | $1.10 / $4.40 |
| Latest (registration required) | GPT-5.4-mini | Да (10M+ px) | Да | $0.75 / $4.50 |

### Почему GPT-5-mini

- Прямой наследник GPT-4.1-mini, рекомендован Microsoft для миграции
- **Дешевле**: $0.25 input vs $0.40 у GPT-4.1-mini (−37%)
- Vision + Structured Outputs API + 400K контекст
- Drop-in замена: тот же OpenAI SDK, та же API-структура

### Tiered architecture (оптимальная стоимость/качество)

```
80% pages (high confidence) → GPT-5-mini       ~$38/мес при 1000 стр/день
20% pages (low confidence)  → o4-mini (reasoning) ~$19/мес
                                         Итого: ~$57/мес
vs текущий GPT-4.1-mini для всех:               ~$62/мес (хуже качество, дороже)
```

---

## Приоритет 6 (ЖЕЛАТЕЛЬНО): Расширить обучающий датасет Custom Model

**Трудозатраты:** 2–4 дня (разметка + переобучение)
**Ожидаемый эффект:** +40–60% к качеству чистого OCR (Azure DI без LLM)

> **Важно:** Это улучшает качество OCR-фундамента — меньше мусорных полей поступает на вход LLM. Само по себе не решает проблему роли LLM (Приоритет 1), но оба улучшения вместе дадут максимальный суммарный эффект.

### Текущее состояние

Custom Neural Model `ClaimsBridgeProject` обучен на **10 формах** (EHN, FLU, IAG x2, SMR, AHA, CENT, BPAD, CVH, QUE). David Crozier протестировал **25+ пейеров** — большинство модель не видела.

### Целевой размер датасета

| Размер | Ожидаемое качество OCR |
|--------|----------------------|
| 10 форм (текущее) | Confidence 0.3–0.6 на знакомых пейерах, 0.004–0.06 на новых |
| 30 форм | Confidence 0.7–0.85 на большинстве стандартных форм |
| 50+ форм (разные пейеры) | Confidence 0.85–0.95, хорошая генерализация |

### Приоритет разметки

```
APA (American Plan Administrators) — confidence 0.04–0.06
UPC, UHPT — confidence не проверялось
YUZU — stuck in "OCR Prompting"
TPC, RCF, SAI — новые пейеры
ARR, SHN, REC, RUB, IDA, LGH — добавить из TestClaims/
```

### Шаги

```
1. В Azure DI Studio → ClaimsBridgeProject → Label data
2. Загрузить формы из TestClaims/ (они уже есть локально)
3. Запустить Auto label → проверить результаты → исправить ошибки
4. Нажать Train → обучение займёт ~30–60 минут
5. Протестировать → задеплоить новую версию модели
```

### Поля с систематически низким confidence (особое внимание при разметке)

- `sender` — поле пейера в правом верхнем углу. Текст варьируется от формы к форме.
- `diagnosis` — Box 21, ICD-10 коды. Мелкий шрифт, сложный layout.
- `billing-provider-info` — нижняя часть формы, часто с логотипами и мелким текстом.
- `row1-provider-id` — рендеринг провайдера, смешивается с NPI.

---

## Приоритет 7 (ЖЕЛАТЕЛЬНО): Добавить валидационный слой

**Трудозатраты:** 3–5 дней
**Ожидаемый эффект:** Автоматическое выявление ошибок до отправки пользователю

Создать агент `OcrDemoValidateResult` который после LLM проверяет:

```csharp
// NPI: ровно 10 цифр
var npiValid = Regex.IsMatch(npi, @"^\d{10}$");

// ICD-10: формат A00.0 или A00.00
var icdValid = Regex.IsMatch(code, @"^[A-Z]\d{2}\.?\d{0,4}$");

// Дата: валидная дата в разумном диапазоне
var dateValid = DateTime.TryParse(date, out var d) && d.Year > 1900 && d < DateTime.Now;

// Сумма: total_charge == sum(service_lines[].charges)
var chargesValid = Math.Abs(totalCharge - serviceLines.Sum(l => l.charges)) < 0.01;
```

Невалидные поля не удаляются — помечаются `"validation_error": "NPI must be 10 digits"` для manual review.

---

## Итоговая дорожная карта

```
Неделя 1 (Quick Wins):
  □ Переписать промпт OCR Result Processing → Linguistic Corrector   [Приоритет 1]
  □ Выставить temperature = 0                                         [Приоритет 2]

Неделя 2 (Core Improvement):
  □ Исправить баг {{OCR}} в шаблоне (или убрать)                     [Приоритет 3]
  □ Разделение полей по confidence в агенте 18                        [Приоритет 4]

Неделя 3 (Optimization):
  □ Тест GPT-4o-mini vs GPT-4.1-mini                                 [Приоритет 5]
  □ Расширить датасет Azure DI Custom Model                           [Приоритет 6]

Неделя 4 (Production Readiness):
  □ Валидационный слой (NPI, ICD-10, суммы)                          [Приоритет 7]
  □ A/B benchmark: новый pipeline vs старый vs Direct Only
  □ Демо David Crozier с улучшенными результатами
```

---

## Ожидаемые результаты после изменений

| Метрика | Сейчас (OCR+LLM) | После изменений |
|---|---|---|
| Accuracy patient_name | ~30% | >90% |
| Accuracy provider info | ~20% | >85% |
| Accuracy diagnosis codes | ~5% | >80% |
| Accuracy service lines | ~40% | >88% |
| Average confidence | 0.3 | 0.85+ |
| Стоимость за форму | ~$0.05 | ~$0.02–0.04 |

---

## Что сказать David Crozier

Его наблюдение точное и важное: текущий OCR+LLM режим действительно хуже, чем чистый OCR Azure DI. Корень проблемы — в промпте: LLM получает те же инструкции "извлеки поля заново", что и в Direct режиме. Вместо того чтобы применить своё лингвистическое знание для исправления OCR-ошибок, LLM принимает мусорные OCR-поля как достоверные данные и копирует их.

После переписки промпта OCR режим начнёт делать то, что умеет только LLM, но не умеет OCR:
- Исправлять OCR character confusion: "CTR STROPHER RUMPLE" → "DR. CHRISTOPHER RUMPLE"
- Восстанавливать адреса: "71 HAYKES TEIN" → "71 HAYES STREET"
- Нормализовать даты, интерпретировать чекбоксы, обрабатывать "SAME"
- При этом не трогать коды (NPI, ICD-10, CPT) — их OCR читает точнее LLM

Режим OCR+LLM должен стать **лучше** чем чистый OCR — особенно для форм с плохим качеством сканирования, рукописными элементами, и для высоконагруженных пайплайнов где нужна скорость и стоимость важна.
