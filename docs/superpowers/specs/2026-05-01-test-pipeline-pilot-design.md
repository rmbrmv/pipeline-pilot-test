# Test Pipeline Pilot — спека для E2E прогона M15.5

**Цель эксперимента:** прогнать pipeline-orchestrator end-to-end на
**минимальном** проекте, чтобы проверить, что все 4 этапа (план →
разработка → тестирование → релиз) реально отрабатывают на боевом VPS,
GitHub-PR создаётся, watcher на ПК показывает события, и Codex-loop
сходится.

**Это не реальный продукт.** Спека намеренно тривиальная.

## Проект

Создать новый репо `pipeline-pilot-test` (rmbrmv/pipeline-pilot-test)
с одной функцией Python:

```python
def add(a: int, b: int) -> int:
    """Returns the sum of two integers."""
    return a + b
```

И тест:

```python
def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
```

## Структура

```
pipeline-pilot-test/
├── pyproject.toml          # минимальный, [project] + pytest dep
├── src/pilot/__init__.py   # экспорт add
├── src/pilot/math.py       # def add(...)
└── tests/test_math.py
```

## Требования

- Python 3.11.
- `pytest` — единственная dev-зависимость.
- `pytest tests/` должен пройти (≥1 тест зелёный).
- Релиз: `gh pr create --base master`, merge после зелёного CI.

## Brainstorm-обязанности (см. spec §10.5)

```yaml
budget:
  total_input_tokens: 250000   # реалистично для subagent-driven workflow на минимальном проекте (M15 E2E pilot 2026-05-03 показал ~67K на одном develop'е)
  expected: 200000
  by_stage:
    plan:
      expected: 25000
      max: 35000
    develop:
      expected: 80000
      max: 120000
    test:
      expected: 40000
      max: 60000
    release:
      expected: 30000
      max: 35000
  reasoning: |
    Base budget после M15 E2E pilot v1: subagent-overhead на любом
    содержательном develop = десятки тысяч токенов даже для одной
    функции (Explore inventory + code-reviewer + Codex review-loop по
    2-3 итерации). Plan/test/release аналогично. v2 базовый бюджет
    ~5x от наивной оценки v1 спеки.
```

## Out of scope

- Knowledge-нота в Obsidian для этого pilot'а **не требуется** (это
  тестовый прогон, не реальная фича).
- TR-эксперименты (TR-2 weekly bucket, TR-3 GitHub конкуренция) на
  этом запуске не воспроизводятся — pilot слишком мал.

## Что считать успехом

- [ ] M15.5 Step 4-7 (см. план) пройдены.
- [ ] PR создан **и merged** в master `pipeline-pilot-test`.
- [ ] `feature/pipeline-test-pipeline-pilot` ветка удалена из remote.
- [ ] Worktree удалён на VPS (`/srv/pipeline/projects/test-pipeline-pilot/`
  не существует).
- [ ] `journal.jsonl` финальный сохранён в merge-коммите (виден в
  diff'е merge'а в master).
- [ ] watcher на ПК выдал хотя бы одно сообщение через `claude --resume -p`
  в журнал-вкладку.

## Что считать провалом

- Pipeline застрял на `blocked` дольше чем 30 минут без явной причины.
- Codex iteration loop > 3 на любом этапе.
- watcher на ПК не получил ни одного события за 30 минут после старта.
- PR создан, но CI красный без разумной причины (тривиальный код
  должен пройти CI).

## Постмортем

После прогона зафиксировать в
[docs/experiments/m15-e2e-pilot-results.md](../../experiments/m15-e2e-pilot-results.md)
(в репо pipeline-orchestrator):

- Дата прогона
- Длительность каждого этапа (фактическая vs expected по budget)
- Tokens used по этапам
- Сколько Codex-итераций на этап
- Найденные баги (если есть)
- Решение по TR-1 (см. `tr1-resume-injection.md`)
