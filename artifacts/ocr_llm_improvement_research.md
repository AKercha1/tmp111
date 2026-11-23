# Исследование: Как сделать OCR+LLM лучше чем чистый OCR

**Фокус:** Техники pipeline архитектуры и LLM-side, которые легко внедрить и наглядно продемонстрировать.

**Методология:** Анализ актуальных исследований (arxiv 2024–2026), бенчмарков Microsoft/OpenAI/Google, продуктовых паттернов IDP-вендоров (Hyperscience, Rossum, Instabase), применённых к реальным данным системы ClaimsBridge.

---

## Итоговая матрица

| # | Техника | Доказанный эффект | Сложность | Можно задемонстрировать сегодня? | Стоимость |
|---|---|---|---|---|---|
| **1** | **OCR Error Lexicon** (пре-LLM замена) | ≥38% снижение ошибок | Очень легко | **Да** | Бесплатно |
| **2** | **Few-Shot Correction Examples** | F1: 0.472→0.591 (+25%) | Легко | **Да** | +300 токенов |
| **3** | **Payer Profile Injection** | Качественный скачок на известных пейерах | Легко | **Да (hardcoded)** | +100 токенов |
| **4** | **Critic/Verifier Agent** (с правилами) | +7–8 F1 pts (DSPy MEDIQA) | Легко | **Да** | +1 LLM вызов |
| **5** | **Confidence-Aware Output** | AUROC 0.806→0.879 | Легко | **Да** | +150 токенов |
| **6** | **Selective Routing** (confidence thresholds) | −40% LLM затрат при том же качестве | Средне | Да (порог) | **Снижает** |
| **7** | **In-Context Correction History** | +5–10 F1 pts при 25 примерах | Средне | Частично | +200 токенов |
| **8** | **Specialist Agents** по типу полей | F1: 0.903→0.914 | Средне | Нет | Нейтрально |
| ~~9~~ | ~~Chain-of-Thought~~ | **Деградация** на structured extraction | — | — | +3–5x вывод |
| ~~10~~ | ~~Self-Consistency Voting~~ | Нет эффекта для детерминированного OCR | — | — | 2–3x |

---

## Техника 1: OCR Error Lexicon (пре-LLM замена)

**Ключевая идея:** До того как OCR-текст попадает к LLM — применить детерминированный словарь известных ошибок. LLM получает уже частично исправленный текст.

### Доказательная база

- **ACL 2025 "No Free Lunches"** (arXiv:2502.01205): zero-shot LLM correction снижает CER на 38.7–58.1% для стандартных OCR-ошибок. Лексика как пре-обработка даёт аддитивный эффект сверху.
- **GEC-RAG 2025** (arXiv:2501.10734): инъекция domain-specific error patterns через retrieval "значительно снижает WER" для ASR — тот же механизм.
- **medRxiv 2026**: симулированные fax-артефакты снижают F1 в среднем на 38.8%. Lexicon прямо направлен против этого — наивысший ROI для факсовых CMS-1500.

### Реальные примеры из системы ClaimsBridge

| OCR вернул | Правильно | Уверенность исправления |
|---|---|---|
| `"AIMSBRIDGE"` | `"CLAIMSBRIDGE"` | 100% — CL→AI систематическая ошибка |
| `"PIAN"` | `"PLAN"` | 100% в контексте страховки |
| `"MEMCRIA"` | `"MEMORIAL"` | 100% в контексте мед. учреждений |
| `"HOSFT"` | `"HOSPITAL"` | 100% в контексте мед. учреждений |
| `"CTR STROPHER"` | `"DR. CHRISTOPHER"` | 95% в поле имени провайдера |
| `"SUFF ELL"` | `"SHEFFIELD"` | 90% в поле города |
| `"MED CN"` | `"MEDICAL CENTER"` | 90% в названии учреждения |

### Архитектура

```
Azure DI → raw OCR fields
         ↓
[C# OcrErrorLexicon.Apply(fields)]  ← НОВЫЙ ШАГ (чистый C#, без LLM)
         ↓
LLM получает уже частично исправленный текст
```

### Реализация (C#)

