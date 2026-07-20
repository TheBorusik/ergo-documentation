# Модели данных API (справочник структур)

Единый справочник моделей для ссылок из API-описаний этапов.
Вместо раскрытия полей везде — ссылка на модель (напр. `Position`).

Формат: Модель { поле: тип (описание) }
Типы: int, string, decimal, date, datetime, bool, enum, FK→модель

═══════════════════════════════════════════════════
ПРОЕКТ И КП
═══════════════════════════════════════════════════

## Project
```
Project {
  id: int
  code: string?               (номер "П-2026-042", для UI)
  name: string
  customer_id: FK→Contractor
  responsible_id: FK→User?    (менеджер проекта — "мои проекты")
  status: enum (active | won | lost | completed)
  deadline: date?             (срок сдачи — дашборд, просрочки)
  description: text?
  completed_at: datetime?
  created_at: datetime
}
```

## KpGroup
```
KpGroup {
  id: int
  project_id: FK→Project
  name: string ("Оборудование", "Материалы")
  order: int
}
```

## KpItem
```
KpItem {
  id: int
  group_id: FK→KpGroup
  name: string ("Приточная установка П1")
  quantity: decimal
  unit: string ("шт", "м")
  source: enum (kp | added)
  order: int
  price_cost: decimal?     (заглушка, позже)
  margin_percent: decimal? (заглушка)
}
```

═══════════════════════════════════════════════════
ПРОДАЖИ — ВОРОНКА СДЕЛОК (пресейл, см. 05/14)
═══════════════════════════════════════════════════

## ProjectSale (сделка — потенциальный проект до выигрыша)
```
ProjectSale {
  id: int
  project_id: FK→Project?     (заполняется при Won — вопрос в 05/14)
  company_id: FK→Contractor   (организация-заказчик)
  name: string                (наименование)
  location: string            (местоположение объекта)
  service_types: enum[] (supply | engineering | jobs | service)  (мультивыбор)
  expected_signing_date: date
  status: enum (in_progress | stopped | won | lost)
  sales_phase: enum (request_analysis | budget_offer | firm_offer |
                     negotiations | decision | contract)
  user_id: FK→User            (продавец)
  launch_percentage: int      (0|30|60|90|100 — вероятность запуска)
  contract_signing_percentage: int (0|30|60|90|100 — вероятность подписания)
  total_percentage: int       (вычисляемое: launch × signing / 100)
  cost: decimal               (стоимость)
  weight: decimal             (вычисляемое: cost × total% — взвешенный прогноз)
  comment: text?
  created_at, updated_at
}

В ответах связи развёрнуты: Company {id, inn, name}, User {id, name}.
```

## ProjectSalesSummary (агрегат для карточек списка)
```
ProjectSalesSummary {
  total_projects: int
  projects_in_progress: int
  total_cost: decimal
  total_weight: decimal   (взвешенная сумма воронки — прогноз)
}
```

═══════════════════════════════════════════════════
ПОЗИЦИИ (КОНВЕЙЕР)
═══════════════════════════════════════════════════

## Position (project_position — ядро конвейера)
```
Position {
  id: int
  project_id: FK→Project
  position_id: FK→ElementPosition (номенклатура)
  kp_item_id: FK→KpItem?           (привязка к КП, nullable)
  parent_position_id: FK→Position? (если раздроблена)
  spec_subtask_id: FK→Subtask?     (из какой спецификации)
  quantity: decimal
  status: enum (created | spec_ready | in_purchase | purchased |
                delivered | mounting | installed | documented |
                utilized | to_warehouse)
  supplier_id: FK→Contractor?      (финальный поставщик)
  warehouse_id: FK→Warehouse?      (текущий склад)
  current_subtask_id: FK→Subtask?  (текущая занятость, эксклюзивность)
  order_id: FK→Order?              (в какой закупке — ставит бухгалтер)
  created_at: datetime
}
```

## PositionDetail (расширенная — с данными номенклатуры)
```
PositionDetail extends Position {
  nomenclature_name: string   ("Воздуховод 500х300х1250 оц")
  nomenclature_article: string ("ВЗ-П-500-300-1250-ОЦ")
  kp_item_name: string?       ("Приточная П1")
  supplier_name: string?
  warehouse_name: string?
  status_label: string        (человекочит. статус)
}
```

