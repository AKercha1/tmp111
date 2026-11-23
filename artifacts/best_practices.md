# Best Practices: OCR + LLM для медицинских форм

## Контекст

Этот документ описывает лучшие практики для построения pipeline по обработке медицинских страховых форм (CMS-1500 / HCFA) с использованием комбинации OCR и LLM. Основан на анализе текущей реализации, реальных результатах тестирования клиента, research papers 2024–2025 и Microsoft reference architectures для IDP (Intelligent Document Processing).

---

## Ключевые исследования (2024–2025)

| Работа | Главный вывод |
|---|---|
| "OCR Error Post-Correction with LLMs: No Free Lunches" (ACL 2025) | LLM post-correction без правильной архитектуры **деградирует** accuracy на -57%–74% |
| "Hybrid OCR-LLM Framework for Enterprise-Scale Document Extraction" (Oct 2025) | Table-based extraction даёт **54x** прирост производительности vs multimodal |
| "LMDX: Language Model-based Document Information Extraction" (ACL 2024) | Верифицируй каждое LLM-значение через bounding boxes — если не совпадает, это галлюцинация |
| "KAP: MLLM-assisted OCR Enhancement" (2025) | LLM должен трогать **только** поля с низким confidence; высокий confidence — pass-through |
| "Seeing is Believing? Mitigating OCR Hallucinations in MLLMs" (2025) | При visual degradation LLM переключается на linguistic priors — галлюцинирует "типичные" значения |
| "The Impact of Temperature on Extracting Information" (PMC 2025) | temperature=0 обязателен для extraction tasks |

---

## 1. Выбор правильного OCR движка

### Специализированные модели vs Generic

**Не делайте так (generic без настройки):**
```
Azure DI prebuilt-layout (без queryFields) → возвращает:
  sender, patients-address, billing-provider-info (generic KV pairs)
  Confidence: 0.004–0.06 для критических полей
```

> **Примечание:** Специализированного prebuilt для CMS-1500 в Azure DI нет — `prebuilt-healthInsuranceCard.us` это про карточку пациента, не про claim form. Правильный путь — **Custom Neural Model**, который уже реализован в проекте `ClaimsBridgeProject`.

**Главный рычаг качества — размер обучающего набора:**

| Размер датасета | Ожидаемый avg confidence |
|---|---|
| 10 форм (текущее) | 0.3–0.6 на знакомых пейерах, 0.004–0.06 на новых |
| 30 форм (разные пейеры) | 0.7–0.85 |
| 50+ форм | 0.85–0.95, хорошая генерализация |

---

## 2. Роль LLM в OCR pipeline: Correction, не Re-extraction

### Антипаттерн: дублирующая экстракция

```
OCR → LLM ("Извлеки поля") → результат
```

LLM выполняет ту же задачу что и OCR, получая при этом конфликтующий контекст.

### Правильный паттерн: targeted correction

```
OCR (высокое confidence) → LLM ("Проверь только эти поля")
OCR (низкое confidence)  → LLM ("Это ненадёжно, читай из картинки")
```

LLM добавляет ценность там, где OCR слаб:

| Задача | OCR | LLM |
|--------|-----|-----|
| Точное распознавание текста | Отлично | Хуже |
| Коррекция char confusion (rn→m, cl→d, 0→O) | Нет | Отлично |
| Восстановление имён ("CTR STROPHER"→"DR. CHRISTOPHER") | Нет | Отлично |
| Восстановление адресов ("HAYKES TEIN"→"HAYES STREET") | Нет | Отлично |
| Разделение склеенных слов ("AndreyKerchin"→"Andrey Kerchin") | Нет | Отлично |
| Понимание чекбоксов | Слабо | Хорошо |
| Нормализация дат (09 25 1963 → ISO) | Нет | Хорошо |
| Семантические связи полей | Нет | Хорошо |
| Инференс ("SAME" = скопировать адрес) | Нет | Хорошо |
| Валидация NPI (10 цифр) | Нет | Хорошо |
| Обработка "фоновых" меток форм | Плохо | Хорошо |

---

## 3. Confidence-based routing

Не все поля требуют LLM post-processing. Используйте confidence как фильтр.

```
Для каждого OCR-поля:
  if confidence >= 0.85:
    → принять значение напрямую (trusted)
  elif confidence >= 0.5:
    → передать LLM на проверку с контекстом из изображения
  elif confidence < 0.5:
    → передать LLM с указанием "OCR-значение ненадёжно, читай из картинки"
  if confidence = 0:
    → поле не найдено OCR, LLM должен найти из изображения
```