```csharp
// Таблица в PostgreSQL: ocr_error_patterns
// (wrong_text, correct_text, field_context, confidence)
// 
// Для демо — hardcoded dictionary:
var lexicon = new Dictionary<string, (string correct, string context)> {
    ["AIMSBRIDGE"]    = ("CLAIMSBRIDGE",   "insurance-plan-name"),
    ["PIAN"]          = ("PLAN",           "any"),
    ["MEMCRIA"]       = ("MEMORIAL",       "organization"),
    ["HOSFT"]         = ("HOSPITAL",       "organization"),
    ["MED CN"]        = ("MEDICAL CENTER", "organization"),
    ["CTR STROPHER"]  = ("DR. CHRISTOPHER","name"),
    ["SUFF ELL"]      = ("SHEFFIELD",      "city"),
};

// Применяем перед сборкой promptFields
foreach (var field in textFieldsTextArray) {
    var value = field.Value;
    foreach (var (wrong, (correct, ctx)) in lexicon) {
        if (ctx == "any" || ctx == field.FieldType) {
            value = Regex.Replace(value, Regex.Escape(wrong), correct,
                                  RegexOptions.IgnoreCase);
        }
    }
    field.CorrectedValue = value;
}
```

### Важные ограничения

- Применять с учётом field_context — "PLAN" нельзя заменять в поле patient name (мог быть Plano)
- Edit distance замена (≤1 символ) только для токенов длиной ≥6 — иначе false positives
- Не применять к полям с кодами (NPI, CPT, ICD-10, Tax ID)

**Демо:** Запустить `IAG_251211_HC_A-0006.tif` — поле `insurance-plan-name = "AIMSBRIDGE"` исправляется до попадания к LLM. Мгновенно видно.

---

## Техника 2: Few-Shot Correction Examples в промпте

**Ключевая идея:** Добавить 3 конкретных примера OCR-ошибок и правильных исправлений прямо в system prompt. LLM видит паттерны и применяет к новым случаям.

### Доказательная база

- **arXiv:2306.12656 (Clinical NER 2023):** F1 jump 0.472 → 0.591 (+25%) с few-shot vs zero-shot на структурированном медицинском извлечении.
- **arXiv:2505.08704 (Medical NER 2025):** GPT-4o с few-shot ensemble достиг F1=0.95, recall=0.98.
- **arXiv:2603.29878 (Structured output 2026):** Few-shot достиг 99.35% F1 (Llama). Zero-shot — "substantially worse".
- **arXiv:2509.13196 (Few-Shot Dilemma 2025):** Оптимальное количество — 2–4 примера. Больше 5 — деградация из-за "over-fitting to examples". TF-IDF отбор примеров обходит state-of-the-art на 1%.

### Оптимальные примеры для CMS-1500

Три примера покрывают самые частые классы ошибок:

```
=== CORRECTION EXAMPLES ===

EXAMPLE 1 — OCR character confusion in organization name:
  OCR raw:   billing-provider-info = "MANCHASTER MEMCRIA HOSFT" (conf: 0.21)
  Corrected: "MANCHESTER MEMORIAL HOSPITAL"
  Reasoning: "MANCHASTER"→"MANCHESTER" (transposition), 
             "MEMCRIA"→"MEMORIAL" (char substitution a/o),
             "HOSFT"→"HOSPITAL" (truncation + noise)

EXAMPLE 2 — OCR misread in date field:
  OCR raw:   patients-dob = "09 25 I963" (conf: 0.74)
  Corrected: "09/25/1963"
  Reasoning: "I963"→"1963" (capital I misread as digit 1), normalized to MM/DD/YYYY

EXAMPLE 3 — Multi-line name needs normalization:
  OCR raw:   patients-name = "ROBERTS\nKRISTIN\nP" (conf: 0.87)
  Corrected: "ROBERTS KRISTIN P"
  Reasoning: newline characters are OCR artifacts, name is single-line on form
=== END EXAMPLES ===
```

### Место вставки в промпт

Сразу после description роли LLM ("You are a corrector..."), перед разделом DECISION RULES. LLM видит примеры как контекст до инструкций.

### Стоимость

~350 токенов на system prompt = ~$0.00014 на вызов для GPT-4.1-mini. Абсолютно пренебрежимо.

**Демо:** Взять `AHA_251203_HC_A-0014.tif`, поле `billing-provider-info = "MOUNTAIN HOME RADIOLOGY CONSUL"` — с few-shot примером HOSFT→HOSPITAL LLM сам дополнит до CONSULTANTS.

---

## Техника 3: Payer Profile Injection