═══════════════════════════════════════════════════
ПОДЗАДАЧИ
═══════════════════════════════════════════════════

## Subtask
```
Subtask {
  id: int
  project_id: FK→Project
  parent_id: FK→Subtask?
  stage_id: FK→ProjectStage   (фиксирован)
  name: string
  status: enum (new | in_progress | done | paused | cancelled)  (ОБЩИЙ)
  stage_status: string        (ВНУТРЕННИЙ, свой на этап)
  responsible_id: FK→User?    (НА КОГО завели — ответственный)
  created_by: FK→User         (КТО завёл — создатель)
  closed_by: FK→User?         (КТО закрыл — nullable, пока не закрыта)
  deadline: date?             (срок задачи — просрочки, дашборд)
  description: text?          (описание задачи)
  created_at: datetime
  closed_at: datetime?        (когда закрыта)
  cancel_reason: string?      (если cancelled)
}

Три роли авторства:
  created_by     — кто завёл задачу
  responsible_id — на кого завели (ответственный, может меняться)
  closed_by      — кто закрыл (done или cancelled)
Полная история смен (в т.ч. переназначения) — в event_log.
```

## SubtaskDetail (с позициями и счётчиками)
```
SubtaskDetail extends Subtask {
  positions: PositionDetail[]
  files: FileAttachment[]
  responsible_name: string?
  positions_count: int
  positions_ready_count: int   (готовых к закупке)
  stage_name: string
}
```

## SubtaskListItem (для списка задач)
```
SubtaskListItem {
  id: int
  name: string
  status: enum
  stage_status: string
  responsible_name: string?
  positions_count: int
  positions_ready_count: int
  supplier_name: string?       (для снабжения)
}
```

═══════════════════════════════════════════════════
НОМЕНКЛАТУРА (для выбора/создания позиций)
═══════════════════════════════════════════════════

## ProjectListItem (для списка проектов — облегчённая)
```
ProjectListItem {
  id: int
  code: string?
  name: string
  customer_name: string       (JOIN)
  status: enum
  current_stage: string       (текущий этап)
  progress: int               (% готовности — вычисляемое)
  deadline: date?
  responsible_name: string?
}
```

## OrderListItem (для раздела Закупки)
```
OrderListItem {
  id: int
  number: string
  supplier_name: string
  total_amount: decimal
  payment_status: enum
  projects_names: string[]    (из позиций — межпроектный)
  paid_at: datetime?
}
```

## ContractorListItem (для справочника)
```
ContractorListItem {
  id: int
  name: string
  role: enum (customer | supplier | contractor)
  inn: string?
  main_activity: string?
  phone: string?
  email: string?
  is_active: bool
}
```

## ElementPosition (позиция номенклатуры — эталон)
```
ElementPosition {
  id: int
  element_id: FK→Element
  code: string      (артикул, генерируется)
  name: string      (имя, генерируется)
  category_name: string?
}
```

## NomenclatureSearchResult
```
NomenclatureSearchResult {
  id: int
  name: string
  article: string
  category: string
  element_name: string
}
```

## Category
```
Category {
  id: int
  name: string
  parent_id: FK→Category?   (плоское дерево)
  order: int
}
```

## Facet (фасет для навигации)
```
Facet {
  property_id: int
  property_name: string      ("Металл", "Ширина")
  type: enum (enum | range)
  values: FacetValue[]?      (для enum)
  range: { min: decimal, max: decimal }?  (для range)
}

FacetValue {
  value: string|int
  label: string              ("оц.сталь")
  count: int                 (сколько позиций с этим значением)
}
```

═══════════════════════════════════════════════════
СПРАВОЧНИКИ
═══════════════════════════════════════════════════

