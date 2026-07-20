# API: Система управления номенклатурой
## Зона настройки (Admin API)

Все эндпоинты возвращают JSON. Базовый URL: `/api/v1`

---

## 1. reference_types

### GET /reference-types
Список всех типов измерений. `kind` — на типе (решение 2026-07-21).

**Response:**
```json
[
  { "id": 1, "name": "Длина", "kind": "unit" },
  { "id": 2, "name": "Масса", "kind": "unit" },
  { "id": 16, "name": "Материал", "kind": "enum" }
]
```

### POST /reference-types
Создать тип измерения. Вид (`kind`) выбирается один раз здесь — все значения типа его наследуют.

**Request:**
```json
{ "name": "Длина", "kind": "unit" }
```

**Валидация:**
- `kind` должен быть `unit` или `enum`; после создания не меняется (значения уже могли использоваться)

### PUT /reference-types/:id
Обновить тип измерения.

### DELETE /reference-types/:id
Удалить (только если нет связанных references).

---

## 2. references

### GET /references
Список всех значений. Поддерживает фильтрацию.

**Query params:**
```
?reference_type_id=1     — только значения типа "Длина"
?kind=unit                 — только значения unit-типов
?kind=enum                 — только значения enum-типов
?search=мм                 — поиск по name/symbol
```

**Response** (`kind` — производное от типа, у значения собственного поля нет):
```json
[
  {
    "id": 1,
    "reference_type_id": 1,
    "reference_type_name": "Длина",
    "name": "Миллиметр",
    "symbol": "мм",
    "kind": "unit"
  },
  {
    "id": 32,
    "reference_type_id": 16,
    "reference_type_name": "Материал",
    "name": "Оцинкованная сталь",
    "symbol": "оц.сталь",
    "kind": "enum"
  }
]
```

### GET /references/:id
Получить конкретное значение.

### POST /references
Создать значение. Вид НЕ передаётся — наследуется от `reference_types.kind` (решение 2026-07-21). `article_code` имеет смысл только для значений enum-типов.

**Request:**
```json
{
  "reference_type_id": 1,
  "name": "Миллиметр",
  "symbol": "мм"
}
```

**Валидация:**
- `reference_type_id` должен существовать

### PUT /references/:id
Обновить значение.

### DELETE /references/:id
Удалить (только если не используется в element_properties или element_position_properties).

---

## 3. qualifiers

### GET /qualifiers
Список всех уточнений. Включает список свойств к которым привязано.

**Response:**
```json
[
  {
    "id": 1,
    "name": "Наружный",
    "properties": [
      { "id": 4, "name": "Диаметр" }
    ]
  },
  {
    "id": 15,
    "name": "Статическое",
    "properties": [
      { "id": 11, "name": "Давление" }
    ]
  }
]
```

### POST /qualifiers
Создать уточнение.

**Request:**
```json
{ "name": "Наружный" }
```

### PUT /qualifiers/:id
Обновить уточнение.

### DELETE /qualifiers/:id
Удалить (только если нет связей в property_qualifiers).

---

## 4. properties

### GET /properties
Список всех свойств. Включает тип измерения и допустимые qualifiers.

**Query params:**
```
?reference_type_id=1   — свойства типа "Длина"
?search=диаметр          — поиск по name
```

**Response:**
```json
[
  {
    "id": 4,
    "name": "Диаметр",
    "reference_type_id": 1,
    "reference_type": {
      "id": 1,
      "name": "Длина"
    },
    "qualifiers": [
      { "id": 1, "name": "Наружный" },
      { "id": 2, "name": "Внутренний" },
      { "id": 3, "name": "Условный (Ду)" }
    ]
  }
]
```

### GET /properties/:id
Получить свойство с полной информацией.

### POST /properties
Создать свойство.

**Request:**
```json
{
  "name": "Диаметр",
  "reference_type_id": 1
}
```