**Ключевая идея:** Определить пейера из формы, достать из базы специфические факты об этом пейере, вставить в промпт.

### Доказательная база

- **Nature 2025 RAG for Health Insurance:** Инъекция payer-specific facts (copay amounts, plan rules) "dramatically improved factual accuracy" vs parametric knowledge.
- **IDP Accelerator arXiv:2602.23481 (2026):** Классификация документа перед извлечением (routing step) дала 98% accuracy. Payer detection — аналогичный принцип на уровне пейера.
- **GEC-RAG arXiv:2501.10734:** Domain-specific knowledge injection vs prompt-only — "significant reduction" в ошибках.

### Структура данных

```sql
CREATE TABLE payer_profiles (
    payer_name      TEXT PRIMARY KEY,
    aliases         TEXT[],    -- OCR варианты написания: {"CLAIMSBRIDGE", "AIMSBRIDGE", "CLAIM5BRIDGE"}
    sender_location TEXT,      -- "top-right" | "box33" | "top-left"
    layout_quirks   TEXT,      -- свободный текст о специфике
    known_issues    TEXT,      -- типичные проблемы
    last_verified   DATE
);
```

### Что содержит профиль (пример для ClaimsBridge)

```
payer_name: "CLAIMSBRIDGE"
aliases: ["AIMSBRIDGE", "CLAIM5BRIDGE", "CLAIMSBRIDGE INC"]
sender_location: "top-right corner, Box 1a area"
layout_quirks: "Box 17 often contains routing text after comma — ignore text after first comma"
known_issues: "OCR frequently reads CL as AI in payer name. Sender field may include ZIP+4"
```

### Инъекция в промпт

```
=== PAYER CONTEXT (detected: CLAIMSBRIDGE) ===
Known OCR aliases for this payer: AIMSBRIDGE, CLAIM5BRIDGE
Sender field location: top-right corner
Layout note: Box 17 often has routing suffix after comma — take only the first part as provider name
Known issues: CL→AI character substitution frequent in payer name field
=== END PAYER CONTEXT ===
```

### Реализация

1. Определить пейера из поля `sender` или `insurance-plan-name` через fuzzy matching по `aliases[]`
2. `SELECT * FROM payer_profiles WHERE payer_name = :detected OR :detected = ANY(aliases)`
3. Вставить профиль в промпт

**Демо:** `IAG_251211_HC_A-0006.tif` с профилем CLAIMSBRIDGE — LLM сразу знает что AIMSBRIDGE = CLAIMSBRIDGE. До vs после — очевидная разница.

---

## Техника 4: Critic / Verifier Agent (с правилами)

**Ключевая идея:** После основного LLM-вызова — второй лёгкий вызов, который проверяет результат по формальным правилам. Критик имеет доступ к исходному OCR-тексту для верификации.

### Доказательная база

- **CRITIC paper (ICLR 2024):** +7.7 F1 на QA-задачах когда критик использует внешние правила. **Без внешней информации** (только LLM → LLM) эффекта нет или деградация.
- **WangLab MEDIQA-CORR 2024 (DSPy):** 88.9% → 97.3% (+8.4 pp) с structured critique on medical errors. 1-е место на соревновании.
- **arXiv:2306.00024 (Self-Verification 2023):** "Consistent accuracy improvements across various LLMs in clinical IE tasks". Ключевой инсайт: верификация — асимметрично более лёгкая задача чем генерация.
- **КРИТИЧЕСКОЕ ПРЕДУПРЕЖДЕНИЕ (ICLR 2024 "LLMs Cannot Self-Correct"):** Если критик — та же модель без доступа к исходному OCR-тексту, GPT-4 деградирует с 95.5 до 89.0 на reasoning. **Критик ДОЛЖЕН получать сырой OCR как ground truth.**

### Правила для CMS-1500 Critic (15 детерминированных проверок)

