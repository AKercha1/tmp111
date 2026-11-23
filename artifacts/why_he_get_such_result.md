# Почему OCR+LLM хуже чем чистый OCR

## Краткий ответ

LLM получил неправильную роль. В OCR-режиме ему дали те же инструкции что и в Direct-режиме: "Extract ONLY the defined fields". Вместо того чтобы **корректировать лингвистические ошибки OCR**, LLM пытается **заново извлечь все поля с нуля** — и при этом получает загрязнённый контекст с низкоуверенными OCR-полями как "достоверные данные". Результат: LLM копирует мусор OCR вместо его исправления.

---

## Root Cause #1: LLM получил роль Re-Extractor вместо Corrector

### Что происходит

Промпт в обоих режимах идентичен:

```
You are an AI system specialized in structured medical form extraction.
TASK: Extract ONLY the defined fields from a CMS-1500 (Health Insurance Claim Form).
```

В `OCR Result Processing` режиме LLM получает:
- Поля, извлечённые Azure DI (с confidence scores)
- Изображение формы

Но инструкции те же: "Извлеки поля". LLM не знает что его задача — **не извлечь заново**, а **принять OCR-поля с высоким confidence и лингвистически скорректировать поля с низким confidence**.

### Почему это проблема

Azure DI Custom Model (`ClaimsBridgeProject`) хорошо извлекает поля из знакомых форм — confidence 0.7–0.95 для форм из обучающего набора. Но LLM получает эти поля и переизвлекает их заново, конкурируя с OCR вместо того чтобы работать поверх него.

Пример для высокоуверенного поля:
```
Azure DI вернул: patients-zip = "06078" (confidence: 0.92)
LLM получает: "[patients-zip, (Confidence: 0.92)] = 06078"
LLM думает: "Мне сказали извлечь поля. Извлекаю..."
Результат: LLM может изменить "06078" — добавив шум туда, где OCR уже дал правильный ответ
```

---

## Root Cause #2: LLM слепо копирует OCR-мусор как достоверные данные

### Механизм отравления

Когда Azure DI не уверен в поле (confidence < 0.3), он возвращает то что видит в регионе — включая текст из соседних областей, артефакты печати, метки формы.

Промпт для OCR Result Processing отправляет LLM-у:

```
### OCR Recognized the following Fields:

[sender, (Confidence: 0.06)] = AMERICAN PLAN ADMINISTRATORS PO BOX 4 !!
ARNOLD, KD 21012
[insured-name, (Confidence: 0.30)] = GNE
[diagnosis, (Confidence: 0.004)] = ON IT IS
```

LLM видит "OCR Recognized the following Fields" и **трактует это как достоверные данные**, даже при confidence 0.06. Никаких инструкций о том, что низкая уверенность = ненадёжность. Никакого порога для отсева.

### Конкретные трассировки ошибок

| Ошибка в результате | Откуда взялась |
|---|---|
| `patient.last_name = "ARNOLD"` | Azure DI поле `sender` содержало `ARNOLD, KD 21012` — LLM распарсил это как имя пациента |
| `patient.first_name = "KD"` | Продолжение той же строки из `sender` |
| `patient.city = "SUFF ELL"` | Azure DI `patients-city = "SUFF ELL"` (искажение "SHEFFIELD") — LLM скопировал вместо исправления |
| `insured.last_name = "GNE"` | Azure DI `insured-name = "GNE"` (обрезанное имя) — LLM скопировал |
| `insured.city = "SAME"` | Azure DI `insured-city = "SAME"` (текст-метка на форме) — LLM скопировал буквально |
| `insurance_plan_name = "AMERICAN PIAN ADMINISTRATORS"` | Опечатка в OCR (PLAN→PIAN) — LLM скопировал |
| `provider.name = "CTR STROPHER RUMPLE"` | OCR разбил "DR CHRISTOPHER" → "CTR STROPHER" — LLM не исправил |

### Что должен был сделать LLM

Это именно та задача, с которой LLM справляется отлично — **лингвистическая коррекция**:

| OCR вернул | LLM должен вернуть | Почему LLM знает это |
|---|---|---|
| `"CTR STROPHER RUMPLE"` | `"DR. CHRISTOPHER RUMPLE"` | Знает сокращения и общепринятые имена |
| `"MANCHESTER MEMCRIA HOSFT"` | `"MANCHESTER MEMORIAL HOSPITAL"` | Знает типичные слова для медучреждений |
| `"71 HAYKES TEIN"` | `"71 HAYES STREET"` | Знает типичные названия улиц |
| `"SUFF ELL"` | `"SHEFFIELD"` | Знает географические названия |
| `"AndreyKerchin"` | `"Andrey Kerchin"` | Знает что имена разделяются пробелом |
| `"09 25 1963M."` | `"1963-09-25"` | Знает форматы дат |
| `insured-city = "SAME"` | Скопировать значение `patients-city` | Понимает семантику формы |

Но сделать это LLM **не может**, потому что промпт говорит "Извлеки поля" — не "Исправь лингвистические ошибки OCR".

---

## Root Cause #3: Баг с {{OCR}} плейсхолдером

### Что должно было быть

В агенте `OcrDemoProcessPageCreatePrompt` (agent_id=18) код строит детальный список OCR с координатами:

```csharp
promptOcr.AppendLine("### OCR Results ((X,Y): Text to add):");
promptOcr.AppendLine($"({item.Location.X},{item.Location.Y}): {item.Content}");
var promptResult = promptTemplate.Replace("{{OCR}}", promptOcr.ToString()); // НИКОГДА НЕ СРАБОТАЕТ
```

Шаблон промпта `OCR Result Processing` содержит только `{{Fields}}` и `{{UseImage}}` — нет `{{OCR}}`. Поэтому `Replace("{{OCR}}", ...)` — это no-op. Весь coordinate-based OCR текст **выбрасывается**. LLM не получает позиционные данные, которые помогли бы понять структуру формы пространственно.

---

## Root Cause #4: Temperature 0.5 для детерминированной задачи

В агентах `OcrDemoProcessPageGpt` и `OcrDemoProcessPageRunPrompt` установлена `temperature: 0.5`. Для structured data extraction нужна `temperature: 0` — максимальная детерминированность. Temperature 0.5 добавляет случайность, что недопустимо для полей типа NPI, ICD-кодов, дат.

---

## Root Cause #5: Слабая модель для vision-задачи

Используется `gpt-4.1-mini` — модель, оптимизированная для скорости и стоимости, а не для точного извлечения данных из медицинских форм с плохим качеством сканирования (мелкий шрифт, рукописные элементы, факсимильные артефакты, штампы).

---

## Итоговая карта причин

```
David видит: "OCR + LLM = хуже, чем чистый OCR"
                            ↑
        ┌───────────────────┴───────────────────┐
        │                                       │
LLM получил роль "Extract ONLY"         LLM копирует OCR-мусор
(идентично Direct Mode prompt)          как достоверные данные
— переизвлекает вместо коррекции                │
        │                                       │
        └──────────────┬────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
         Баг {{OCR}}       Temperature=0.5
         (координаты       (недетерминированность)
          выбрасываются)
```

---

## Почему правильный OCR+LLM должен быть ЛУЧШЕ чем чистый OCR

Azure DI OCR хорошо читает текст, но не понимает смысл. LLM понимает смысл, но может ошибаться при чтении пикселей. Их сильные стороны дополняют друг друга:

| Задача | Чистый OCR | Правильный OCR+LLM |
|--------|-----------|-------------------|
| Точное распознавание символов | Отлично | Принимает OCR без изменений |
| Коррекция char confusion (rn→m, cl→d, 0→O) | Нет | Отлично |
| Восстановление имён ("CTR STROPHER"→"DR. CHRISTOPHER") | Нет | Отлично |
| Восстановление адресов ("HAYKES TEIN"→"HAYES STREET") | Нет | Отлично |
| Нормализация дат ("09 25 1963M."→"1963-09-25") | Нет | Отлично |
| Чекбоксы | Слабо (артефакты :selected:) | Отлично (читает из изображения) |
| "SAME" → скопировать адрес | Нет | Отлично |
| NPI, ICD-10, CPT коды | Точно | Копирует OCR без изменений |

**Условие**: LLM должен получить правильный промпт — роль "лингвистического корректора", а не "экстрактора".