**Валидация (правило 2026-07-21):**
- если `reference_types.kind = enum` — на этот тип уже НЕ должно быть другого свойства (на перечисление максимум одно свойство; unit-типов это не касается — Длина → Ширина/Высота/Диаметр…). Ошибка: «Для перечисления уже есть свойство "{name}"»

### PUT /properties/:id
Обновить свойство.

### DELETE /properties/:id
Удалить (только если не используется в element_properties).

---

## 5. property_qualifiers

### GET /properties/:id/qualifiers
Список допустимых qualifiers для свойства.

**Response:**
```json
[
  { "id": 1, "name": "Наружный" },
  { "id": 2, "name": "Внутренний" },
  { "id": 3, "name": "Условный (Ду)" }
]
```

### POST /properties/:id/qualifiers
Привязать qualifier к свойству.

**Request:**
```json
{ "qualifier_id": 1 }
```

### DELETE /properties/:id/qualifiers/:qualifier_id
Отвязать qualifier от свойства.

---

## 6. categories

### GET /categories
Дерево категорий.

**Response:**
```json
[
  {
    "id": 1,
    "name": "Вентиляция",
    "parent_id": null,
    "children": [
      {
        "id": 2,
        "name": "Воздуховоды",
        "parent_id": 1,
        "children": [
          { "id": 3, "name": "Прямоугольные", "parent_id": 2, "children": [] },
          { "id": 4, "name": "Круглые", "parent_id": 2, "children": [] }
        ]
      }
    ]
  }
]
```

### GET /categories/flat
Плоский список всех категорий (для дропдаунов).

### POST /categories
Создать категорию.

**Request:**
```json
{
  "name": "Прямоугольные воздуховоды",
  "parent_id": 2
}
```

### PUT /categories/:id
Обновить (можно менять parent_id — перемещение в дереве).

### DELETE /categories/:id
Удалить (только если нет дочерних категорий и нет элементов).

---

## 7. elements

### GET /elements
Список элементов. Включает категорию и количество свойств и позиций.

**Query params:**
```
?category_id=3      — только прямоугольные воздуховоды
?search=воздуховод  — поиск по name
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "Воздуховод прямоугольный",
    "category_id": 3,
    "category": {
      "id": 3,
      "name": "Прямоугольные",
      "path": "Вентиляция / Воздуховоды / Прямоугольные"
    },
    "properties_count": 6,
    "positions_count": 124
  }
]
```

### GET /elements/:id
Получить элемент с полной информацией — свойствами и шаблоном названия.

**Response:**
```json
{
  "id": 1,
  "name": "Воздуховод прямоугольный",
  "category_id": 3,
  "properties": [
    {
      "id": 1,
      "element_property_id": 1,
      "property_id": 1,
      "property_name": "Ширина",
      "reference_type": "Длина",
      "reference": { "id": 1, "name": "Миллиметр", "symbol": "мм", "type": "unit" },
      "qualifier": null,
      "is_required": true,
      "sort_order": 1
    },
    {
      "id": 4,
      "element_property_id": 4,
      "property_id": 22,
      "property_name": "Материал",
      "reference_type": "Материал",
      "reference": null,
      "qualifier": null,
      "is_required": true,
      "sort_order": 4
    }
  ],
  "name_template": [
    { "id": 1, "element_property_id": null, "static_text": "Воздуховод прям.", "separator": " ", "sort_order": 1 },
    { "id": 2, "element_property_id": 1, "static_text": null, "separator": "х", "sort_order": 2 },
    { "id": 3, "element_property_id": 2, "static_text": null, "separator": "х", "sort_order": 3 },
    { "id": 4, "element_property_id": 3, "static_text": null, "separator": " ", "sort_order": 4 },
    { "id": 5, "element_property_id": 4, "static_text": null, "separator": "", "sort_order": 5 }
  ]
}
```

### POST /elements
Создать элемент (шаг 1 из 3 мастера).

**Request:**
```json
{
  "name": "Воздуховод прямоугольный",
  "category_id": 3
}
```

### PUT /elements/:id
Обновить основные данные элемента.

### DELETE /elements/:id
Удалить (только если нет позиций).