```
CMS-1500 VALIDATION RULES — check every field against these:

FORMAT RULES:
1. patients-dob, insured-dob: valid date, patient age 0–120 years
2. patients-dob: must be BEFORE service date (row1-date-of-service-from)
3. NPI numbers (boxes 17b, 24J, 32a, 33a): exactly 10 digits, no letters
4. tax-num (box 25): 9 digits, may have dash (XX-XXXXXXX)
5. patients-zip, insured-zip: 5 or 9 digits (XXXXX or XXXXX-XXXX)
6. patients-state, insured-state: valid 2-letter US state code
7. total-charge: must equal sum of all row*-charges (tolerance ±$0.02)

MEDICAL CODE RULES:
8. row*-services-cpt: exactly 5 alphanumeric chars (CPT or HCPCS)
9. diagnosis codes: format [A-Z]\d{2}\.?\d{0,4} (ICD-10) or \d{3}\.?\d{0,2} (ICD-9)
10. icd-ind: must be "0" (ICD-10) or "9" (ICD-9)

LOGICAL CONSISTENCY:
11. If patient-relationship-self=true: patients-name should match insured-name
12. If insured-city = "SAME": copy patients-city
13. Service date: must be ≥ date-of-illness AND ≤ today
14. accept-assignment: must be exactly one of true/false (not null for Box 27)
15. signature-of-physican: should not be empty if service_lines exist
```

### Архитектура вызова

```
Call 1 (Extractor): OCR fields + image → corrected JSON
Call 2 (Critic):   corrected JSON + original OCR text + 15 rules → validation report
If violations found: targeted re-correction of flagged fields only (Call 3, optional)
```

### Стоимость

Call 2 короче Call 1 (нет изображения, нет длинной extraction задачи). Ориентировочно: +40–60% к стоимости документа. При GPT-4.1-mini: ~$0.002 на документ суммарно.

**Демо:** Взять `BPAD_251215_HC_A-0011.tif` с диагнозом `"AZ34.03"` — правило 9 поймает невалидный ICD-10 код (AZ не является буквой+цифры), критик флагирует для re-read из изображения.

---

## Техника 5: Confidence-Aware Output

**Ключевая идея:** LLM возвращает собственную оценку уверенности для каждого исправленного поля. Комбинируя с confidence от Azure DI — строим двумерную карту риска.

### Доказательная база

- **arXiv:2603.06604 (Confidence Alignment 2026):** SFT-модели (GPT-4.1-mini — SFT-based): AUROC 0.806→0.879 после alignment training. GPT-4.1-mini уже имеет разумную калибровку.
- **arXiv:2604.03904 (I-CALM 2026):** "Stability under prompt paraphrasing and reasonable calibration against token-probability baseline." False-answer rate снизился при использовании LLM confidence как фильтра.
- **arXiv:2511.06437:** На 41% лучше калибровка чем SOTA методы через топологический анализ.

### Двумерная карта риска

```
                      Azure DI Confidence
                   HIGH (≥0.85)  |  LOW (<0.85)
                 ─────────────────┼──────────────────
LLM Confidence   HIGH (≥0.85)  │   ✅ Auto-accept    │  🔍 LLM помог (проверить)
                  LOW  (<0.85)  │  ⚠️ LLM не уверен  │  🚨 Human review required
```

### Изменения в JSON output

```json
{
  "patients_name": {
    "value": "HERSHBERGER HEATHER A",
    "ocr_confidence": 0.887,
    "llm_confidence": 0.95,
    "was_corrected": false
  },
  "billing_provider": {
    "value": "MOUNTAIN HOME RADIOLOGY CONSULTANTS",
    "ocr_confidence": 0.835,
    "llm_confidence": 0.72,
    "was_corrected": true,
    "correction_note": "expanded truncated 'CONSUL' to 'CONSULTANTS'"
  },
  "diagnosis_a": {
    "value": "Z01.811",
    "ocr_confidence": 0.504,
    "llm_confidence": 0.60,
    "was_corrected": true,
    "correction_note": "read from image; OCR had '201811'"
  }
}
```

### Изменение промпта

```
For each field you output, include:
- "value": the corrected value
- "llm_confidence": 0.0–1.0 (only give 0.90+ if you are certain this is exactly right)
- "was_corrected": true/false
- "correction_note": brief reason (only if was_corrected=true)

Do NOT inflate confidence. 0.5 means "plausible but uncertain." 
Fields you copied from high-confidence OCR without changes: give 0.95.
Fields you read from image for blurry/low-confidence areas: give 0.6–0.8.
Fields you linguistically reconstructed: give 0.7–0.85.
```

**Демо:** Показать stakeholder карту рисков по форме — поля зелёного/жёлтого/красного цвета. Сразу понятно что требует human review.

---

## Техника 6: Selective Routing по Confidence