Пример промпта для correction mode:
```
TRUSTED OCR FIELDS (confidence >= 0.85, do not modify):
  patients-zip: 06078
  patients-dob: 09/25/1963

LOW CONFIDENCE FIELDS (confidence < 0.5, verify from image):
  sender: "AMERICAN PLAN ADMINISTRATORS PO BOX 4 !!" [conf: 0.06]
  insured-name: "GNE" [conf: 0.30]
  diagnosis: "ON IT IS" [conf: 0.004]

NOT FOUND BY OCR (extract from image):
  patient_sex
  insurance_type_checkbox
```

---

## 4. Spatial context: используйте координаты OCR

Azure DI возвращает bounding boxes для каждого слова. Это ценнейший контекст для LLM.

### Почему это важно для CMS-1500

CMS-1500 — это форма с фиксированной структурой. Позиция поля на странице определяет его смысл:
- Верхний левый угол (Box 1) = Insurance Type
- Box 2 = Patient Name
- Box 21 = Diagnosis Codes
- Нижняя правая часть = Billing Provider

### Как передавать пространственный контекст

```
TEXT LAYOUT (sorted by position):
(120, 45): MEDICARE  (170, 45): MEDICAID  (240, 45): TRICARE
(120, 55): [ ]        (170, 55): [ ]        (240, 55): [X]
(50, 120): PATIENT NAME (LAST, FIRST, MI)
(50, 135): OBERA        QUEEN
(430, 120): DATE OF BIRTH
(430, 135): 09/25/1963
```

Так LLM понимает что именно к какому полю относится, даже если OCR выдал его в другом порядке.

---

## 5. Prompt Engineering для медицинских форм

### Принципы

**1. Разделяй роли**: OCR как "источник позиций", LLM как "источник смысла"

**2. Давай явные инструкции о приоритетах источников**:
```
Priority rules:
1. HIGH-CONFIDENCE OCR values (>= 0.85) → USE DIRECTLY, do not change
2. Spatial layout → use to understand field context
3. Image → use for LOW-CONFIDENCE and MISSING fields only
4. Never hallucinate values not visible in the image
```

**3. Явно укажи что такое артефакты формы**:
```
Form artifacts to IGNORE (these are printed on the blank form, not filled in):
- "SAME" in address fields means "copy patient address"
- Checkbox labels: MEDICARE, MEDICAID, TRICARE, etc.
- Field labels in gray/small text
```

**4. Используй structured thinking**:
```
For each field:
1. Check if OCR extracted it with confidence >= 0.85
2. If yes → use OCR value
3. If no → locate the field visually using its position
4. Extract the handwritten/typed value (not the printed label)
5. Normalize to the required format
```

**5. Явная инструкция для checkboxes**:
```
Checkbox states from image:
- Filled/marked box = true
- Empty box = false
- "X" or checkmark = true
- Do NOT trust OCR ":selected:" / ":unselected:" labels
  (they are often wrong for checkboxes near text)
```

---

## 6. Temperature и параметры модели

| Параметр | Рекомендуемое значение | Почему |
|---|---|---|
| temperature | 0.0 | Детерминированная экстракция, повторяемые результаты |
| top_p | 1.0 | — |
| max_tokens | 2000–3000 | Достаточно для полной CMS-1500 формы |
| response_format | json_schema / json_object | Гарантированный JSON |

---

## 7. Выбор модели

| Задача | Рекомендуемая модель | Причина |
|---|---|---|
| Direct extraction (только изображение) | GPT-4o / GPT-4.1 | Лучшее vision понимание |
| OCR correction (текст + изображение) | GPT-4o-mini / GPT-4.1-mini | Достаточно для коррекции |
| Только текстовая нормализация (без изображения) | GPT-4.1-mini / Claude Haiku | Дёшево и быстро |

Для medical form extraction качество важнее стоимости — используйте GPT-4o или Claude Sonnet.

---

## 8. Hybrid architecture: "Trust but Verify"

Лучшая архитектура — не "OCR потом LLM", а **confidence-gated hybrid**:

```
┌─────────────────────────────────────────────┐
│  Input: CMS-1500 TIFF/PDF                  │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────▼──────────┐
        │ Azure DI             │
        │ Custom Neural Model  │
        │ ClaimsBridgeProject  │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ Confidence Router   │
        │ HIGH (≥0.85) →  ✓  │  ← принять напрямую
        │ MED  (0.5–0.85) → ? │  ← отправить LLM на verify
        │ LOW  (<0.5) →   ✗  │  ← отправить LLM с image
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ LLM (correction)    │
        │ Роль: исправлять    │
        │ низко-confidence    │
        │ и заполнять пустые  │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ Validation Layer    │
        │ - NPI format check  │
        │ - ICD-10 code check │
        │ - Date format check │
        │ - Required fields   │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │ Final CMS-1500 JSON │
        └─────────────────────┘
```

---

## 9. Валидация после LLM

