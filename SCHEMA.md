# Blank Page — content & progress schema (v2)

All curriculum lives in one JSON block inside `index.html`:

```html
<script type="application/json" id="bp-content"> … </script>
```

Edit that block to add or change lessons. The app logic never needs touching. Inside JSON strings, write `</` as `<\/` so the browser doesn't end the script tag early. After editing, open **Settings → Run content self-test**: every reference solution is run against its own tests, so a typo in an expected value shows up immediately.

## Top level

| Key | Type | Meaning |
|---|---|---|
| `schemaVersion` | `1` | Bump only on breaking schema changes |
| `meta` | object | `title`, `contentVersion`, `pyodideVersion`, `syntaxTarget` |
| `skills` | `[{id, name, description}]` | The five skills: `planning`, `decomposition`, `recall`, `debugging`, `structuring` |
| `concepts` | `[{id, name, skill}]` | Units of mastery. Each has a stage 1–5 (6 = mastered) in progress |
| `modules` | `[{id, title, project, objectives, packages, lessons:[lessonId]}]` | Groups lessons; `lessons` is filled at build time from each lesson's `module` |
| `challenges` | see below | Weekly no-AI laptop projects |
| `lessons` | see below | |
| `items` | see below | Every exercise, referenced by id from lessons |
| `diagnostic` | `{title, intro, items:[item]}` | Placement items (same item schema plus `places`) |

## Lesson

```json
{
  "id": "m1l2", "module": "m1", "title": "Dicts as portfolios",
  "skill": "recall", "device": "phone", "estMinutes": 25,
  "objectives": ["..."],
  "steps": [
    {"kind": "item", "ref": "pr-dict-iter"},
    {"kind": "explain", "md": "Short concept note (markdown-lite)"},
    {"kind": "item", "ref": "w-dicts"}
  ]
}
```

Start every lesson with an active item (predict, Parsons, write) so the learner writes or predicts within the first minute.

## Item: common fields

| Field | Required | Notes |
|---|---|---|
| `id` | ✓ | Unique |
| `type` | ✓ | `worked` · `flashcard` · `predict` · `parsons` · `fillBlank` · `spotBug` · `plan` · `code` |
| `concepts` | ✓ | Concept ids. The item is available when every concept's stage ≥ `stage` |
| `stage` | ✓ | 1 worked example · 2 completion · 3 write from spec · 4 tests only · 5 blank page |
| `device` | ✓ | `phone` · `laptop` · `either`. Laptop items are left out of phone sessions |
| `title`, `prompt` | | Markdown-lite: paragraphs, `- ` lists, `1. ` lists, fenced code, `` `code` ``, `**bold**` |
| `hints` | | Exactly 3 strings for gate items at stages 2–4: guiding question, approach in words, partial skeleton. **Empty at stage 5.** |
| `explanation` | | Model answer shown after the explain-back |
| `financeNotes` | | Conventions to flag (annualisation, day counts, etc.) |
| `timeLimitSec` | stage 5 | Countdown. Passing in time with no help = a blank-page win |
| `gate` | | `false` stops an item advancing its concept's stage (default: true for worked, fillBlank, parsons, code, spotBug, plan) |
| `packages` | | e.g. `["pandas"]`: loaded (with dependencies) before the item runs. Bundled: numpy 2.4.6, pandas 3.0.2 |

## Type-specific fields

**flashcard**: `front`, `back`.

**predict**: `code`, `answer` (exact stdout), optional `choices` (must contain `answer`), or `expectError: "ValueError"` when the code raises.

**worked**: `show` (markdown), optional `code`, `runnable`, `explainPrompt`. Self-test checks `verifyOutput` (exact stdout) or `verifyError: {type, line}`.

**fillBlank**: `code` with `__1__`, `__2__`… markers; `answers: [["primary", "alternative", …], …]`. Any filling that passes the tests is accepted.

**parsons**: `lines` (indentation included), `order` (indices of the correct lines; the rest are decoys). With `cases`/`tests`, any order that passes is accepted; without, the order must match exactly.

**spotBug**: `starter` (buggy code), `bugLines` (1-based), `symptom`: `run` | `tests` | `none`, `solution`, test fields; optional `expectError: {type, line}`.

**plan**: `brief`, optional `prefill: {inputs, outputs, steps, signatures}`, `rubric: [{id, text, any:[keywords]}]`, `modelPlan: {inputs, outputs, steps, signatures}`. Signatures must be valid `def name(arg: type) -> type` lines.

**code**: `starter` (`""` for blank), `solution`, test fields. `mode: "tests"` means the learner writes the tests: give `impl` (correct code), `mutants: [{label, code}]` and `minTests`.