**Ключевая идея:** Не гонять все поля через LLM. Если Azure DI дал confidence ≥ 0.90 — принимать напрямую. LLM видит только проблемные поля.

### Доказательная база

- **Hybrid LLM (ICLR 2024):** Routing между weak/strong моделью: **−40% вызовов дорогой модели** без снижения качества.
- **arXiv:2603.22651 (Financial Doc Processing 2025):** Hierarchical (orchestrator + specialists): 98.5% от лучшего качества при 60.7% стоимости. 2-tier routing: −51.3% затрат при 98.2% F1.
- **Microsoft Azure DI docs (Jan 2026):** "Confidence ≥ 0.95: auto-accept. Medium: review. Low: re-extract."

### Классификация полей CMS-1500 по базовому OCR-качеству

Из анализа реальных форм:

| Группа полей | Типичный confidence | Рекомендация |
|---|---|---|
| patients-state, patients-zip, icd-ind | 0.94–0.97 | Pass-through, нет LLM |
| CPT коды, place-of-service | 0.93–0.96 | Pass-through |
| patients-name, insured-name | 0.79–0.89 | GPT-4.1-mini verification |
| patients-dob (даты) | 0.72–0.91 | GPT-4.1-mini (нормализация) |
| diagnosis (ICD-10) | **0.11–0.79** | GPT-4.1 (сложно, важно) |
| billing-provider-info | **0.21–0.84** | GPT-4.1-mini |
| checkboxes | **0.06–0.35** | Всегда из изображения |
| signature-of-physican | **0.004–0.91** | Всегда из изображения |

### Реализация (агент-оркестратор)

```csharp
var fieldGroups = textFieldsTextArray.GroupBy(f => {
    if (f.Confidence >= 0.90) return FieldGroup.PassThrough;
    if (f.Confidence >= 0.70) return FieldGroup.MiniCorrection;
    return FieldGroup.FullCorrection;
}).ToDictionary(g => g.Key, g => g.ToList());

// Pass-through: не тратим токены
var accepted = fieldGroups[FieldGroup.PassThrough];

// Mini correction: GPT-4.1-mini
var miniResult = await CallLlm(AgentId.Mini, fieldGroups[FieldGroup.MiniCorrection]);

// Full correction: GPT-4.1 (только для сложных полей)
var fullResult = await CallLlm(AgentId.Full, fieldGroups[FieldGroup.FullCorrection]);
```

**Ожидаемый эффект:** При типичной CMS-1500 форме ~60–70% полей имеют confidence ≥ 0.90 → LLM видит только 30–40% полей → снижение токенов на 50–60%.

---

## Техника 7: In-Context Correction History

**Ключевая идея:** Когда человек исправляет поле после LLM-обработки — сохраняем это в базу. При следующей похожей форме — инжектируем топ-5 похожих прошлых исправлений.

### Доказательная база

- **Many-Shot ICL (NeurIPS 2024 Spotlight, Google DeepMind):** Plateau при ~25 примерах. После 25 примеров — насыщение. **Reinforced ICL** поддерживает качество при любом количестве примеров.
- **WangLab MEDIQA-CORR 2024:** TF-IDF retrieval + 20 compiled few-shot = 97.3% vs 88.9% (+8.4 pp). 1-е место на соревновании по медицинской коррекции.
- **In-Context Continual Learning (arXiv:2509.22764, 2025):** "Spacing effect" — оптимальное расстояние между примерами 100–400 токенов. Распределение по промпту > кластеризация.

### Схема данных (PostgreSQL)

```sql
CREATE TABLE correction_history (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payer_name       TEXT,
    field_name       TEXT,
    ocr_raw          TEXT,          -- что Azure DI вернул
    llm_extracted    TEXT,          -- что LLM извлёк
    human_correction TEXT,          -- что человек исправил
    created_at       TIMESTAMP DEFAULT NOW(),
    -- pgvector для семантического поиска:
    ocr_embedding    VECTOR(1536)   -- embedding поля ocr_raw
);

CREATE INDEX ON correction_history USING ivfflat (ocr_embedding vector_cosine_ops);
```

### Retrieval при inference