---

## 8. element_properties

### GET /elements/:id/properties
Свойства элемента — уже включены в GET /elements/:id

### POST /elements/:id/properties
Добавить свойство к элементу (шаг 2 мастера).

**Request:**
```json
{
  "property_id": 4,
  "reference_id": 1,
  "qualifier_id": 1,
  "is_required": true,
  "sort_order": 1
}
```

**Валидация:**
- `reference_id` должен принадлежать тому же `reference_type` что и `property`
- `qualifier_id` должен быть в `property_qualifiers` для данного `property_id`
- Комбинация `property_id + qualifier_id` должна быть уникальна в рамках элемента

### PUT /elements/:id/properties/:element_property_id
Обновить свойство элемента (единицу, qualifier, обязательность, порядок).

### PUT /elements/:id/properties/reorder
Изменить порядок свойств (drag & drop).

**Request:**
```json
{
  "order": [3, 1, 4, 2, 5]
}
```

### DELETE /elements/:id/properties/:element_property_id
Удалить свойство из элемента (только если нет позиций или у позиций нет этого значения).

---

## 9. element_templates

### GET /elements/:id/name-template
Шаблон названия — уже включён в GET /elements/:id

### POST /elements/:id/name-template
Добавить часть шаблона (шаг 3 мастера).

**Request:**
```json
{
  "element_property_id": 1,
  "static_text": null,
  "separator": "х",
  "sort_order": 2
}
```

**Или фиксированный текст:**
```json
{
  "element_property_id": null,
  "static_text": "Воздуховод прям.",
  "separator": " ",
  "sort_order": 1
}
```

### PUT /elements/:id/name-template/reorder
Изменить порядок частей шаблона.

### DELETE /elements/:id/name-template/:template_part_id
Удалить часть шаблона.

### POST /elements/:id/name-template/preview
Предпросмотр названия с тестовыми значениями.

**Request:**
```json
{
  "values": {
    "1": "500",
    "2": "300",
    "3": "1000",
    "4": "оц.сталь"
  }
}
```

**Response:**
```json
{
  "preview": "Воздуховод прям. 500х300х1000 оц.сталь"
}
```

---

## 10. Вспомогательные эндпоинты для UI

### GET /references/by-type/:reference_type_id
Все значения для конкретного типа — для дропдаунов при настройке свойств элемента.

**Response:**
```json
{
  "reference_type": { "id": 1, "name": "Длина" },
  "type": "unit",
  "values": [
    { "id": 1, "name": "Миллиметр", "symbol": "мм" },
    { "id": 2, "name": "Сантиметр", "symbol": "см" },
    { "id": 3, "name": "Метр", "symbol": "м" }
  ]
}
```

### GET /properties/:id/available-qualifiers
Допустимые qualifiers для свойства — для дропдауна при настройке элемента.

**Response:**
```json
[
  { "id": 1, "name": "Наружный" },
  { "id": 2, "name": "Внутренний" },
  { "id": 3, "name": "Условный (Ду)" }
]
```

### GET /elements/:id/available-properties
Список свойств доступных для добавления в элемент (все properties).
Используется для дропдауна "+ Добавить свойство".

**Response:**
```json
[
  {
    "id": 1,
    "name": "Ширина",
    "reference_type": { "id": 1, "name": "Длина" },
    "has_qualifiers": false,
    "already_added": true
  },
  {
    "id": 4,
    "name": "Диаметр",
    "reference_type": { "id": 1, "name": "Длина" },
    "has_qualifiers": true,
    "already_added": false
  }
]
```

---

## Коды ошибок

| Код | Описание |
|---|---|
| 400 | Ошибка валидации — тело ответа содержит details |
| 404 | Запись не найдена |
| 409 | Конфликт — запись используется и не может быть удалена |
| 422 | Нарушение бизнес-логики (например qualifier не привязан к свойству) |

