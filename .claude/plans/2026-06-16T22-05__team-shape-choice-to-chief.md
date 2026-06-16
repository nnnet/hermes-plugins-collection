# Перенос выбора team-shape с Гермеса на Тимлида (chief-manager)

**Дата:** 2026-06-16
**Плагин:** `sources/hermes-plugins-collection/assistant-prompt-overlay`

## Context (зачем)

Пользователь: «команду должен выбирать ТОЛЬКО Тимлид после того, как ему
делегировали задачу». Сейчас выбор формы команды (team-shape) сидит у
делегирующего слоя — и это баг.

**Проверено по живому коду:**
- Гейт инъекции в `__init__.py:108-119`:
  `can_delegate = chief_spawn|mc_project_create in tools` И
  `is_operator_assistant = "terminal" not in tools`.
- Telegram-Гермес: `platform_toolsets.telegram = [hermes-telegram, kanban,
  cronjob, moa]` — **`terminal` отсутствует**, `chief_spawn` есть (из kanban).
  → гейт срабатывает → Гермес **получает** блок делегирования + каталог
  шаблонов + инструкцию «классифицируй тип команды, назови шаблон в брифе»
  (`constants/assistant_delegation.py:701-760`).
- chief-manager (Тимлид): `disabled_toolsets:[terminal,...]`, `chief_spawn`
  есть → **тот же** блок идёт и ему.
- Итог: один текст у обоих, выбор команды дублируется на Гермеса И Тимлида.

**Целевое поведение (подтверждено пользователем — chief-manager = Тимлид):**
- **Гермес/оператор** — передаёт Тимлиду ТОЛЬКО дословную цель. Не
  классифицирует тип команды, не называет шаблон.
- **Тимлид (chief-manager)** — получив цель, САМ классифицирует тип команды,
  зовёт `workflow_list_templates()`, выбирает шаблон по description и
  запускает `workflow_run()`. Каталог шаблонов — только у него.

## Дискриминатор оператор vs Тимлид

Оба без `terminal`, оба с `chief_spawn` — текущий гейт их не различает.
Надёжный признак Тимлида — **процесс chief-воркера**: dispatcher выставляет
env `HERMES_KANBAN_BOARD` (+ `HERMES_PROFILE=chief-manager`), и на этой доске
лежат chief-метаданные (так же детектит `chief-tools/chief_tools.py` через
`HERMES_KANBAN_BOARD` + `_read_chief_meta`). Оператор — gateway-процесс, у
него `HERMES_KANBAN_BOARD` не задан.

```
is_chief = bool(os.environ.get("HERMES_KANBAN_BOARD"))   # chief-воркер
is_operator = can_delegate and not terminal and not is_chief
```

(Под-чифы тоже chief — им каталог нужен так же, признак тот же.)

## Изменения

### 1. `constants/assistant_delegation.py` — убрать выбор команды у оператора
- Вырезать секцию «Team-shape workflow templates — classify team TYPE…»
  (строки 701-760) из `ASSISTANT_DELEGATION_GUIDANCE`.
- «Brief contract to chief_spawn» переписать: бриф несёт **только** дословную
  цель пользователя как `task_brief`. Явно: «НЕ классифицируй тип команды, НЕ
  называй шаблон — форму команды выбирает Тимлид». Skeleton брифа упростить
  до голой цитаты цели.

### 2. `constants/team_shape.py` (НОВЫЙ) — блок для Тимлида
- Перенести вырезанную секцию, **переписав адресата**: «Ты — Тимлид. Тебе
  делегировали цель. Классифицируй её в тип команды (dev/research/creative/
  ops), вызови `workflow_list_templates()`, выбери шаблон по description,
  запусти `workflow_run(template=…, inputs={task_brief: <дословная цель>})`».
- Константа `TEAM_SHAPE_SELECTION_GUIDANCE`.

### 3. `overlays/workflow_templates.py` — переписать шапку каталога
- Текст блока: вместо «name the matching template in the chief brief» →
  «classify the goal and `workflow_run` the matching template yourself»
  (адресат — Тимлид, не делегатор).

### 4. `__init__.py` — разнести инъекции по ролям
- Вычислить `is_chief` / `is_operator` (см. дискриминатор), `import os`.
- `is_operator` → `_add(ASSISTANT_DELEGATION_GUIDANCE)` (уже без team-shape).
- `is_chief` → `_add(ASSISTANT_DELEGATION_GUIDANCE)` (под-чифы делегируют) +
  `_add(TEAM_SHAPE_SELECTION_GUIDANCE)` + `_add(build_workflow_templates_block())`.
- Обновить docstring (строки 16-18) под новую развязку.

## Критические файлы
- `assistant-prompt-overlay/__init__.py` (гейт-развязка, +`import os`)
- `assistant-prompt-overlay/constants/assistant_delegation.py` (вырезать 701-760, упростить brief contract)
- `assistant-prompt-overlay/constants/team_shape.py` (новый)
- `assistant-prompt-overlay/overlays/workflow_templates.py` (шапка каталога)

## Деплой
1. Коммит+пуш сабмодуля `assistant-prompt-overlay` И родителя (sync делает
   `git reset --hard origin/main` — иначе правки сотрутся).
2. `sync-external-plugins.sh` (или ребилд оверлея), `find . -name '*.pyc'
   -delete` в плагине, `docker restart hermes` + рестарт активных chief-воркеров.

## Верификация
1. Дамп системного промпта оператора и chief-воркера; grep «Team-shape» и
   «WORKFLOW TEMPLATES»: у **оператора отсутствуют**, у **chief присутствуют**.
2. Функционально: оператору «построй X» → в брифе `chief_spawn` только
   дословная цель, без типа команды/имени шаблона.
3. chief-воркер по этой цели сам классифицирует и зовёт
   `workflow_list_templates()`/`workflow_run()`.
4. Под-чиф (sub-chief) тоже видит каталог (тот же env-признак).

## Вне scope (отдельно)
Сам «голос» `ASSISTANT_DELEGATION_GUIDANCE` написан как «ты Гермес, делегируй
Тимлиду» и сейчас по гейту попадает и в chief-manager. Полная переозвучка
роли для Тимлида — бóльшая правка; здесь чиним только ответственность за
выбор команды, как просил пользователь.

## После ExitPlanMode (конвенция проекта)
Перенести этот план в `/mnt/9/aimanager/sources/hermes-plugins-collection/.claude/plans/`
как `2026-06-16T<HH-MM>__team-shape-choice-to-chief.md`, удалить из
`~/.claude/plans/`, `git add`.