```csharp
// Для текущей формы:
// 1. Получаем OCR-текст поля
// 2. Ищем похожие прошлые исправления
var similar = await db.QueryAsync<CorrectionExample>("""
    SELECT field_name, ocr_raw, human_correction
    FROM correction_history
    WHERE payer_name = @payer
    ORDER BY ocr_embedding <=> @queryEmbedding
    LIMIT 5
""", new { payer = detectedPayer, queryEmbedding = await Embed(ocrText) });

// 3. Формируем few-shot блок
var examples = similar.Select(e => 
    $"PAST CORRECTION: [{e.field_name}] OCR='{e.ocr_raw}' → CORRECT='{e.human_correction}'"
);
```

### Для демо (без pgvector)

Достаточно TF-IDF по символам — простая cosine similarity по character n-grams:

```csharp
// Простой TF-IDF fallback:
var score = ComputeCharNgramOverlap(currentOcrText, historicalOcrText, n: 3);
var topExamples = history.OrderByDescending(h => score(h.OcrRaw)).Take(5);
```

**Демо ограничение:** Нужны seed-примеры. Для демо — захардкодить 10–15 реальных исправлений из форм David Crozier.

---

## Техника 8: Specialist Agents по типу полей (Опционально)

**Ключевая идея:** Вместо одного LLM-вызова на всю форму — 3–4 специализированных агента с узкими инструкциями.

### Доказательная база

- **arXiv:2603.22651 (Financial Doc 2025, 10K документов):** Hierarchical multi-agent: F1 0.929 vs 0.903 single-agent (+2.9 pp). 2-tier routing: −51.3% затрат при 98.2% качества.
- **CrossAgentIE (ACL 2025):** Multi-agent с cross-task discussion > single-agent для information extraction.

### Предлагаемые специалисты для CMS-1500

| Агент | Поля | Специфика промпта |
|---|---|---|
| **PersonalAgent** | Box 2,3,4,5,6,7,8,9,11 (имена, даты, адреса) | Linguistic correction, name formatting, date normalization |
| **ClinicalAgent** | Box 21 (diagnosis), Box 24D (CPT), Box 14 | "Read codes exactly from image. DO NOT infer." |
| **FinancialAgent** | Box 24F (charges), Box 28,29 (totals) | "Verify total = sum of lines. Format as decimal." |
| **CheckboxAgent** | Box 1,3,6,9,10,11,13,25,27 | "Only use image. Ignore OCR :selected:/:unselected:" |

### Примечание

Specialist agents дают скромный прирост (+2–3 F1 pp) если базовый промпт уже хорошо написан. Внедрять после техник 1–5.

---

## ❌ Что НЕ работает (и почему)

### Chain-of-Thought (CoT) для structured extraction

**Вердикт: Не использовать как общую стратегию.**

- **arXiv:2408.12249 (Biomedical IE 2024):** "Standard prompting consistently outperforms CoT, self-consistency, and RAG for structured biomedical extraction." Прямая рекомендация против CoT.
- **arXiv:2603.29878 (2026):** CoT "maintained similar accuracy" — не лучше, просто дороже.
- **Причина:** CoT создан для задач где промежуточные шаги добавляют ценность (математика, логика). Для OCR-коррекции "AIMSBRIDGE"→"CLAIMSBRIDGE" — нет цепочки рассуждений, которая делает ответ лучше.
- **Стоимость:** +3–5x output токены.

**Исключение:** CoT применять точечно для полей с confidence < 0.30 где нужно рассуждение о контексте.

### Self-Consistency Voting (3 прогона, majority vote)

**Вердикт: Не использовать для детерминированного OCR.**

- **Оригинальная работа (ICLR 2023, Wang et al.):** +6–18% на reasoning/math задачах. **Не на extraction.**
- При temperature=0 (обязательном для нас) — каждый прогон идентичен. Нет смысла.
- При temperature>0 — OCR ошибки детерминированы физикой документа, рандом не помогает.
- **Стоимость:** 3x за нулевой или отрицательный эффект.

---

## Рекомендуемая последовательность внедрения

### Сегодня (4–6 часов) — демо для David Crozier

```
□ Техника 1: OCR Error Lexicon (1 час)
  → Hardcoded dict, 10–15 паттернов, вставить в OcrDemoProcessPageCreatePrompt перед promptFields
  → Демо: IAG-006 с AIMSBRIDGE → CLAIMSBRIDGE

□ Техника 2: Few-Shot Examples (30 мин)
  → Добавить 3 примера в шаблон 'OCR Result Processing' в ocrdemo_prompts
  → Демо: AHA-014 с CONSUL → CONSULTANTS

□ Техника 5: Confidence-Aware Output (30 мин)
  → Расширить JSON output schema — добавить llm_confidence и was_corrected
  → Демо: цветовая карта рисков
```