**Пример ошибки валидации:**
```json
{
  "error": "validation_error",
  "message": "Ошибка валидации",
  "details": {
    "qualifier_id": "Уточнение 'Нетто' не допустимо для свойства 'Диаметр'",
    "reference_id": "Единица 'кг' не соответствует типу измерения 'Длина'"
  }
}
```

---

## Модуль мэппинга поставщиков

---

## 11. suppliers

### GET /suppliers
Список всех поставщиков.

**Response:**
```json
[
  { "id": 1, "name": "ООО Альфа Вентиляция", "code": "ALPHA", "mappings_count": 234 },
  { "id": 2, "name": "ИП Петров В.С.", "code": "PETROV", "mappings_count": 87 }
]
```

### POST /suppliers
Создать поставщика.

**Request:**
```json
{ "name": "ООО Альфа Вентиляция", "code": "ALPHA" }
```

### PUT /suppliers/:id
Обновить поставщика.

### DELETE /suppliers/:id
Удалить (только если нет связанных mappings).

---

## 12. Импорт прайса

### POST /suppliers/:id/import
Загрузить файл прайса. Запускает парсинг и мэппинг всех строк.

**Request:** `multipart/form-data`
```
file: прайс_альфа.xlsx
sheet: 0          (номер листа, по умолчанию 0)
column: 2         (номер колонки с названиями, по умолчанию 0)
skip_rows: 1      (пропустить строк сверху, по умолчанию 1)
```

**Response:**
```json
{
  "import_id": "imp_abc123",
  "total": 150,
  "status": "processing"
}
```

Мэппинг запускается асинхронно — результаты получаем через polling.

---

### GET /suppliers/:id/imports/:import_id
Получить результаты импорта.

**Response:**
```json
{
  "import_id": "imp_abc123",
  "supplier_id": 1,
  "status": "done",
  "total": 150,
  "stats": {
    "exact": 87,
    "pattern": 34,
    "fuzzy": 18,
    "not_found": 11
  },
  "rows": [
    {
      "row_id": 1,
      "raw_text": "Воздуховод 500х300х1250 оц/ст т.0,7",
      "match_level": "exact",
      "confidence": 100,
      "position": {
        "id": 123,
        "name": "Воздуховод прям. 500х300х1250 оц.сталь"
      },
      "status": "auto_accepted"
    },
    {
      "row_id": 2,
      "raw_text": "Воздух. 400х200 L=1000 ст.оц.",
      "match_level": "pattern",
      "confidence": 91,
      "position": {
        "id": 456,
        "name": "Воздуховод прям. 400х200х1000 оц.сталь"
      },
      "extracted": {
        "ширина": 400,
        "высота": 200,
        "длина": 1000,
        "материал": "оц.сталь"
      },
      "status": "pending"
    },
    {
      "row_id": 3,
      "raw_text": "Фильтр ФРП 500х300",
      "match_level": "fuzzy",
      "confidence": 63,
      "candidates": [
        { "id": 301, "name": "Фильтр карманный 500х300", "score": 63 },
        { "id": 302, "name": "Фильтр панельный 500х300", "score": 58 }
      ],
      "status": "pending"
    },
    {
      "row_id": 4,
      "raw_text": "Диффузор потолочный ДП-4 400мм",
      "match_level": "not_found",
      "confidence": 0,
      "status": "pending"
    }
  ]
}
```

**match_level значения:**
```
exact      — точное совпадение из supplier_mappings (100%)
pattern    — сработал паттерн из supplier_patterns (85-95%)
fuzzy      — нечёткий поиск по токенам (50-80%)
not_found  — ничего не найдено
```

**status значения:**
```
auto_accepted — принято автоматически (exact 100%)
pending       — ожидает подтверждения пользователя
confirmed     — подтверждено пользователем
rejected      — отклонено пользователем
created       — создана новая позиция
```

---

### POST /suppliers/:id/imports/:import_id/confirm
Подтвердить или отклонить соответствия. Принимает массив решений.

