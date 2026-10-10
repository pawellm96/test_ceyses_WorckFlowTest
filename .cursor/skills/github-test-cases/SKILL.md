---
name: github-test-cases
description: >-
  Interprets GitHub issues with the label test case as test cases, computes run
  statistics from checkboxes and sub-tasks, posts tester questions on status
  conflicts, and marks a case as execution error. Use when the user asks for
  test run stats, pass/fail of GitHub test cases, checkbox vs sub-task
  comparison, or when issues labeled test case are involved.
---

# GitHub test cases

Read this skill before computing test-run statistics or posting comments on issues labeled `test case`.

## Правила

1) Если тиккет имеет лэйбл test case то этот тиккет я вляется тест кейсом и может строится по двумправилам:

а) если нету sub-task но есть чекбоксы, то статистика считается по отмеченным чекбоксам. Чекбокс test pass отмечается только при условии что все шаги успешно пройдены

б) при наличии sub-task сравнивать описание в чек боксах и сабтасках. Учитывать если сабтаска и чекбокс обсольютно идентичны и считать за один шаг проверки. В случае если идентичные чек бокс и сабтаск имеют разные статусы (например чекбокс не отмечен, а сабтаска в статусе completed) задавать вопрос тестировщику в комментарии Тест кейса и в статистике отмечать весь тест кейс как Ошибка при выполнении. Статистику по прогону составлять совокупно по чекбоксам и сабтаскам.

## How to apply

Work from GitHub issues (label `test case`) and GitHub Projects. Do not treat unlabeled issues as cases.

### Identify a case

An issue is a test case only if it has the label `test case`.

Load:

- issue body (markdown checkboxes)
- sub-issues (titles and state)
- project Status if the issue is on a board

Ignore the `test pass` checkbox when counting steps. It is a summary, not a step.

`test pass` may be checked only if every verification step passed. If it is checked while any step is unchecked, open, reopened, or in conflict — treat the case as **Ошибка при выполнении** and comment.

### Checkbox text

From a line like `- [ ] make step 1` or `- [x] make step 1`:

- checked = `[x]` / `[X]`
- unchecked = `[ ]`
- description = text after the box, trimmed
- a checkbox is `test pass` when that description equals `test pass` (ASCII, case-insensitive)

### Sub-task status

- passed / completed: sub-issue `state=closed` and `state_reason=completed`
- not passed: `open`, `reopened`, or closed not as completed

### Rule a — no sub-tasks

Steps = all body checkboxes except `test pass`.

- passed steps = checked
- remaining = unchecked
- case pass percent = `passed / total steps * 100` (0 steps → do not invent a percent; report «нет шагов»)
- `test pass` checked only if every step is checked; otherwise it is invalid → **Ошибка при выполнении** + comment
- case result:
  - all steps checked and `test pass` checked → **Пройден**
  - some steps checked, none invalid → **В процессе**
  - no steps checked → **Не запускался**

### Rule b — sub-tasks present

Compare checkbox descriptions (excluding `test pass`) with sub-issue titles.

**Absolutely identical** means exact string match after trim. No fuzzy match, no ignoring «make », no case folding.

- Identical checkbox + sub-task = **one** verification step.
- Combined run stats = union of unique steps from checkboxes and sub-tasks:
  - identical pair → 1 step
  - checkbox without matching sub-task → 1 step (status from checkbox)
  - sub-task without matching checkbox → 1 step (status from sub-task)

For an identical pair:

| Checkbox | Sub-task | Step status |
|---|---|---|
| checked | completed | passed |
| unchecked | not completed | not passed |
| checked vs completed mismatch | any other combo | **Ошибка при выполнении** for the **whole** case |

On mismatch: do not keep a mixed percent as the case result. Mark the entire test case **Ошибка при выполнении**. Still list step-level facts in the report.

Then post a comment on the **parent test case issue** (not on the sub-task). Use GitHub `add_issue_comment`. If a comment with the same pair already exists in the thread, do not duplicate.

Comment template (Russian):

```text
Ошибка при выполнении: статусы шага не совпадают.

Чекбокс: "<описание>" — <отмечен|не отмечен>
Сабтаск: "<title>" (#<n>) — <completed|open|reopened>

Подтвердите актуальный результат шага и выровняйте чекбокс и сабтаск.
```

### Run-level statistics

Aggregate only issues with label `test case`.

For each case report: rule used (a/b), unique steps, passed, not passed, errors, percent, result (`Пройден` / `В процессе` / `Не запускался` / `Ошибка при выполнении`).

Run totals:

- cases counted
- **Пройден** / **В процессе** / **Не запускался** / **Ошибка при выполнении**
- overall step pass percent across unique combined steps (exclude cases already marked **Ошибка при выполнении** from the pass percent, but still show them in the error count)
- list Done-on-board vs actual result when they disagree

Board column `Done` is not Pass. Pass comes only from checkboxes, sub-tasks, and the rules above.

## Examples

**Rule a.** Body:

```markdown
- [x] make step 1
- [x] make step 2
- [ ] test pass
```

No sub-tasks. Steps 2/2 passed, but `test pass` empty → **В процессе**, not **Пройден**. Do not check `test pass` yourself unless the user asks to update the ticket.

**Rule b, identical, conflict.** Checkbox `- [ ] step 2`, sub-issue title `step 2` closed completed. One combined step, statuses differ → comment on the parent case, result **Ошибка при выполнении**.

**Rule b, not identical.** Checkbox `make step 1` and sub-issue `step 1` are two different steps. Count both in combined stats.