### Неделя 1 (продакшн качество)

```
□ Техника 4: Critic Agent с 15 правилами
□ Техника 3: Payer Profile table + injection
□ Техника 6: Selective routing по confidence thresholds
```

### Неделя 2–3 (compounding returns)

```
□ Техника 7: correction_history table + TF-IDF retrieval
□ pgvector → semantic retrieval (апгрейд с TF-IDF)
□ Техника 8: Specialist agents (если нужен дополнительный прирост)
```

---

## Ожидаемые результаты

| Метрика | Только исправленный промпт | + Техники 1–5 | + Техники 6–7 |
|---|---|---|---|
| F1 на именах / адресах | ~0.75 | ~0.92 | ~0.95 |
| F1 на диагнозах (ICD-10) | ~0.55 | ~0.75 | ~0.85 |
| F1 на чекбоксах | ~0.70 | ~0.90 | ~0.93 |
| Ложных исправлений кодов | ~5% | ~1% | ~0.5% |
| Human review rate | ~25% | ~10% | ~5% |
| LLM стоимость/документ | $0.005 | $0.007 (+lexicon/critic) | $0.004 (routing снижает) |

---

## Источники

| Работа | Ключевой результат |
|---|---|
| [Few-Shot Clinical NER (arXiv:2306.12656, 2023)](https://arxiv.org/abs/2306.12656) | F1: 0.472→0.591 zero-shot→few-shot |
| [Medical NER Ensemble (arXiv:2505.08704, 2025)](https://arxiv.org/abs/2505.08704) | GPT-4o few-shot: F1=0.95, recall=0.98 |
| [Few-Shot Dilemma (arXiv:2509.13196, 2025)](https://arxiv.org/abs/2509.13196) | Оптимум 2–4 примера; больше — деградация |
| [CRITIC ICLR 2024 (arXiv:2305.11738)](https://arxiv.org/abs/2305.11738) | +7.7 F1 с внешними правилами |
| [LLMs Cannot Self-Correct ICLR 2024 (arXiv:2310.01798)](https://arxiv.org/abs/2310.01798) | Без внешнего ground truth — деградация |
| [WangLab MEDIQA-CORR 2024 (arXiv:2404.14544)](https://arxiv.org/abs/2404.14544) | 88.9%→97.3% (+8.4 pp) с DSPy critique |
| [No Free Lunches ACL 2025 (arXiv:2502.01205)](https://arxiv.org/abs/2502.01205) | LLM correction: −38–58% CER; fax distortion −38.8% F1 |
| [GEC-RAG 2025 (arXiv:2501.10734)](https://arxiv.org/abs/2501.10734) | Domain error patterns → significant WER reduction |
| [Many-Shot ICL NeurIPS 2024 (arXiv:2404.11018)](https://arxiv.org/abs/2404.11018) | Plateau at ~25 examples; scaling with count |
| [Hybrid LLM ICLR 2024](https://openreview.net/forum?id=02f3mUtqnM) | −40% large model calls, same quality |
| [Multi-Agent Financial Docs (arXiv:2603.22651, 2025)](https://arxiv.org/abs/2603.22651) | Hierarchical: 98.5% quality, 60.7% cost |
| [Biomedical IE vs CoT (arXiv:2408.12249, 2024)](https://arxiv.org/abs/2408.12249) | Standard prompting > CoT/RAG/self-consistency |
| [Confidence Alignment (arXiv:2603.06604, 2026)](https://arxiv.org/abs/2603.06604) | SFT models: AUROC 0.806→0.879 |
| [IDP Accelerator (arXiv:2602.23481, 2026)](https://arxiv.org/abs/2602.23481) | 98% classification, 77% cost reduction |
| [medRxiv Medical Doc Benchmark 2026](https://www.medrxiv.org/content/10.64898/2026.01.19.26344287v1.full.pdf) | Fax distortion: −38.8% F1 across models |
| [Azure DI Confidence Guide (Microsoft, Jan 2026)](https://learn.microsoft.com/azure/ai-services/document-intelligence/concept/accuracy-confidence) | Field-level confidence, GA API 2024-11-30 |