**Request:**
```json
{
  "decisions": [
    {
      "row_id": 2,
      "action": "confirm",
      "position_id": 456
    },
    {
      "row_id": 3,
      "action": "confirm",
      "position_id": 301
    },
    {
      "row_id": 4,
      "action": "create",
      "element_id": 9,
      "properties": {
        "property_id_44": { "value": 400, "reference_id": 1 }
      }
    }
  ]
}
```

**Response:**
```json
{
  "saved_mappings": 3,
  "new_positions": 1,
  "new_synonyms": [
    {
      "raw_text": "ст.оц.",
      "reference_id": 32,
      "suggested": true
    }
  ],
  "new_patterns_pending": [
    {
      "id": 5,
      "pattern_template": "Воздух. {d}х{d} L={d} ст.оц.",
      "confirmations": 3,
      "status": "pending"
    }
  ]
}
```

После подтверждения система автоматически:
- Сохраняет в `supplier_mappings`
- Предлагает новые синонимы через `suggested: true`
- Генерирует паттерны если накопилось 3+ похожих строк

---

## 13. supplier_mappings

### GET /suppliers/:id/mappings
Список подтверждённых соответствий поставщика.

**Query params:**
```
?search=воздуховод    — поиск по raw_text
?position_id=123      — все строки ведущие к этой позиции
?confirmed_by=manual  — только ручные подтверждения
```

**Response:**
```json
[
  {
    "id": 1,
    "raw_text": "Воздуховод 500х300х1250 оц/ст т.0,7",
    "position": { "id": 123, "name": "Воздуховод прям. 500х300х1250 оц.сталь" },
    "confidence": 100,
    "confirmed_by": "manual",
    "created_at": "2024-03-15"
  }
]
```

### DELETE /suppliers/:id/mappings/:mapping_id
Удалить соответствие (если оно ошибочное).

---

## 14. supplier_synonyms

### GET /synonyms
Все синонимы. Включает глобальные (supplier_id = null).

**Query params:**
```
?supplier_id=1        — только для поставщика
?supplier_id=global   — только глобальные
?reference_id=32    — синонимы для конкретного значения
```

**Response:**
```json
[
  {
    "id": 1,
    "supplier_id": 1,
    "supplier_name": "ООО Альфа",
    "raw_text": "оц/ст",
    "reference": { "id": 32, "name": "Оцинкованная сталь", "symbol": "оц.сталь" },
    "hits": 47
  },
  {
    "id": 3,
    "supplier_id": null,
    "supplier_name": null,
    "raw_text": "оцинковка",
    "reference": { "id": 32, "name": "Оцинкованная сталь", "symbol": "оц.сталь" },
    "hits": 12
  }
]
```

### POST /synonyms
Создать синоним вручную или подтвердить предложенный системой.

**Request:**
```json
{
  "supplier_id": 1,
  "raw_text": "ст.оц.",
  "reference_id": 32
}
```

### DELETE /synonyms/:id
Удалить синоним.

---

## 15. supplier_patterns

### GET /suppliers/:id/patterns
Список паттернов поставщика.

**Query params:**
```
?status=pending   — только ожидающие активации
?status=active    — только активные
?element_id=1     — паттерны для конкретного элемента
```

**Response:**
```json
[
  {
    "id": 1,
    "element": { "id": 1, "name": "Воздуховод прямоугольный" },
    "pattern_template": "Воздуховод {d}х{d}х{d} оц/ст т.{f}",
    "property_map": {
      "1": { "property_id": 1, "property_name": "Ширина" },
      "2": { "property_id": 2, "property_name": "Высота" },
      "3": { "property_id": 3, "property_name": "Длина" },
      "4": { "property_id": 7, "property_name": "Толщина", "qualifier": "Стенки" },
      "constants": { "material": 32 }
    },
    "hits": 47,
    "confirmations": 12,
    "status": "active"
  },
  {
    "id": 4,
    "element": { "id": 8, "name": "Вентилятор канальный" },
    "pattern_template": "Вент.канальный {s} {s} {d}",
    "property_map": {
      "1": { "property_id": 44, "property_name": "Производитель" },
      "2": { "property_id": 46, "property_name": "Модель оборудования" },
      "3": { "property_id": 4,  "property_name": "Диаметр" }
    },
    "hits": 2,
    "confirmations": 2,
    "status": "pending"
  }
]
```