## Contractor (заказчики + поставщики + подрядчики)
```
Объединено с Company из ergo-old (реквизиты ФНС/Checko) — см. 05/13.

Contractor {
  id: int
  role: enum (customer | supplier | contractor)  ← была type (customer|supplier|both)
  name: string                (краткое, "ООО Вент")
  full_name: string           (полное юридическое)
  inn: string
  kpp: string?
  ogrn: string
  org_type: enum (ЮЛ | ИП | НР)?  (из ФНС, readonly)
  register_date: date?        (дата регистрации организации)
  fns_status: string?         (статус из ФНС: "Действующее"...)
  address: string?
  main_activity: string?      (основная деятельность)
  contact_person: string?     (контактное лицо)
  phone: string?
  email: string?
  website: string?
  is_active: bool             (архивные скрывать в UI)
  note: text?
  created_at, updated_at
}

Вопрос: одна роль или несколько (старый "both") — см. 05/13.
```

## CompanySearchResult (результат поиска ФНС/Checko — НЕ хранится)
```
CompanySearchResult {
  type: string        (ЮЛ | ИП | НР)
  inn: string
  ogrn: string
  kpp: string?
  name: string
  full_name: string
  status: string
  address: string
  main_activity: string
  registration_date: string
  okved: string?
  region_code: string?
  liquidation_date: string?
  source: string?     (FNS | CHECKO)
}

CheckoMeta { balance: int, today_requests: int }
  (остаток платного сервиса — показывается в модалке поиска)
```

## User
```
User {
  id: int
  name: string
  role: enum (manager | engineer | supplier_mgr | foreman |
              accountant | employee | admin)
  email: string
  is_active: bool
}
```

## Warehouse
```
Warehouse {
  id: int
  name: string
  type: enum (project | shared)  (проектный / общий)
  project_id: FK→Project?        (для проектных)
}
```

## Work (виды работ)
```
Work {
  id: int
  name: string ("Монтаж воздуховодов")
  unit: string?
  is_active: bool
}
```

## Document (сертификаты/паспорта)
```
Document {
  id: int
  type: enum (certificate | passport)
  name: string
  number: string?
  file_path: string
  issued_date: date?
  expiry_date: date?    (срок годности — ключевой)
  issuer: string?
  is_active: bool
}
```

═══════════════════════════════════════════════════
СНАБЖЕНИЕ
═══════════════════════════════════════════════════

## SupplierRequest (запрос поставщику)
```
SupplierRequest {
  id: int
  subtask_id: FK→Subtask
  supplier_id: FK→Contractor
  document_path: string?   (сгенерированный файл запроса)
  email_to: string?
  sent_at: datetime?
  status: enum (draft | sent | answered | rejected | selected)
  is_selected: bool
}
```

## SupplierQuote (ответ с ценами)
```
SupplierQuote {
  id: int
  request_id: FK→SupplierRequest
  received_at: datetime
  status: enum (new | mapped | confirmed)
}
```

## PriceImportFile (файл ответа — несколько на quote)
```
PriceImportFile {
  id: int
  quote_id: FK→SupplierQuote
  file_path: string
  file_type: enum (excel | pdf | word)
  parse_status: enum (pending | parsed | error)
  template_id: FK→SupplierParseTemplate?
  uploaded_at: datetime
}
```

## PriceImportLine (строка прайса)
```
PriceImportLine {
  id: int
  file_id: FK→PriceImportFile
  raw_name: string        (название у поставщика)
  raw_article: string?
  price: decimal
  quantity: decimal?
  unit: string?
  note: string?
  matched_position_id: FK→Position?  (после маппинга)
  match_confidence: enum (exact | pattern | fuzzy | manual | none)
}
```

## Approval (согласование)
```
Approval {
  id: int
  subtask_id: FK→Subtask
  manager_id: FK→User?
  status: enum (pending | approved | rejected)
  comment: string?
  approved_at: datetime?
}
```

## Payment (оплата)
```
Payment {
  id: int
  subtask_id: FK→Subtask
  accountant_id: FK→User?
  amount: decimal?
  status: enum (pending | paid)
  paid_at: datetime?
}
```

## Delivery (доставка)
```
Delivery {
  id: int
  subtask_id: FK→Subtask
  ready_date: date?
  delivered_at: datetime?
  warehouse_id: FK→Warehouse?
  status: enum (pending | ready | shipping | delivered)
}
```

═══════════════════════════════════════════════════
ЦЕНЫ И МАППИНГ
═══════════════════════════════════════════════════