LLM может галлюцинировать. После получения JSON применяйте детерминированную валидацию:

```
Required validations for CMS-1500:
- NPI: exactly 10 digits
- ICD-10: format A##.#### 
- CPT: 5 digits
- Dates: valid calendar dates
- ZIP: 5 or 9 digits
- State: valid 2-letter US state code
- Total charge: sum of service line charges ± tolerance
```

Невалидные поля → вернуть null + добавить в requires_review флаг, не заменять галлюцинацией.

---

## 9b. Dual-Extraction Cross-Validation (Microsoft IDP паттерн)

Microsoft's published Azure AI Document Processing Pipeline использует параллельную двойную экстракцию:

```
Extractor A: Azure DI fields output (быстро, дёшево, точно на стандартных полях)
Extractor B: LLM direct image extraction (медленнее, дороже)
        ↓
Agreement (оба источника совпадают) → высокая уверенность, pass-through
Disagreement → флаг на human review
```

Паттерн даёт ~87.5% field-level match rate в production, остальные 12.5% — human review. Это полностью исключает необходимость определять "кто прав": разногласие само по себе является сигналом.

Reference: [Azure AI Document Processing Pipeline](https://github.com/Azure/ai-document-processing-pipeline)

---

## 9c. Принцип LMDX: верифицируй каждое LLM-значение через bounding boxes

Из работы "LMDX: Language Model-based Document Information Extraction" (ACL 2024):

> Любое значение, которое LLM вернул и которое не соответствует ни одному bounding box в OCR output, является галлюцинацией и должно быть отброшено.

Практическое применение:
```
LLM вернул: billing_provider.npi = "1234567890"
Azure DI bounding boxes content: ["1234567890"] exists at (700,450)
→ VERIFIED ✓

LLM вернул: patient.sex = "F"  
Azure DI bounding boxes content: no text "F" found on page
→ HALLUCINATION — return null ✗
```

---

## 10. Benchmark и итерация

До выхода в production нужна оценочная система:

1. **Создать ground truth dataset**: вручную разметить 50–100 форм с правильными значениями
2. **Измерять field-level accuracy**: для каждого поля отдельно
3. **Измерять по типу документа**: разные пейеры → разное качество
4. **A/B test**: Direct LLM vs OCR+LLM vs Hybrid

Минимально приемлемое качество для production: 95% accuracy на критических полях (patient_name, diagnosis_codes, service_lines, billing_provider_npi).

---

## 11. Антипаттерны (что точно не делать)

Основаны на исследованиях и анализе текущей реализации:

| Антипаттерн | Почему плохо |
|---|---|
| Просить LLM "улучшить" или "исправить" OCR | LLM не умеет отличать OCR-ошибки от правильного текста, исправляет оба |
| Передавать все поля в LLM вне зависимости от confidence | LLM "улучшает" правильные поля, вводя ошибки |
| Temperature > 0 для extraction | Добавляет недетерминированность в структурированные данные |
| Required поля в JSON schema | LLM галлюцинирует значения вместо возврата null |
| Одна и та же инструкция для "извлечь" и "проверить" | LLM делает re-extraction вместо verification |
| Смешивать checkbox extraction с text extraction в одном промпте | Разные механизмы ошибок, нужны разные инструкции |
| Использовать vision LLM для всей страницы когда нужен только один регион | Attention dilution, галлюцинации на не-релевантных регионах |

---

## Ссылки

- [OCR Error Post-Correction with LLMs: No Free Lunches (ACL 2025)](https://arxiv.org/html/2502.01205v1)
- [Hybrid OCR-LLM Framework for Enterprise-Scale Document Extraction](https://arxiv.org/html/2510.10138v1)
- [LMDX: Language Model-based Document Information Extraction (ACL 2024)](https://aclanthology.org/2024.findings-acl.899.pdf)
- [KAP: MLLM-assisted OCR Text Enhancement for Hybrid Retrieval (2025)](https://arxiv.org/html/2503.08452v1)
- [Seeing is Believing? Mitigating OCR Hallucinations in MLLMs](https://arxiv.org/html/2506.20168v2)
- [The Impact of Temperature on Extracting Information (PMC 2025)](https://pmc.ncbi.nlm.nih.gov/articles/PMC11731902/)
- [Azure AI Document Processing Pipeline (Microsoft)](https://github.com/Azure/ai-document-processing-pipeline)
- [Azure AI Document Processing Samples](https://github.com/Azure-Samples/azure-ai-document-processing-samples)
- [ARGUS: Azure Document Intelligence + GPT](https://github.com/Azure-Samples/ARGUS)
- [From Manual Document Processing to AI-Orchestrated Intelligence (Microsoft Blog)](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/from-manual-document-processing-to-ai-orchestrated-intelligence/4498835)