### PUT /suppliers/:id/patterns/:pattern_id
Активировать, отключить или отредактировать паттерн.

**Request:**
```json
{ "status": "active" }
```

```json
{
  "status": "active",
  "pattern_template": "Воздуховод {d}х{d}х{d} оц/ст т.{f}",
  "property_map": { "1": { "property_id": 1 } }
}
```

### DELETE /suppliers/:id/patterns/:pattern_id
Удалить паттерн.

---

## 16. Вспомогательные эндпоинты мэппинга

### POST /mapping/test
Протестировать строку без сохранения — показывает как система её распознаёт.

**Request:**
```json
{
  "supplier_id": 1,
  "raw_text": "Воздуховод 600х400х1000 оц/ст т.0,7"
}
```

**Response:**
```json
{
  "raw_text": "Воздуховод 600х400х1000 оц/ст т.0,7",
  "match_level": "pattern",
  "confidence": 94,
  "pattern_used": {
    "id": 1,
    "template": "Воздуховод {d}х{d}х{d} оц/ст т.{f}"
  },
  "extracted": {
    "ширина": 600,
    "высота": 400,
    "длина": 1000,
    "толщина_стенки": 0.7,
    "материал": "оц.сталь"
  },
  "position": {
    "id": 789,
    "name": "Воздуховод прям. 600х400х1000 оц.сталь"
  }
}
```

### GET /mapping/stats
Статистика по мэппингу — для дашборда.

**Response:**
```json
{
  "total_mappings": 1245,
  "total_synonyms": 89,
  "active_patterns": 23,
  "pending_patterns": 4,
  "by_supplier": [
    {
      "supplier_id": 1,
      "name": "ООО Альфа",
      "mappings": 567,
      "auto_rate": 94
    }
  ]
}
```

═══════════════════════════════════════════════════
WIRE-ТИПЫ МЕТОДОВ (Area.Entity.Action — 01/07 A4)
═══════════════════════════════════════════════════
```
REST выше — детальная форма (по одному эндпоинту на сущность).
РЕАЛИЗАЦИЯ (ergo-erp, 05/16) агрегирует справочники — они маленькие
и нужны вместе (форма позиции, конструктор):

Все справочники разом (categories, reference_types, references,
qualifiers, properties, literals):
  → ErgoArea.NomDictionaries.Get                       ✓
Мутации справочников (по сущности):
  → ErgoArea.NomDictionaries.CategoriesAdd/Delete      ✓
  → ErgoArea.NomDictionaries.ReferenceTypesAdd/Delete  ✓
  → ErgoArea.NomDictionaries.ReferencesAdd/Update/Delete ✓
  → ErgoArea.NomDictionaries.QualifiersAdd/Delete      ✓
  → ErgoArea.NomDictionaries.PropertiesAdd/Update/Delete ✓ (+QualifierIds внутри)
  → ErgoArea.NomDictionaries.LiteralsAdd/Delete        ✓

Элементы (шаблоны, включая свойства и группы токенов):
  GET /elements                → ErgoArea.NomElements.Query   ✓
  GET /elements/{id}           → ErgoArea.NomElements.Get     ✓ (+properties+groups)
  POST/PUT elements + вложения → ErgoArea.NomElements.Save    ✓ (Id=0 — создание)
  создание новой версии        → ErgoArea.NomElements.Replace (immutable, TODO)

Позиции номенклатуры:
  GET /positions (+фильтры)    → ErgoArea.NomPositions.Query  ✓ (Filters/Limit/Offset → Total)
  GET /positions/facets        → ErgoArea.NomPositions.Facets ✓ (динамические фасеты категории:
                                   enum → значения со счётчиками, unit → числовой ввод)
  POST /positions              → ErgoArea.NomPositions.Add    ✓ (имя/артикул на сервере)
```