## SupplierPositionLink (соответствие наша↔поставщика)
```
SupplierPositionLink {
  id: int
  our_position_id: FK→ElementPosition
  supplier_id: FK→Contractor
  supplier_name: string    (название у поставщика)
  supplier_article: string?
  created_at: datetime
}
```

## SupplierPrice (история цен)
```
SupplierPrice {
  id: int
  link_id: FK→SupplierPositionLink
  price: decimal
  currency: string
  quoted_at: datetime
  is_current: bool
}
```

═══════════════════════════════════════════════════
РАБОТЫ И ДОКУМЕНТЫ
═══════════════════════════════════════════════════

## SubtaskWork (работа в подзадаче)
```
SubtaskWork {
  id: int
  subtask_id: FK→Subtask
  work_id: FK→Work
  volume: decimal?     (под вопросом)
}
```

## WorkAct (акт выполненных работ)
```
WorkAct {
  id: int
  project_id: FK→Project
  number: string
  date: date
  status: enum (draft | signed)
  created_by: FK→User
}
```

## PositionDocument (документ на позицию)
```
PositionDocument {
  id: int
  position_id: FK→Position
  document_id: FK→Document
}
```

═══════════════════════════════════════════════════
ОБЩИЕ
═══════════════════════════════════════════════════

## FileAttachment
```
FileAttachment {
  id: int
  entity_type: string   (subtask | project | position)
  entity_id: int
  file_path: string
  file_name: string
  uploaded_by: FK→User
  uploaded_at: datetime
}
```

## ProjectStage (этап проекта со статусом)
```
ProjectStage {
  id: int
  project_id: FK→Project
  stage_type: enum (sales|design|supply|works|docs|handover)
  status: enum (waiting | in_progress | completed)
  started_at: datetime?
  completed_at: datetime?
  completed_by: FK→User?
}
```

## EventLog (унифицированная история — заменяет PositionHistory)
```
EventLog {
  id: int
  project_id: FK→Project
  event_type: string    (stage_started, stage_completed, stage_reopened,
                        position_created, position_purchased, ...)
  entity_type: enum (stage | position | subtask | project)
  entity_id: int
  from_status: string?
  to_status: string?
  reason: string?       (причина доработки/переоткрытия)
  detail: string?
  changed_by: FK→User
  changed_at: datetime
}

Срезы:
  история позиции = WHERE entity_type=position AND entity_id={id}
  история этапа = WHERE entity_type=stage AND entity_id={id}
  timeline проекта = WHERE project_id={id} ORDER BY changed_at
```

## Order (закупка — финансовая таблица, создаёт бухгалтер)
```
Order {
  id: int
  number: string              (автономер, "ORD-101")
  supplier_id: FK→Contractor  (поставщик)
  total_amount: decimal       (сумма закупки)
  payment_status: enum (draft | approved | in_payment | paid)
  invoice_file: string?       (счёт)
  approved_by: FK→User?, approved_at: datetime?
  paid_by: FK→User?, paid_at: datetime?
  created_by: FK→User         (бухгалтер, создавший)
  created_at: datetime
}

Позиции закупки — SELECT positions WHERE order_id={id}
  (обратная связь, order может быть межпроектным)
Задача закупки (subtask stage=закупки) — рабочий процесс,
  order создаётся бухгалтером при оплате (отдельно от задачи).
```

## DashboardWidget (каталог виджетов — админ настраивает)
```
DashboardWidget {
  id: int
  widget_key: string        ("metric.projects_active")
  name, description: string
  category: enum (metric|list|chart|feed|attention|calendar|personal)
  min_w, min_h: int         (минимальный размер)
  max_w, max_h: int         (максимальный)
  default_w, default_h: int
  preview_image: string?
  required_permission: string?
  config_schema: JSONB?
  is_enabled: bool
}
```

## UserDashboard (раскладка пользователя)
```
UserDashboard {
  id: int
  user_id: FK→User
  layout: JSONB    (grid_columns + rows[] с виджетами)
  updated_at: datetime
}
```

## DashboardTemplate (шаблон дашборда)
```
DashboardTemplate {
  id: int
  name: string
  layout: JSONB
  created_by: FK→User
  is_public: bool
  role_hint: string?
}
```

## ApiError
```
ApiError {
  error: string
  message: string
  details: object?
}
```
