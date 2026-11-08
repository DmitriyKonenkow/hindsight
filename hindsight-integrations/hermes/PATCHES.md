# Локальные патчи Hermes-плагина Hindsight

Этот форк несёт локальные правки поверх upstream
[`vectorize-io/hindsight`](https://github.com/vectorize-io/hindsight)
(`hindsight-integrations/hermes`). База — 1.2.4, SHA `f14cc1eb5624b0374cb4d82768eb2e0b9c21ae24`.

Ветка с патчами: **`hermes-local`**. Первый коммит поверх апстрима — `local: Hermes memory patches`
(в нём всё перечисленное ниже), дальше — обычная история.

## Как обновляться с апстрима

```bash
git fetch upstream                       # remote на vectorize-io/hindsight
git rebase upstream/main                 # или merge, если так удобнее
# конфликты почти всегда в __init__.py — правки помечены комментариями «локальная доработка»
git push origin hermes-local
```

Если ветка `hermes-local` — дефолтная ветка форка, `hermes plugins update hindsight`
подтянет её одной командой; иначе ставьте плагин с `--ref <sha>` из этой ветки и обновляйте
вручную (пин отключает авто-обновление).

## Что именно пропатчено

### 1. Журнал впрысков (audit log)
Константы `_INJECT_LOG_PATH` / `_INJECT_LOG_LOCK` / `_INJECT_LOG_MAX_BYTES`, `_log_injection()`,
`_injection_item()`; вызов из `_finish_prefetch()`; поля `_last_recall_items` /
`_prefetch_items` / `_prefetch_query`.

Каждая авто-инъекция пишется строкой JSONL в `logs/hindsight_inject.jsonl`: запрос, сколько
отдано, какие записи (id, тип, дата, текст), режим sync/async. Без этого пользу памяти нельзя
измерить постфактум — инъекция не сохраняется ни в `state.db`, ни в промпт-логах.

Ручка: `HINDSIGHT_INJECT_LOG`, ротация по 32 МБ в `.1`.

### 2. Гейт авто-recall на бессмысленных запросах
`_SKIP_RECALL_PREFIXES`, `_SKIP_RECALL_MIN_CHARS`, `_recall_worth_asking()`; вызовы в
`prefetch()` и `queue_prefetch()`.

Авто-recall не запускается на сообщениях короче 12 символов и на `[important:`,
`cronjob response:`, `[system`, `[background`. На таких ходах у запроса нет извлекаемого
смысла, но реранкер даёт высокий скор случайным фактам (`06:25` → 0.97 на правиле, где
встретилось `06:23`; `да` → 0.50). Замер по журналу: 9 из 50 ходов и 15 из 112 впрыснутых
фактов — ровно те, где судья ставил `unused`. Явный `hindsight_recall` работает всегда.

Ручка: `HINDSIGHT_RECALL_SKIP_SYSTEM=0` — выключить гейт.

### 3. Дата в строке впрыска + политика в заголовке
`_recall_line()`; текст `header` в `_finish_prefetch()`.

Строка блока — `- [2026-11-07] текст`; заголовок говорит: при конфликте новее = текущее
решение; если строка противоречит текущему контексту, контекст важнее — фактическую ошибку
гасить через `hindsight_retire`, смену решения сначала обсудить. Оба `harmful`-вердикта судьи
были про то, что решение от 3 ноября и отменяющее его от 7-го выглядели одинаково.

### 4. Кап на число строк впрыска
`_do_recall()`: в контекст уходит не больше 8 строк (p90 впрыска — 6). На запросах вроде
«дай ssh ключ» сервер отдавал 19–22 факта, использован был один.

Ручка: `HINDSIGHT_RECALL_MAX_ITEMS` (по умолчанию 8).

### 5. Инструмент `hindsight_retire` (+ журнал гашений)
`RETIRE_SCHEMA`, `_tool_retire()`, `_RETIRE_LOG_PATH`, `_log_retire()`; регистрация в
`get_tool_schemas()` и `_TOOL_HANDLERS`.

Модель гасит ложный факт (`action=retire`) или возвращает его (`restore`) по фрагменту строки
с обязательной причиной. Наблюдения курировать нельзя (`PATCH /memories/{id}` → 400
«observations are derived»), поэтому инструмент идёт по исходникам: `alist_memories(search_query=…,
state=valid)` → `source_memory_ids` → `update_memory(state=invalidated, reason=…)` →
`clear_memory_observations()` (сбрасывает производные наблюдения и запускает пересборку).

Предохранители: фрагмент ≥ 10 символов; нечёткое совпадение → отказ со списком кандидатов;
`reason` обязателен; возвращает `ids` для отката; каждое гашение пишется в
`logs/hindsight_retire.jsonl`, дневной монитор показывает блок `[ГАШЕНИЯ]`.

Ручка: `HINDSIGHT_RETIRE_LOG`.

## Проверено

- `hindsight_retire` E2E на отдельном банке: погашение факта → наблюдение пересобралось из
  оставшихся, recall отдаёт только новое; `restore` по `ids` вернул погашенное.
- Патч применяется к чистому апстриму этого SHA и даёт файл, побайтово равный рабочему.
- Живой процесс подхватывает правки без рестарта:
  `gateway.control_socket.reload_gateway_plugins(Path("/opt/data"))`; хуки оживают сразу,
  инструменты и промпт-секции — со следующей сессии.