## Test fields (code, spotBug, fillBlank, parsons)

```json
"setup": "M = {...}\n",
"cases": [
  {"call": "f(1, 2)", "expect": "3.0", "label": "…", "visible": true,
   "tol": 1e-9, "cmp": "approx | exact", "onFail": "extra hint",
   "diagnoses": [{"expect": "<known wrong value>", "message": "what that wrong answer means"}]},
  {"kind": "raises", "call": "f(-1)", "raises": "ValueError", "match": "regex"},
  {"kind": "stdout", "expect": "exact printed output when run as __main__"},
  {"kind": "stdoutHas", "contains": ["Portfolio"]},
  {"kind": "raises", "call": "main([])", "raises": "SystemExit"},
  {"kind": "silent"}
],
"tests": "def test_x():\n    assert f(1) == approx(1.0)\n",
"checks": [{"check": "defines | typeHints | docstring | mainGuard", "name": "f", "label": "…"}]
```

- `setup` runs **before** the learner's code (and before Run), so it can provide classes, sample data and files under /tmp. Test calls see both.
- `call` and `expect` are Python expressions. `expect` is evaluated with `setup` in scope (not the learner's code).
- pandas Series/DataFrames and numpy arrays compare with a tolerance (`assert_series_equal` / `allclose`).
- Each case runs in a fresh namespace, so mutable state can't leak between cases.
- `approx` and `raises` behave like `pytest.approx` / `pytest.raises`.
- `selftestWrong: [{code, note, expectMessage}]` (optional): known-wrong versions that must fail; `expectMessage` checks a diagnosis fires.

## Challenges

```json
{"id": "ch3-clean", "title": "Messy price file", "summary": "…", "estMinutes": 60, "ladder": 3,
 "brief": "markdown", "checklist": ["…"], "rubric": ["…"],
 "packages": ["pandas"], "setup": "…", "cases": [ … ], "solution": "reference code (self-test only)"}
```

One challenge is offered per week (the first one not yet completed). The learner plans, builds on a laptop, pastes the module back for the hidden tests, ticks the checklist and self-rates the rubric.

## Diagnostic items

Same schema plus `places: [{concept, stage}]`. A pass raises the concept to at least that stage (capped at 4: stage 5 is always earned). Diagnostic items have no hints.

## Progress (localStorage key `blankpage.progress.v1`)

```json
{
  "schemaVersion": 1,
  "settings": {"theme": "auto", "codeSize": 16, "device": "auto"},
  "diagnostic": {"at": 0, "placement": {"dicts": 3}, "score": "6/9"},
  "concepts": {"dicts": {"stage": 4, "weak": false}},
  "items": {"cd-top-n": {"box": 2, "due": "2026-10-20", "attempts": 3, "solves": 1,
            "bestHint": 1, "lastHint": 1, "draft": null, "plan": null,
            "explains": [{"at": 0, "text": "…"}], "timeSec": 240, "blank": {"tries": 0, "wins": 0}}},
  "redo": [{"id": "cd-top-n", "due": "2026-10-18"}],
  "events": [{"t": 0, "id": "cd-top-n", "k": "solve", "lvl": 1, "sk": "recall"}],
  "streak": {"current": 3, "best": 5, "lastDay": "2026-10-17"},
  "xp": 120, "timeLog": {"2026-10-17": 540}, "snippets": [], "lastBackup": null,
  "challenges": {"ch1-attribution": {"plan": {}, "checklist": {"0": true}, "rubric": {"0": "yes"}, "code": "…", "passed": true, "doneAt": 0}},
  "resume": {"title": "Commute", "mode": "commute", "steps": [], "index": 3}
}
```

Hint level: 0 none · 1–3 tiers opened · 4 solution viewed. Leitner boxes 1–5 with intervals 1, 3, 7, 14, 30 days: a clean solve moves up a box, levels 1–2 stay, level 3+ or a miss resets to box 1. Viewing the solution schedules a redo for the next day.

## Commute mode

Commute sessions use only flashcard, predict, parsons and spot-the-bug items. Spot-the-bug ends after the tap (logged as `tap-pass`, no stage change; the fix stays open for a keyboard session), explain-back is optional, and Parsons falls back to an order check if Python hasn't loaded.

## Hosting

`pyodide/` next to the page holds Pyodide 314.0.7, numpy and pandas. Hosts that refuse .zip/.whl get them as `*.wasm`, listed in `pyodide/bundle.json`; the app unpacks the wheels itself. Without that folder the app loads Pyodide from cdn.jsdelivr.net.
