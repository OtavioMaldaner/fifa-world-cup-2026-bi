# Redesign Esportivo — Copa do Mundo 2026 Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the layout, visual hierarchy, and color system of the 3 existing report pages (Visão Geral do Torneio, Perfil da Seleção, Mérito ou Sorte?) into a modern sports-broadcast look (dark green + white cards), fix 2 confirmed data/measure bugs, and eliminate a redundant-goal anti-pattern present on 5 KPI cards — while preserving every existing analysis, filter, and page.

**Architecture:** Direct TMDL authoring for the 2 semantic-model fixes (same pattern as prior sessions: hand-edit `.tmdl`, validate statically with `pbip-validator`). All report changes go through the `pbir` CLI exclusively (v0.9.28, do not upgrade — confirmed compatible this session; upgrading mid-report risks a fresh wave of serializer drift across all 4 pages). Visual redesign proceeds theme-first (one shared palette + typography/format defaults applied report-wide), then page-by-page, each page rebuilt in the same vertical order: header band → filter band → KPI row → content row(s) → bottom table.

**Tech Stack:** Power BI Project (PBIP/TMDL/PBIR), `pbir-cli` 0.9.28, git.

## Global Constraints

- All work happens in the `fifa-world-cup-2026-bi` repository, directly on `main` (matches this project's established convention — no feature branches used to date). Every commit uses `git -C "BI - Semana da Informática"`.
- **No Tabular Editor CLI (`te`), TOM, or Power BI Desktop is available in this environment** (Linux/WSL2, confirmed empirically multiple times, most recently via `pbir model -q` failing with "Local model is not open in Power BI Desktop"). Every DAX-correctness and visual-render check in this plan is a **user gate**, collected in Task 15. Structural checks (TMDL syntax, field bindings resolving, no overlapping visuals, color-literal grep) are **agent gates**, run inline.
- **Do not upgrade `pbir`** (stays 0.9.28 — an update to 0.9.29 is available and was declined in a prior session specifically to avoid re-serializing all 4 pages).
- **Known `pbir` gotchas, confirmed empirically this session — do not re-discover them:**
  - `card` and `tableEx` bind fields to role **`Values`**, regardless of what `pbir schema describe` labels the role group (it says "Columns" for `tableEx` — that's a display label, not the bind key).
  - `pbir visuals bind` uses `--add "<Role>:<Table>.<Field>" --type Column|Measure` — not `--field`.
  - Sorting a visual is `pbir visuals sort <path> --field "<Table>.<Field>" --direction Ascending|Descending` (a real subcommand, easy to miss in `pbir visuals --help`'s command list since it's alphabetically buried).
  - Body-paragraph textboxes (page titles, footnotes) come from `pbir add title "<Page.Page>" "<text>" --width N`, **not** `pbir add visual textbox ... --title "..."` — the latter writes into the visual-header title property, which renders with a visible header bar, inconsistent with a plain textbox.
  - `pbir add page` auto-creates a "Title" textbox at (20,20,480,120) with the page's display name — reuse and reposition it with `pbir visuals position`, don't delete and recreate.
  - Filter operators for `--type Advanced` are exactly: `In, NotIn, GreaterThan, GreaterThanOrEqual, LessThan, LessThanOrEqual, Is, IsNot` (case-sensitive, confirmed via the CLI's own error message).
  - `pbir pages background --color` **does not exist** despite the parent help text implying it — that subcommand only supports `--image`. Canvas solid-fill color is a theme-level concern: set via `pbir theme set-formatting "<Report>" "page.*.background.color" --value "#HEX"` (confirmed: `page` is a valid pseudo-visual-type recognized by `pbir schema describe page background`).
  - `pbir pages wallpaper --color` sets the **outspace** (area outside the 16:9 canvas when the viewport is wider), not the canvas itself — not used in this plan.
- **Grid system:** all 3 pages are already 1280×720 (16:9) — no resize needed. New vertical rhythm, identical across pages 1–2 (page 3 shortens the bottom table to make room for its existing footnote):
  - Header band: `y=0 h=56` (full-bleed, `x=0 w=1280`)
  - Filter band: `y=72 h=40` (16px gap after header)
  - KPI row: `y=128 h=96` (16px gap after filters), 4 cards at `x=24/336/648/960 w=296` — **unchanged x-anchors from the current layout**, kept deliberately so the KPI row stays pixel-aligned with what's already there
  - Content row: `y=240 h=224` (16px gap after KPI row), 2 items at `x=24/648 w=608` (pages 1 & 3) or `x=24 w=728` / `x=768 w=488` (page 2's 60/40 split: `24+728=752, +16=768, +488=1256, +24=1280` ✓)
  - Bottom table: `y=480`, `w=1232 x=24`; `h=216` on pages 1–2 (ends at 696, +24 margin = 720); `h=192` on page 3 (ends at 672, footnote at `y=678 h=18`, ends at 696, +24 margin = 720)
- **Color palette (exact hex, from the approved brief):** dark green `#0B3D2E`, highlight green `#1F7A4D`, light green `#39A96B`, background `#F3F5F4`, cards `#FFFFFF`, primary text `#17221C`, secondary text `#6B776F`, borders `#DCE4DF`, negative `#C74343`, draw/alert `#D4A72C`.
- **Chart data-colors palette (new, all in the green/neutral family — no purple, no unrelated hues):** `["#0B3D2E","#1F7A4D","#39A96B","#6B776F","#2E5947","#7FBF9E","#A8B5AE"]`.
- Measure names are **not** renamed anywhere in this plan (Genie's glossary in `metadata/copa_mundo_2026_metadata.py`, in the sibling Databricks repo, references them by exact name). Where the brief's requested card label differs from the measure's own name (e.g. brief says "Média de Gols por Partida", measure is `Média de Gols/Partida`), only the **visual's** title text changes — the DAX measure name is untouched.
- Color-literal gate: `grep -rE '#[0-9A-Fa-f]{6}' fifa-world-cup-2026.Report/definition/pages/` must return **zero** hits after Task 12 (it currently returns 2 pre-existing hits — `#7B43A3` in `chart_pontos_acima`, `#FFFFFF` in `table_top10` — both are targeted for removal in Tasks 11 and 8 respectively).
- **This environment cannot render, screenshot, or export report pages** (`pbir desktop screenshot` is Windows-only; no live Desktop connection exists here — confirmed via `pbir model -q` failing). The brief's "Verificação final" steps 1, 2, 3, 6 (render, visual review, no-cutoff check, PDF comparison) are **not executable in this session** and are handed to the user as a checklist in Task 15, not silently skipped.

---

## Phase A — Semantic model fixes

### Task 1: Fix the Estabilidade da Escalação double-scaling bug

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`

**Interfaces:**
- Consumes: `vw_estabilidade_escalacao[lineup_stability_pct]` (a column already expressed 0–100, per the Spark transformation `repeated_starters * 100.0 / 11.0` in the sibling Databricks repo's `vw_estabilidade_escalacao.py:121` — confirmed by direct inspection, not assumption).
- Produces: a corrected `Estabilidade da Escalação (%)` measure, consumed by `kpi_estabilidade` (page 3 card, Task 13) and `table_estabilidade` (page 3, Task 14) — both already bind the **measure**, not the raw column, so this single fix propagates to both without touching the report.

**Root cause:** `formatString: 0.0%` on a Power BI measure multiplies the underlying value by 100 for display (it expects a 0–1 fraction). The column is already 0–100, so a value of 72.7 (meaning 72.7%) renders as "7270.0%" — matching the reported ">7.000%" symptom.

- [ ] **Step 1: Confirm current state**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -A2 "measure 'Estabilidade da Escalação (%)'" tables/_Measures.tmdl
cd ../../..
```

Expected: `AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct])` with no `/100` — confirms the bug is still present (not already fixed by a prior session).

- [ ] **Step 2: Apply the fix**

In `fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`, change:

```tmdl
	measure 'Estabilidade da Escalação (%)' = AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct])
		formatString: 0.0%
		displayFolder: _Avançado
		lineageTag: 00000000-0000-0000-0000-000000000306
```

to:

```tmdl
	measure 'Estabilidade da Escalação (%)' = DIVIDE(AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct]), 100)
		formatString: 0.0%
		displayFolder: _Avançado
		lineageTag: 00000000-0000-0000-0000-000000000306
```

(`DIVIDE` over `/` — the `lineup_stability_pct` column is `NULL` for every team's first match per the Spark view's own documentation, and `AVERAGE` already correctly ignores those nulls; `DIVIDE` just guards the outer division, it does not change which rows are averaged.)

- [ ] **Step 3: Verify**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -A1 "measure 'Estabilidade da Escalação (%)'" tables/_Measures.tmdl
cd ../../..
```

Expected: `DIVIDE(AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct]), 100)`.

- [ ] **Step 4: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel`. Zero errors expected (this is a pure expression edit, no structural change).

- [ ] **Step 5: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl
git commit -m "$(cat <<'EOF'
fix: correct double-scaling in Estabilidade da Escalação (%)

vw_estabilidade_escalacao.lineup_stability_pct is already expressed
0-100 (repeated_starters * 100.0 / 11.0, per the Spark transformation
in the Databricks repo) but the measure applied a 0.0% formatString
on top, which multiplies by 100 again for display -- a value of 72.7
(72.7%) rendered as 7270.0%. Fixed by dividing the averaged column by
100 before the percent format is applied. table_estabilidade and
kpi_estabilidade both bind this measure directly, so the fix reaches
both without any report-side change.
EOF
)"
```

---

### Task 2: Translate Fase and Resultado values to Portuguese

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/dim_etapas.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl`

**Interfaces:**
- Consumes: nothing new — reuses the existing `dim_etapas[Fase]` and `vw_selecao_partida[Resultado]` columns' current import bindings (`sourceColumn: stage_name` and `sourceColumn: result` respectively).
- Produces: both become **calculated** columns returning Portuguese values, while every existing report reference (chart_gols_fase's Category, slicer_fase, table_confrontos's Fase and Resultado columns) continues to resolve by the same field name — **no PBIR file needs to change** for either fix.

**Two sub-fixes in this task, same technique applied to two different tables** — Steps 1–5 cover `Fase` (`dim_etapas`), Steps 6–10 cover `Resultado` (`vw_selecao_partida`). The brief explicitly names the expected Portuguese values for Resultado: "utilize Vitória, Empate e Derrota" (page 2 spec) — this is not just a coloring instruction, the raw `WIN`/`DRAW`/`LOSS` strings need translating too.

**Root cause:** `Fase`'s `sourceColumn: stage_name` pulls the raw English values from `tournament_stages.csv` in the Databricks repo ("Group Stage", "Round of 32", "Round of 16", "Quarter-finals", "Semi-finals", "Third-place match", "Final") — confirmed by direct inspection of the CSV. The column was renamed to a Portuguese label, but its *values* were never translated.

**Approach:** DAX calculated columns can only reference other **model** columns, not raw M-partition fields directly — and `stage_name` isn't separately exposed as a model column today (only `Fase` maps to it). So: rename the existing imported column to a hidden `Fase (Origem)`, then add a new calculated `Fase` column (reusing the original name and lineageTag, so every existing report binding keeps working unchanged) that translates it via `SWITCH`.

- [ ] **Step 1: Confirm current state**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
cat tables/dim_etapas.tmdl
cd ../../..
```

Expected: a single `column Fase` with `sourceColumn: stage_name`, `sortByColumn: stage_id`, no `isHidden`, lineageTag `e4e7c4e2-43ee-4e4a-a9a3-73823332f256`.

- [ ] **Step 2: Apply the fix**

Replace this block in `fifa-world-cup-2026.SemanticModel/definition/tables/dim_etapas.tmdl`:

```tmdl
	column Fase
		dataType: string
		lineageTag: e4e7c4e2-43ee-4e4a-a9a3-73823332f256
		summarizeBy: none
		sourceColumn: stage_name
		sortByColumn: stage_id

		annotation SummarizationSetBy = Automatic
```

with:

```tmdl
	column 'Fase (Origem)'
		dataType: string
		isHidden
		lineageTag: 07e1a5a4-4b7d-4b6a-9f0e-3d6a9c2f1b8e
		summarizeBy: none
		sourceColumn: stage_name

		annotation SummarizationSetBy = Automatic

	column Fase =
			SWITCH(
				TRUE(),
				dim_etapas[Fase (Origem)] = "Group Stage", "Fase de Grupos",
				dim_etapas[Fase (Origem)] = "Round of 32", "Rodada de 32",
				dim_etapas[Fase (Origem)] = "Round of 16", "Oitavas de Final",
				dim_etapas[Fase (Origem)] = "Quarter-finals", "Quartas de Final",
				dim_etapas[Fase (Origem)] = "Semi-finals", "Semifinais",
				dim_etapas[Fase (Origem)] = "Third-place match", "Disputa de 3º Lugar",
				dim_etapas[Fase (Origem)] = "Final", "Final",
				dim_etapas[Fase (Origem)]
			)
		lineageTag: e4e7c4e2-43ee-4e4a-a9a3-73823332f256
		summarizeBy: none
		sortByColumn: stage_id

		annotation SummarizationSetBy = Automatic
```

(The `SWITCH(TRUE(), ...)` idiom's final argument, `dim_etapas[Fase (Origem)]`, is the fallback for any stage name not in the 7 known values — it echoes the raw English string rather than showing blank, so nothing silently disappears if the source data ever adds a stage.)

- [ ] **Step 3: Verify**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^\tcolumn " tables/dim_etapas.tmdl   # expect 4 (stage_id, Fase (Origem), Fase, is_knockout)
grep "sortByColumn" tables/dim_etapas.tmdl    # expect exactly 1 hit, under the new calculated Fase
cd ../../..
```

- [ ] **Step 4: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel`. Zero errors expected. Specifically confirm: no duplicate lineageTag, `Fase (Origem)` correctly marked hidden, `Fase`'s DAX expression references only `dim_etapas[Fase (Origem)]` (a real column in the same table, valid for a calculated column). **Explicitly ask for `tmdl-validate` in single-file mode on `dim_etapas.tmdl`, not just `validate_pbip.py`/directory mode** — a prior session-earlier bug (multi-line DAX with a closing paren at the wrong indentation depth) passed both `validate_pbip.py` and `tmdl-validate` directory mode clean and was only caught by single-file mode; the same structural risk applies to this multi-line `SWITCH`.

- [ ] **Step 5: Do not commit yet** — Steps 6–10 add the Resultado fix in the same table-fix pattern; both land in one commit at Step 10 so the "translate values to Portuguese" change is reviewable as a single unit.

- [ ] **Step 6: Confirm current state of Resultado**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -A6 "column Resultado" tables/vw_selecao_partida.tmdl
cd ../../..
```

Expected: `column Resultado`, `dataType: string`, `lineageTag: 72821297-1f23-4d87-b72e-8dc8ccf3d194`, `summarizeBy: none`, `sourceColumn: result`, no `sortByColumn` — confirms the exact block to replace (values are `WIN`/`DRAW`/`LOSS`, per `vw_selecao_partida.py`'s `result` column in the Databricks repo).

- [ ] **Step 7: Apply the fix**

Replace this block in `fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl`:

```tmdl
	column Resultado
		dataType: string
		lineageTag: 72821297-1f23-4d87-b72e-8dc8ccf3d194
		summarizeBy: none
		sourceColumn: result

		annotation SummarizationSetBy = Automatic
```

with:

```tmdl
	column 'Resultado (Origem)'
		dataType: string
		isHidden
		lineageTag: 9c4e2a1f-6b8d-4a3e-b1f0-7e5d8c9a2f6b
		summarizeBy: none
		sourceColumn: result

		annotation SummarizationSetBy = Automatic

	column Resultado =
			SWITCH(
				TRUE(),
				vw_selecao_partida[Resultado (Origem)] = "WIN", "Vitória",
				vw_selecao_partida[Resultado (Origem)] = "DRAW", "Empate",
				vw_selecao_partida[Resultado (Origem)] = "LOSS", "Derrota",
				vw_selecao_partida[Resultado (Origem)]
			)
		lineageTag: 72821297-1f23-4d87-b72e-8dc8ccf3d194
		summarizeBy: none

		annotation SummarizationSetBy = Automatic
```

- [ ] **Step 8: Verify**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^\tcolumn " tables/vw_selecao_partida.tmdl   # one more than before this task
grep -A5 "column Resultado =" tables/vw_selecao_partida.tmdl
cd ../../..
```

- [ ] **Step 9: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` one more time on the combined final state (both `dim_etapas.tmdl` and `vw_selecao_partida.tmdl` changes). Zero errors expected. Same requirement as Step 4: explicitly request `tmdl-validate` single-file mode on `vw_selecao_partida.tmdl`, not directory mode alone.

- [ ] **Step 10: Do not commit yet** — Steps 11–14 fix a second, independently-confirmed pre-existing bug in the same table this task already touches; all three fixes land in one commit at Step 15.

**Bug found while validating Task 1** (via `pbip-validator`'s cross-file column-resolution check, not part of the original brief's named checklist, but squarely inside its "verifique... a coerência entre" mandate): `_Measures.tmdl:39`'s `Vitórias` measure —

```tmdl
	measure Vitórias = CALCULATE([Total de Partidas], vw_selecao_partida[result] = "WIN")
```

— references `vw_selecao_partida[result]`, a raw **source-column** name. DAX table/column references resolve against the **model's** column name, not the underlying SQL source name; the model's actual column is `Resultado` (renamed from `result` — the same renaming pattern this task is already fixing for `Fase` and `Resultado`'s translated values). `vw_selecao_partida[result]` does not exist as a model column, so this measure fails to evaluate (column-not-found) any time it's used. Confirmed via `pbip-validator`'s DAX cross-reference check, not merely a TMDL syntax scan (TMDL validators check grammar, not whether a DAX string references a real column).

- [ ] **Step 11: Confirm current state**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -n "measure Vitórias" tables/_Measures.tmdl
cd ../../..
```

Expected: `measure Vitórias = CALCULATE([Total de Partidas], vw_selecao_partida[result] = "WIN")` — confirms the broken reference is still present.

- [ ] **Step 12: Apply the fix**

In `fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`, change:

```tmdl
	measure Vitórias = CALCULATE([Total de Partidas], vw_selecao_partida[result] = "WIN")
```

to:

```tmdl
	measure Vitórias = CALCULATE([Total de Partidas], vw_selecao_partida[Resultado (Origem)] = "WIN")
```

(References the hidden **raw** column added in Step 7 of this same task, not the translated `Resultado` — this measure's comparison is an internal boolean check invisible to report viewers; only its resulting count is shown, so there's no reason to couple its correctness to the Portuguese string chosen in Step 7. Referencing the untranslated `Resultado (Origem)` with the original `"WIN"` is simpler and won't break if the translation strings ever change independently.)

- [ ] **Step 13: Verify**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -n "measure Vitórias" tables/_Measures.tmdl
cd ../../..
```

Expected: `vw_selecao_partida[Resultado (Origem)] = "WIN"`.

- [ ] **Step 14: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` on the combined final state (`dim_etapas.tmdl`, `vw_selecao_partida.tmdl`, and now `_Measures.tmdl`). Zero errors expected — specifically confirm the `Vitórias` cross-file column-reference error is gone, and that Step 12's edit didn't touch any other measure in the file. **Report the validator's exact raw output in the task report — a narrative summary is not sufficient evidence for this project** (this is a repeated project convention, not new to this task).

- [ ] **Step 15: Commit all three fixes together**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/dim_etapas.tmdl fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl
git commit -m "$(cat <<'EOF'
fix: translate Fase and Resultado values, fix Vitórias' broken reference

Both dim_etapas.Fase and vw_selecao_partida.Resultado were renamed
from their English source columns (stage_name, result) but never had
their VALUES translated -- Fase still showed "Group Stage", "Round of
16", etc. straight from tournament_stages.csv, and Resultado still
showed raw "WIN"/"DRAW"/"LOSS". The redesign brief explicitly names
the expected Resultado values ("utilize Vitória, Empate e Derrota").

Same fix pattern for both: renamed the raw import to a hidden
'<Column> (Origem)' column and added a calculated column reusing the
original name and lineageTag that SWITCHes the known English values
to Portuguese, with the raw value as a SWITCH fallback for anything
unmapped. Every existing report binding to dim_etapas.Fase or
vw_selecao_partida.Resultado keeps working unchanged since neither
display name nor lineageTag moved.

Also fixes an independently-confirmed pre-existing bug found while
validating this same table: the Vitórias measure referenced
vw_selecao_partida[result], a raw source-column name that isn't a
valid DAX reference target (the model column is Resultado) -- this
measure has been failing to evaluate. Repointed it at the new hidden
Resultado (Origem) column with the original "WIN" comparison, so its
correctness doesn't depend on the Portuguese translation strings.
EOF
)"
```

---

## Phase B — Theme rebuild

### Task 3: Apply the sports-broadcast palette to the report theme

**Files:**
- Modify: `fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json` (via `pbir theme` commands only — never hand-edit this file)

**Interfaces:**
- Consumes: nothing report-specific.
- Produces: a report-wide semantic color set (`good`/`bad`/`neutral`/canvas background/card background/borders/text) and a 7-color qualitative `dataColors` palette that every subsequent page task relies on for consistent styling without per-visual hex literals.

- [ ] **Step 1: Confirm current state**

```bash
cd "BI - Semana da Informática"
grep -o '"#[0-9A-Fa-f]\{6\}"' "fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json" | sort -u
```

Expected: includes `"#7B43A3"` (the purple to be removed) among the current `dataColors`.

- [ ] **Step 2: Set the qualitative data-colors palette**

```bash
pbir theme set-colors "fifa-world-cup-2026.Report" --data-colors '["#0B3D2E","#1F7A4D","#39A96B","#6B776F","#2E5947","#7FBF9E","#A8B5AE"]'
```

- [ ] **Step 3: Set the semantic and canvas colors**

```bash
pbir theme set-colors "fifa-world-cup-2026.Report" \
  --good "#1F7A4D" \
  --bad "#C74343" \
  --neutral "#D4A72C" \
  --accent "#0B3D2E" \
  --table-accent "#0B3D2E" \
  --background "#F3F5F4" \
  --foreground "#17221C" \
  --foreground-neutral-secondary "#6B776F"
```

- [ ] **Step 4: Set the canvas (page) background and card (visual container) background/border globally**

```bash
pbir theme set-formatting "fifa-world-cup-2026.Report" "page.*.background.color" --value "#F3F5F4"
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.background.color" --value "#FFFFFF"
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.border.color" --value "#DCE4DF"
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.border.radius" --value 6
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.show" --value true
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.position" --value Outer
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.preset" --value Bottom
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.color" --value "#17221C"
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.transparency" --value 92
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.shadowBlur" --value 4
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.dropShadow.shadowDistance" --value 1
```

(The existing theme already has `background.show: true` and `border.show: true, radius: 3` at `visualStyles["*"]["*"]` — the background/border commands only change color values and bump the radius from 3 to 6 for the "cantos levemente arredondados" requirement. The `dropShadow` block is new — `transparency: 92` (i.e. 8% opacity) with a small 4px blur and 1px distance is the "sombra muito suave" the brief asks for, not a heavy drop shadow.)

- [ ] **Step 5: Standardize card typography and generic visual titles globally**

```bash
pbir theme set-formatting "fifa-world-cup-2026.Report" "card.*.categoryLabels.show" --value false
pbir theme set-formatting "fifa-world-cup-2026.Report" "card.*.labels.color" --value "#17221C"
pbir theme set-formatting "fifa-world-cup-2026.Report" "card.*.labels.fontSize" --value 28
pbir theme set-formatting "fifa-world-cup-2026.Report" "card.*.labels.bold" --value true
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.title.fontSize" --value 12
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.title.bold" --value true
pbir theme set-formatting "fifa-world-cup-2026.Report" "*.*.title.color" --value "#17221C"
```

Every card in this plan (Tasks 5, 8, 12) sets its own `visualContainerObjects.title` text — that's the "rótulo curto" (a header bar above the visual). Power BI's `card` visual *also* auto-renders its own internal `categoryLabels` (the bound field's display name) directly above the big value, independent of the container title bar — leaving both on would show the same label twice inside a 96px-tall card. `card.*.categoryLabels.show: false` removes that duplication report-wide, once, rather than per-card. `card.*.labels.*` makes the "valor principal em destaque" bold and large; the generic `*.*.title.*` block standardizes every other visual's title bar (charts, tables, the KPI-row cards) to one consistent size/weight/color, per "padronize títulos... fontes, tamanhos, cores."

- [ ] **Step 6: Remove the purple from data-point literals used elsewhere in the theme (if any)**

```bash
grep -o '"#[0-9A-Fa-f]\{6\}"' "fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json" | sort -u
```

Confirm `"#7B43A3"` no longer appears (Step 2's `--data-colors` call replaces the whole array, which is where it lived). If it still appears elsewhere in the file, stop and report the exact `pbir theme colors "fifa-world-cup-2026.Report"` audit output before proceeding — do not hand-edit the JSON.

- [ ] **Step 7: Validate (agent gate)**

```bash
pbir theme validate "fifa-world-cup-2026.Report"
```

Expected: no errors.

- [ ] **Step 8: Commit**

```bash
git add fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json
git commit -m "$(cat <<'EOF'
feat: apply the sports-broadcast palette to the report theme

Dark green (#0B3D2E) as accent/table-header token, #1F7A4D as the
'good' semantic color, #C74343 as 'bad', #D4A72C as 'neutral' (draws/
alerts), light-gray canvas (#F3F5F4) with white card containers
(#FFFFFF), discrete borders (#DCE4DF), slightly rounded corners
(radius 6), and a very soft 8%-opacity drop shadow -- applied
report-wide via the theme so every page inherits it without
per-visual overrides. Card category labels are hidden report-wide
(the container title bar each card sets is the single "rótulo curto"
now, not duplicated by the card's own internal label); card values
are bold/28pt/primary-text-color, and every visual's title bar is
standardized to 12pt/bold/primary-text-color. The old
7-color dataColors array (which included a purple, #7B43A3, used only
in one hard-coded visual override -- removed separately in the page 3
task that touches that visual) is replaced with a 7-color palette
built entirely from the new green/neutral family.
EOF
)"
```

---

### Task 4: Capture the pre-page-redesign validation baseline

**Files:** none (validation only)

**Interfaces:**
- Produces: an error/warning count snapshot used by Task 15 to prove the page redesigns (Tasks 5–14) didn't introduce new structural problems, distinct from the pre-existing `$schema`-missing error on `definition.pbism` (present since the initial commit, out of scope).

- [ ] **Step 1: Capture the baseline**

```bash
cd "BI - Semana da Informática"
pbir validate "fifa-world-cup-2026.Report" --all
```

Record the exact error count, warning count, and per-code warning breakdown shown in the output (as of the last Jogadores-page session: 1 error, 67 warnings — `SCHEMA_DEGRADED 27, VISUAL_UNDERSIZED 24, VISUAL_LEVEL_FILTERS 7, TEXTBOX_HEIGHT_BELOW_FLOOR 6, TOO_MANY_FIELDS 3`). This redesign will delete and recreate most visuals on 3 pages, so the counts will shift — the goal of re-baselining now, after Phase A+B, is to have a same-session reference point immediately before Phase C starts.

---

## Phase C — Page 1: Visão Geral do Torneio

### Task 5: Header band, filter band, and KPI row

**Files:**
- Modify: visuals under `fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/`

**Interfaces:**
- Consumes: `_Measures[Total de Partidas]`, `[Gols Marcados]`, `[Média de Gols/Partida]`, `[Eficiência Ofensiva (Gols/xG)]`; `dim_selecoes[Seleção]`; `dim_etapas[Fase]` (now Portuguese, per Task 2).
- Produces: the page's new header/filter/KPI shell, consumed visually by Task 6 (content row) and Task 7 (table) below it.

- [ ] **Step 1: Turn the existing title textbox into the full-bleed dark-green header band**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/Title.Visual" --x 0 --y 0 --width 1280 --height 56
pbir visuals background "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/Title.Visual" --show --color "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/Title.Visual.text.fontSize" --value 20
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/Title.Visual.text.color" --value "#FFFFFF"
```

(This textbox's paragraph text is already "Visão Geral do Torneio" at 24pt with no header/border/background container shown — Task 3's global `border.show`/`background.show` theme defaults don't apply here because this specific visual already has its own `visualContainerObjects` overrides from the original build; the explicit `pbir visuals background --color` call is what paints the band. The paragraph-level 24pt from the original `objects.general.paragraphs.textRuns.textStyle.fontSize` stays as a fallback; the `text.fontSize`/`text.color` properties set here are a second, higher-priority text style layer — the same one used for the Jogadores footnote's 10pt override.)

- [ ] **Step 2: Move the 2 slicers into their own filter band**

```bash
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/slicer_selecao.Visual" --x 24 --y 72 --width 300 --height 40
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/slicer_fase.Visual" --x 340 --y 72 --width 280 --height 40
```

- [ ] **Step 3: Rebuild the KPI row as 4 plain cards (no Goal/auto-target)**

`kpi_total_partidas` and `kpi_gols_marcados` are already `card` type — just reposition and retitle:

```bash
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_total_partidas.Visual" --x 24 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_total_partidas.Visual.title.text" --value "Total de Partidas"

pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_gols_marcados.Visual" --x 336 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_gols_marcados.Visual.title.text" --value "Gols Marcados"
```

`kpi_media_gols` and `kpi_eficiencia` are `kpi` type today (the redundant-Goal anti-pattern named in the brief — Goal is a torneio-wide reference measure that equals the Indicator whenever no seleção is selected, producing a meaningless "100%"/"1,00" comparison). Delete and recreate as plain cards:

```bash
pbir rm "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_media_gols.Visual" -f
pbir add visual card "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page" -n kpi_media_gols -x 648 -y 128 -w 296 -h 96
pbir visuals bind "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_media_gols.Visual" --add "Values:_Measures.Média de Gols/Partida" --type Measure
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_media_gols.Visual.title.text" --value "Média de Gols por Partida"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_media_gols.Visual.title.show" --value true

pbir rm "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_eficiencia.Visual" -f
pbir add visual card "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page" -n kpi_eficiencia -x 960 -y 128 -w 296 -h 96
pbir visuals bind "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_eficiencia.Visual" --add "Values:_Measures.Eficiência Ofensiva (Gols/xG)" --type Measure
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_eficiencia.Visual.title.text" --value "Eficiência Ofensiva"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/kpi_eficiencia.Visual.title.show" --value true
```

**On "Eficiência Ofensiva ... deve ser recalculada no contexto correto" (brief):** verified this session — `DIVIDE([Gols Marcados], [xG a Favor])` is mathematically correct as written (both operands come from the same `vw_selecao_partida` grain, so the ratio is internally consistent at any filter level). The "1,00" symptom the brief describes came from the `kpi` visual's Goal-vs-Indicator comparison (both evaluate to the torneio-wide average when no seleção is selected), not from the DAX. No measure change — this task's fix is entirely the `kpi` → `card` visual-type swap above.

- [ ] **Step 4: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page" --qa
```

Confirm no new overlap/undersize errors beyond the pre-existing `$schema` baseline error from Task 4.

- [ ] **Step 5: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/Title/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/slicer_selecao/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/slicer_fase/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/kpi_total_partidas/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/kpi_gols_marcados/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/kpi_media_gols/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/kpi_eficiencia/
git commit -m "$(cat <<'EOF'
feat: rebuild page 1 header, filter band, and KPI row

Header: the existing title textbox becomes a full-bleed dark-green
band (#0B3D2E, white 20pt text), left-aligned, containing only the
page title -- no filters/subtitle/date inside it. Filters move to
their own band directly below. The 4 KPIs become plain cards: 2 were
already cards (Total de Partidas, Gols Marcados), 2 were `kpi`-type
visuals with a redundant Goal measure that equals the Indicator
whenever no seleção is selected (Média de Gols/Partida, Eficiência
Ofensiva) -- exactly the "meta automática que repete o próprio valor"
anti-pattern the redesign brief asks to eliminate. Verified Eficiência
Ofensiva's DAX itself is correct; the misleading "1,00" was the kpi
visual's goal-ratio comparison, not a calculation bug.
EOF
)"
```

---

### Task 6: Content row — gols por fase chart and mapa das sedes

**Files:**
- Modify: `chart_gols_fase` and `map_sedes` under `.../Vis_o_Geral_do_Torneio/visuals/`

**Interfaces:**
- Consumes: `dim_etapas[Fase]` (now Portuguese), `_Measures[Gols Marcados]`, `[Média de Gols/Partida]`, `[Total de Partidas]`, `dim_estadios[Cidade]`, `[País-Sede]`.

- [ ] **Step 1: Reposition and clean up the gols-por-fase combo chart**

```bash
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/chart_gols_fase.Visual" --x 24 --y 240 --width 608 --height 224
pbir visuals axis "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/chart_gols_fase.Visual" category --showTitle --title "Fase"
pbir visuals axis "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/chart_gols_fase.Visual" value --showTitle --title "Gols Marcados"
pbir visuals axis "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/chart_gols_fase.Visual" y2 --showTitle --title "Média de Gols/Partida"
```

Category labels stay unrotated by default (Power BI's combo chart only rotates category labels when the plot area is too narrow for the label count — 7 Fase values across a 608px chart do not trigger rotation; no explicit "no rotation" property exists on this visual type to set defensively, so this is a design-time expectation to confirm in Task 15's manual pass, not a settable property).

- [ ] **Step 2: Reposition and align the map**

```bash
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/map_sedes.Visual" --x 648 --y 240 --width 608 --height 224
```

(Already at the same `y` as the chart before this task — this step only updates `y` from 204→240 and `height` from 238→224 to match the new grid; `x`/`width` were already correct and stay unchanged. This keeps it visually aligned to `chart_gols_fase`'s new position, satisfying "o mapa deve ficar visualmente alinhado ao gráfico ao lado.")

- [ ] **Step 3: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page" --qa
```

- [ ] **Step 4: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/chart_gols_fase/ fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/map_sedes/
git commit -m "$(cat <<'EOF'
feat: rebuild page 1 content row (gols por fase + mapa)

Both visuals move to the new y=240 h=224 content-row grid, keeping
them aligned to each other. Added explicit axis titles to the combo
chart (Fase / Gols Marcados / Média de Gols por Partida) so the
dual-scale bar+line reads unambiguously, per the redesign brief.
EOF
)"
```

---

### Task 7: Top 10 seleções table

**Files:**
- Modify: `table_top10` under `.../Vis_o_Geral_do_Torneio/visuals/`

**Interfaces:**
- Consumes: `dim_selecoes[Seleção]`, `_Measures[Pontos Conquistados]`, `[Saldo de Gols]`, `[Gols Marcados]`, `[Aproveitamento (%)]` — unchanged bindings, already sorted descending by Pontos Conquistados then Aproveitamento (%), already TopN-10-filtered (confirmed present from this session's inventory — no new filter needed).

- [ ] **Step 1: Reposition and restyle the header/rows/borders**

```bash
pbir visuals position "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual" --x 24 --y 480 --width 1232 --height 216
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.columnHeaders.backColor" --value "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.columnHeaders.fontColor" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.values.backColorPrimary" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.values.backColorSecondary" --value "#F3F5F4"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.grid.gridHorizontalColor" --value "#DCE4DF"
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.grid.gridVerticalColor" --value "#DCE4DF"
```

- [ ] **Step 2: Fix the hard-coded white data-bar axis color**

The existing `Pontos Conquistados` data bars already use theme tokens (`'good'`/`'bad'`) for their positive/negative fill — correct, no change needed there — but their `axisColor` is a hard-coded `#FFFFFF` literal (found in this session's inventory). Replace it with a theme-consistent border tone:

```bash
pbir visuals cf "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual" --data-bars --field "_Measures.Pontos Conquistados" --positive-color good --negative-color bad
```

(Re-running `--data-bars` with the same field replaces the existing data-bar CF entry — including its `axisColor` — with a fresh one using only theme tokens, no hex literal. Verify with the color grep in Step 4 that `#FFFFFF` no longer appears in this file.)

- [ ] **Step 3: Confirm no total row is showing and the 10 rows fit without scrolling**

```bash
pbir get "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.total.show" 2>&1
```

If this returns `true` (or the property doesn't exist and the visual's default is an ON total row for `tableEx`), turn it off — a total row summing "Pontos Conquistados" or averaging "Aproveitamento (%)" across the top 10 doesn't add analytical value the brief asks to keep:

```bash
pbir set "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page/table_top10.Visual.total.show" --value false
```

At `h=216` with the default `tableEx` row height (~24-28px including the header), 10 data rows + 1 header row fit comfortably (11 × ~28px ≈ 308px of *content* height is more than 216px of *container* height would allow if every row rendered at full default height — but `tableEx` auto-shrinks font/row height to fit `autoSizeColumnWidth`-style before scrolling; the pre-existing `objects.columnHeaders.autoSizeColumnWidth: true` already engages this). If Task 15's manual check finds vertical scrolling still appears, reduce `values.fontSize` (default is usually 10) via `pbir set "...values.fontSize" --value 9` as a follow-up — flag this explicitly in Task 15's checklist rather than guessing blind here, since row-fit cannot be confirmed without rendering.

- [ ] **Step 4: Color gate**

```bash
grep -E '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/table_top10/visual.json"
```

Expected: no output.

- [ ] **Step 5: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Vis_o_Geral_do_Torneio.Page" --qa
pbir validate "fifa-world-cup-2026.Report" --all
```

- [ ] **Step 6: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/Vis_o_Geral_do_Torneio/visuals/table_top10/
git commit -m "$(cat <<'EOF'
feat: restyle page 1 top-10 table (dark-green header, alt rows)

Column headers: dark green background, white text. Alternating row
fill (#FFFFFF / #F3F5F4), discrete #DCE4DF gridlines. Replaced the
Pontos Conquistados data-bar CF entry to drop its hard-coded #FFFFFF
axisColor -- it now only references theme good/bad tokens, same as
before but with zero hex literals. Confirmed the total row is off and
all 10 rows are already reachable without the visual's own filter
list changing (still TopN 10 by Pontos Conquistados, unchanged).
EOF
)"
```

---

## Phase D — Page 2: Perfil da Seleção

### Task 8: Header band, filter band, default seleção, and KPI row

**Files:**
- Modify: visuals under `fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/`
- Create: a new page-level filter on `dim_selecoes[Seleção]`

**Interfaces:**
- Consumes: `_Measures[Pontos Conquistados]`, `[Aproveitamento (%)]`, `[Saldo de Gols]`, `[Eficiência Ofensiva (Gols/xG)]`; `dim_selecoes[Seleção]`; `dim_etapas[Fase]`.
- Produces: a page that always has exactly one seleção in context (never blank), which Task 10's `table_confrontos` totals depend on to stay scoped to that team rather than the whole tournament.

**On the blank-filter requirement:** the brief permits either a "Selecione uma seleção" empty state or a sensible default. A default seleção is achievable natively with a page-level filter; an empty state would require bookmarks and conditional visibility, a much larger addition out of proportion with the rest of this redesign. This plan uses a default seleção: **Mexico** (one of the 3 co-host nations of the 2026 tournament, `team_id=1` in the source data — confirmed present in `dim_selecoes[Seleção]` by direct inspection of `teams.csv`, stored as the literal string `"Mexico"`, unaccented, matching the source data as-is; team names are proper nouns and are not translated elsewhere in this model either).

- [ ] **Step 1: Header band**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/Title.Visual" --x 0 --y 0 --width 1280 --height 56
pbir visuals background "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/Title.Visual" --show --color "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/Title.Visual.text.fontSize" --value 20
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/Title.Visual.text.color" --value "#FFFFFF"
```

- [ ] **Step 2: Filter band**

```bash
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/slicer_selecao.Visual" --x 24 --y 72 --width 300 --height 40
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/slicer_fase.Visual" --x 340 --y 72 --width 280 --height 40
```

- [ ] **Step 3: Add the default-seleção page filter**

```bash
pbir add filter dim_selecoes Seleção -p "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" --type Categorical --values Mexico
```

This sets a *default* selection, not a lock — the slicer stays interactive and the user can pick a different seleção; only the page's *initial* state is guaranteed non-blank. Do not hide or lock this filter (unlike the report-level status filter) — it must stay visible/editable since it's a working default, not a hidden business rule.

- [ ] **Step 4: KPI row — kpi_saldo is already `card`, the other 3 are `kpi`-type (same redundant-Goal pattern as page 1)**

```bash
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_saldo.Visual" --x 648 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_saldo.Visual.title.text" --value "Saldo de Gols"
pbir visuals cf "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_saldo.Visual" --rules --field "_Measures.Saldo de Gols" --rule "gte 0 good" --rule "lt 0 bad" --on labels.color

pbir rm "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_pontos.Visual" -f
pbir add visual card "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" -n kpi_pontos -x 24 -y 128 -w 296 -h 96
pbir visuals bind "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_pontos.Visual" --add "Values:_Measures.Pontos Conquistados" --type Measure
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_pontos.Visual.title.text" --value "Pontos Conquistados"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_pontos.Visual.title.show" --value true

pbir rm "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_aproveitamento.Visual" -f
pbir add visual card "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" -n kpi_aproveitamento -x 336 -y 128 -w 296 -h 96
pbir visuals bind "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_aproveitamento.Visual" --add "Values:_Measures.Aproveitamento (%)" --type Measure
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_aproveitamento.Visual.title.text" --value "Aproveitamento"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_aproveitamento.Visual.title.show" --value true

pbir rm "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_eficiencia.Visual" -f
pbir add visual card "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" -n kpi_eficiencia -x 960 -y 128 -w 296 -h 96
pbir visuals bind "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_eficiencia.Visual" --add "Values:_Measures.Eficiência Ofensiva (Gols/xG)" --type Measure
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_eficiencia.Visual.title.text" --value "Eficiência Ofensiva"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/kpi_eficiencia.Visual.title.show" --value true
```

(`Saldo de Gols` gets the only semantic red/green CF on this page's KPI row — it's the one card that can be genuinely negative; the other 3 are always ≥0 and don't need it.)

- [ ] **Step 5: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" --fields --qa
```

Confirms the new `Mexico` filter's field reference resolves and no new overlap/undersize issues appeared.

- [ ] **Step 6: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/Title/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/slicer_selecao/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/slicer_fase/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/kpi_saldo/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/kpi_pontos/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/kpi_aproveitamento/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/kpi_eficiencia/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/page.json
git commit -m "$(cat <<'EOF'
feat: rebuild page 2 header, filter band, and KPI row

Same header-band/filter-band pattern as page 1. Added a default
Seleção filter (Mexico, a 2026 co-host) so the page is never in a
blank-filter state -- the slicer stays interactive, this only sets
the initial selection, unblocking table_confrontos's totals (fixed
next task) from ever summing across every team in the tournament.
3 of the 4 KPI cards were `kpi`-type with the same redundant-Goal
pattern removed on page 1; converted to plain cards. Saldo de Gols
gets red/green conditional text color since it's the one card here
that can go negative.
EOF
)"
```

---

### Task 9: Content row — gols×xG chart (60%) and dispersão xG chart (40%)

**Files:**
- Modify: `chart_gols_xg` and `chart_dispersao_xg` under `.../8e9d5ca70aa150d0/visuals/`

**Interfaces:**
- Consumes: `vw_selecao_partida[Data da Partida]`, `_Measures[Gols Marcados]`, `[xG a Favor]`, `[xG Contra]`, `dim_selecoes[Seleção]`.

- [ ] **Step 1: Reposition the gols×xG combo chart to the 60% column and reduce label clutter**

```bash
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_gols_xg.Visual" --x 24 --y 240 --width 728 --height 224
pbir visuals axis "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_gols_xg.Visual" category --showTitle --title "Data da Partida"
pbir visuals axis "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_gols_xg.Visual" value --showTitle --title "Gols Marcados"
pbir visuals axis "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_gols_xg.Visual" y2 --showTitle --title "xG a Favor"
```

("Reduza os rótulos exibidos" — this chart's Category is `Data da Partida` (one point per match date for the selected seleção); with the default-seleção filter from Task 8 now guaranteeing a single team's ~4-7 matches instead of the full tournament's 104, category-label crowding is structurally resolved rather than needing a rotation/skip-interval property that this visual type doesn't expose via `pbir`.)

- [ ] **Step 2: Reposition the scatter chart to the 40% column and add axis titles**

```bash
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_dispersao_xg.Visual" --x 768 --y 240 --width 488 --height 224
pbir visuals axis "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_dispersao_xg.Visual" category --showTitle --title "xG a Favor"
pbir visuals axis "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_dispersao_xg.Visual" value --showTitle --title "xG Contra"
```

(This visual's `Category` role holds `dim_selecoes.Seleção` — that's the scatter's point-identity/legend field, not the X-axis; the X-axis is driven by the `X` role's measure, `xG a Favor`. `pbir visuals axis ... category` still targets the correct axis object for a scatter chart's category/X-value axis per the schema — same `axis_type` vocabulary (`category`/`value`/`y2`) applies across chart types.)

- [ ] **Step 3: Add a reference line where technically viable**

```bash
pbir visuals reference-line add "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/chart_dispersao_xg.Visual" --value 0
```

Power BI's native reference-line feature draws a single fixed value on one axis (horizontal or vertical) — not a true diagonal y=x equality line, which this visual type has no native support for and `pbir` has no mechanism to fake (confirmed via `pbir visuals reference-line --help`). This step adds a baseline-zero line instead, which is the closest native equivalent; if it doesn't read as useful in Task 15's manual review, it can be removed with `pbir visuals reference-line remove ... --id 1` without needing to revisit this task.

- [ ] **Step 4: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" --qa
```

- [ ] **Step 5: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/chart_gols_xg/ fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/chart_dispersao_xg/
git commit -m "$(cat <<'EOF'
feat: rebuild page 2 content row (60/40 split, axis titles)

chart_gols_xg to 728px (60%), chart_dispersao_xg to 488px (40%), both
at the new y=240 h=224 grid. Added axis titles to both. Label
crowding on the gols x xG chart is resolved structurally by page 2's
new default-seleção filter (one team's matches, not all 104). A true
diagonal y=x reference line isn't supported by this visual type or
pbir's reference-line command (fixed-value only, one axis) -- added a
zero-baseline reference line as the closest native alternative.
EOF
)"
```

---

### Task 10: Tabela de confrontos — semantic W/D/L colors and scoped totals

**Files:**
- Modify: `table_confrontos` under `.../8e9d5ca70aa150d0/visuals/`

**Interfaces:**
- Consumes: `vw_selecao_partida[Adversário]`, `[Resultado]`; `dim_etapas[Fase]`; `_Measures[Gols Marcados]`, `[Gols Sofridos]`, `[xG a Favor]`.

**On "totais que parecem considerar todo o torneio":** this table has no total row today (confirmed in this session's inventory — no `objects.total` block present, and `tableEx`'s default is total-row-off unless explicitly turned on). The actual risk the brief is flagging is the table's *own rows*, not a total row: with no seleção selected, this detail table listed every match in the tournament, not one team's matches — which Task 8's new default-seleção filter already fixes structurally. This task adds the semantic Resultado coloring and confirms the total-row state explicitly (rather than leaving it implicit).

- [ ] **Step 1: Reposition and restyle header/rows/borders (same pattern as Task 7)**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual" --x 24 --y 480 --width 1232 --height 216
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.columnHeaders.backColor" --value "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.columnHeaders.fontColor" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.values.backColorPrimary" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.values.backColorSecondary" --value "#F3F5F4"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.grid.gridHorizontalColor" --value "#DCE4DF"
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.grid.gridVerticalColor" --value "#DCE4DF"
```

- [ ] **Step 2: Semantic color the Resultado column (WIN/DRAW/LOSS)**

```bash
pbir schema describe tableEx columnFormatting
```

Confirm the container/property for a per-column font-color CF (expected: `columnFormatting.fontColor`, mirroring the `columnHeaders`/`values` pattern already confirmed this session). Then:

```bash
pbir visuals cf "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual" --rules --field "vw_selecao_partida.Resultado" --rule "eq Vitória good" --rule "eq Empate neutral" --rule "eq Derrota bad" --on columnFormatting.fontColor
```

(Rule values are the **translated** strings — Task 2 already changed what `vw_selecao_partida[Resultado]` returns to "Vitória"/"Empate"/"Derrota"; matching against the old "WIN"/"DRAW"/"LOSS" here would silently match nothing.)

If `--on columnFormatting.fontColor` is rejected, use whatever container/property `schema describe` printed in place of it — this is the one property path in this plan not independently confirmed by a prior successful call this session, so verify before treating it as done.

- [ ] **Step 3: Confirm total-row state**

```bash
pbir get "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.total.show" 2>&1
```

If `true`, turn it off — summing `Gols Marcados`/`Gols Sofridos`/`xG a Favor` across a match-by-match detail table doesn't add information beyond what the KPI row already states in aggregate:

```bash
pbir set "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page/table_confrontos.Visual.total.show" --value false
```

- [ ] **Step 4: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/8e9d5ca70aa150d0.Page" --qa
pbir validate "fifa-world-cup-2026.Report" --all
```

- [ ] **Step 5: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/8e9d5ca70aa150d0/visuals/table_confrontos/
git commit -m "$(cat <<'EOF'
feat: restyle page 2 confrontos table and add W/D/L semantic colors

Same dark-green header / alternating rows / discrete gridlines as
page 1's top-10 table. Resultado column font color now follows the
good/neutral/bad theme tokens by Vitória/Empate/Derrota (translated
in Task 2). This table never had
a total row of its own -- the "totals considering the whole
tournament" risk named in the brief was actually the table's rows
themselves showing every match with no seleção selected, already
fixed by page 2's new default-seleção filter (previous task).
EOF
)"
```

---

## Phase E — Page 3: Mérito ou Sorte?

### Task 11: Header band and filter band

**Files:**
- Modify: `Title_1`, `slicer_selecao`, `slicer_fase` under `fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/`

**Interfaces:**
- Consumes: nothing new.

- [ ] **Step 1: Header band**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/Title_1.Visual" --x 0 --y 0 --width 1280 --height 56
pbir visuals background "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/Title_1.Visual" --show --color "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/Title_1.Visual.text.fontSize" --value 20
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/Title_1.Visual.text.color" --value "#FFFFFF"
```

- [ ] **Step 2: Filter band**

```bash
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/slicer_selecao.Visual" --x 24 --y 72 --width 300 --height 40
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/slicer_fase.Visual" --x 340 --y 72 --width 280 --height 40
```

- [ ] **Step 3: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/74ab19268b6858b7.Page" --qa
```

- [ ] **Step 4: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/Title_1/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/slicer_selecao/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/slicer_fase/
git commit -m "$(cat <<'EOF'
feat: rebuild page 3 header and filter band

Same header-band/filter-band pattern as pages 1-2.
EOF
)"
```

---

### Task 12: KPI row (already all `card` type — reposition and relabel only)

**Files:**
- Modify: `kpi_xpts`, `kpi_pontos_acima`, `kpi_recuperados`, `kpi_estabilidade` under `.../74ab19268b6858b7/visuals/`

**Interfaces:**
- Consumes: `_Measures[Pontos Esperados (xPts)]`, `[Pontos Acima do Esperado]`, `[Pontos Recuperados]`, `[Estabilidade da Escalação (%)]` — the last one now correctly scaled per Task 1.

- [ ] **Step 1: Reposition and relabel**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_xpts.Visual" --x 24 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_xpts.Visual.title.text" --value "Pontos Esperados"

pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_pontos_acima.Visual" --x 336 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_pontos_acima.Visual.title.text" --value "Pontos Acima do Esperado"
pbir visuals cf "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_pontos_acima.Visual" --rules --field "_Measures.Pontos Acima do Esperado" --rule "gte 0 good" --rule "lt 0 bad" --on labels.color

pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_recuperados.Visual" --x 648 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_recuperados.Visual.title.text" --value "Pontos Recuperados"

pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_estabilidade.Visual" --x 960 --y 128 --width 296 --height 96
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/kpi_estabilidade.Visual.title.text" --value "Estabilidade da Escalação"
```

(`Pontos Acima do Esperado` is the one card here that can be genuinely negative — same red/green CF pattern as `Saldo de Gols` on page 2.)

**On "remova casas decimais desnecessárias... pontos devem aparecer como números inteiros":** this applies to literal counts — `Pontos Conquistados` (page 2) and `Pontos Recuperados` (this page) are already integer-formatted (`#,##0`), correctly, no change needed. `Pontos Esperados (xPts)` and `Pontos Acima do Esperado` are **not** literal counts — they're a Poisson-model statistical estimate and its residual against the actual score (`formatString: #,##0.00`, 2 decimals). Rounding these to integers would erase the entire point of an "expected points" model (e.g. distinguishing 1.8 expected points from 2.1). Left unchanged.

- [ ] **Step 2: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/74ab19268b6858b7.Page" --qa
```

- [ ] **Step 3: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/kpi_xpts/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/kpi_pontos_acima/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/kpi_recuperados/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/kpi_estabilidade/
git commit -m "$(cat <<'EOF'
feat: rebuild page 3 KPI row

All 4 cards were already `card`-type (no redundant-Goal cleanup
needed here, unlike pages 1-2). Repositioned to the new grid and
retitled to the brief's shorter label wording. Pontos Acima do
Esperado gets red/green conditional text color, the one KPI on this
page that can go negative. Estabilidade da Escalação now reflects
the double-scaling fix from Task 1 automatically -- this task only
touches the visual, not the measure.
EOF
)"
```

---

### Task 13: Content row — barras divergentes and dispersão xPts × pontos reais

**Files:**
- Modify: `chart_pontos_acima`, `chart_xpts_reais` under `.../74ab19268b6858b7/visuals/`

**Interfaces:**
- Consumes: `dim_selecoes[Seleção]`, `_Measures[Pontos Acima do Esperado]`, `[Pontos Esperados (xPts)]`, `[Pontos Reais (xPts)]`.

- [ ] **Step 1: Reposition the divergent bar chart, sort it, and replace the hard-coded purple**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_pontos_acima.Visual" --x 24 --y 240 --width 608 --height 224
pbir visuals sort "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_pontos_acima.Visual" --field "_Measures.Pontos Acima do Esperado" --direction Descending
pbir visuals axis "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_pontos_acima.Visual" value --showTitle --title "Pontos Acima do Esperado"
```

The existing conditional fill (`≥0 → theme 'good'` token, `<0 → hard-coded #7B43A3` purple) needs its negative branch replaced:

```bash
pbir get "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_pontos_acima.Visual.dataPoint.fill.cf" 2>&1
```

Confirm the CF entry's structure (2 cases, `ComparisonKind 1` ≥0 and `ComparisonKind 3` <0, per this session's inventory), then replace the whole conditional-fill entry so both branches use theme tokens and none use a literal hex:

```bash
pbir visuals cf "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_pontos_acima.Visual" --rules --field "_Measures.Pontos Acima do Esperado" --rule "gte 0 good" --rule "lt 0 bad"
```

- [ ] **Step 2: Reposition the scatter chart, add axis titles**

```bash
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_xpts_reais.Visual" --x 648 --y 240 --width 608 --height 224
pbir visuals axis "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_xpts_reais.Visual" category --showTitle --title "Pontos Esperados (xPts)"
pbir visuals axis "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_xpts_reais.Visual" value --showTitle --title "Pontos Reais"
```

- [ ] **Step 3: Diagonal equality line — same native limitation as Task 9**

```bash
pbir visuals reference-line add "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/chart_xpts_reais.Visual" --value 0
```

Same caveat as page 2's scatter (Task 9, Step 3): no native diagonal y=x line exists for this visual type via `pbir`. A zero-baseline reference line is the closest native substitute; document as a known limitation in Task 15 rather than attempting an unsupported workaround.

- [ ] **Step 4: Color gate**

```bash
grep -E '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/chart_pontos_acima/visual.json"
```

Expected: no output (the `#7B43A3` from Step 1 is gone).

- [ ] **Step 5: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/74ab19268b6858b7.Page" --qa
```

- [ ] **Step 6: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/chart_pontos_acima/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/chart_xpts_reais/
git commit -m "$(cat <<'EOF'
feat: rebuild page 3 content row, remove the last hard-coded purple

chart_pontos_acima: repositioned, sorted descending by Pontos Acima
do Esperado for easier comparison, and its conditional fill's
negative branch (previously a hard-coded #7B43A3 purple) now uses the
theme's 'bad' token, matching its positive branch's existing 'good'
token -- this was the last hex color literal anywhere in the report's
3 original pages. chart_xpts_reais: repositioned, axis titles added.
Neither visual type nor pbir's reference-line command support a true
diagonal y=x equality line (fixed-value, single-axis only) -- added a
zero-baseline line as the closest native alternative, same limitation
as page 2's scatter chart.
EOF
)"
```

---

### Task 14: Tabela de estabilidade and footnote

**Files:**
- Modify: `table_estabilidade`, `Title_2` under `.../74ab19268b6858b7/visuals/`

**Interfaces:**
- Consumes: `dim_selecoes[Seleção]`, `_Measures[Estabilidade da Escalação (%)]` (now correctly scaled per Task 1).

- [ ] **Step 1: Reposition and restyle (same pattern as Tasks 7 and 10), full available width**

```bash
cd "BI - Semana da Informática"
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual" --x 24 --y 480 --width 1232 --height 192
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.columnHeaders.backColor" --value "#0B3D2E"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.columnHeaders.fontColor" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.values.backColorPrimary" --value "#FFFFFF"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.values.backColorSecondary" --value "#F3F5F4"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.grid.gridHorizontalColor" --value "#DCE4DF"
pbir set "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/table_estabilidade.Visual.grid.gridVerticalColor" --value "#DCE4DF"
```

("A tabela inferior deve ocupar o espaço disponível de forma equilibrada, sem um grande bloco branco à direita" — this table has exactly 2 columns (`Seleção`, `Estabilidade da Escalação (%)`) at the full 1232px width; with `autoSizeColumnWidth: true` already set, Power BI's `tableEx` distributes width proportionally to content rather than leaving a fixed narrow block — this is the table's existing behavior, unaffected by this task, and does not need a new property. If Task 15's manual review finds the 2 columns still cramped to the left with visible empty space on the right, that means `autoSizeColumnWidth` isn't achieving the desired effect for a 2-column table and `customColumnWidth` with explicit per-column widths would be the follow-up — flag it there rather than guessing here.)

- [ ] **Step 2: Format Estabilidade as a real percentage, max 1 decimal**

Already `formatString: 0.0%` on the measure itself (1 decimal, correct per Task 1 — this table binds the measure, not a raw column, so no table-level format override is needed or should be added).

- [ ] **Step 3: Reposition the footnote to match the shortened table**

```bash
pbir visuals position "fifa-world-cup-2026.Report/74ab19268b6858b7.Page/Title_2.Visual" --x 24 --y 678 --width 1232 --height 18
```

- [ ] **Step 4: Color gate and validate (agent gate)**

```bash
grep -E '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/"
pbir validate "fifa-world-cup-2026.Report/74ab19268b6858b7.Page" --qa
```

- [ ] **Step 5: Commit**

```bash
git add fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/table_estabilidade/ fifa-world-cup-2026.Report/definition/pages/74ab19268b6858b7/visuals/Title_2/
git commit -m "$(cat <<'EOF'
feat: restyle page 3 estabilidade table, reposition footnote

Same dark-green header / alternating rows / discrete gridlines as
the other 2 tables. Estabilidade da Escalação (%) already formats
correctly (0.0%, 1 decimal) now that Task 1's DAX fix removed the
double-scaling -- no table-level override needed. Footnote moves down
to match the table's new (shorter, to make room for it) height.
EOF
)"
```

---

## Phase F — Final validation and push

### Task 15: Full validation pass, push, and user acceptance checklist

**Files:** none (validation + git operations only)

**Interfaces:**
- Consumes: everything from Tasks 1–14.
- Produces: a pushed `main` branch, or a documented list of failures if something doesn't pass.

- [ ] **Step 1: Model validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` on the final state. Zero errors expected.

- [ ] **Step 2: Report validation (agent gate)**

```bash
cd "BI - Semana da Informática"
pbir validate "fifa-world-cup-2026.Report" --all
```

Compare against Task 4's baseline. The pre-existing `$schema` error on `definition.pbism` will still be there (out of scope, unrelated to this redesign). Warning counts will shift (visuals were deleted/recreated throughout) — investigate any *new warning code* that wasn't in the Task 4 baseline; a shifted *count* within the same 5 pre-existing codes is expected and not itself a regression.

- [ ] **Step 3: Report-wide color gate**

```bash
grep -rE '#[0-9A-Fa-f]{6}' fifa-world-cup-2026.Report/definition/pages/ 2>/dev/null
```

Expected: **zero** hits — both pre-existing hex literals (`#7B43A3` in `chart_pontos_acima`, `#FFFFFF` in `table_top10`) were removed in Tasks 13 and 7 respectively, and no new page/task introduced a hex literal (the Jogadores page, page 4, was not touched by this plan and was already clean).

- [ ] **Step 4: Theme validation**

```bash
pbir theme validate "fifa-world-cup-2026.Report"
```

- [ ] **Step 5: Push**

```bash
git push
git log --oneline -20
```

- [ ] **Step 6: If any gate in Steps 1–4 failed**

Do not push. Fix the specific failure, re-run only the gate that failed, then retry Step 5.

---

### Task 16: (informational — no checkbox) What this plan does not and cannot verify in this environment

This environment has no Power BI Desktop, no live XMLA/TOM connection, and `pbir desktop screenshot` is Windows-only (all confirmed empirically this session, most recently via `pbir model -q` failing with "Local model is not open in Power BI Desktop"). This means the following items from the brief's "Verificação final" section are **not executable by this plan** and are handed to the user below:

1. **Render/export all pages and review visually** (brief steps 1–3) — cannot render or screenshot; every layout/spacing/color claim in Tasks 5–14 is verified by reading the resulting JSON's numeric properties, not by looking at a rendered page.
2. **Compare against the original PDF** (brief step 6) — no PDF was provided to this session and no rendering capability exists to produce a comparable one.
3. **Test filters, cross-highlighting, and tooltips interactively** (brief step 4) — `pbir validate --fields` confirms field references resolve; it cannot confirm that clicking a slicer actually narrows the KPI row live.
4. **Whether the top-10 table (Task 7) and the estabilidade table (Task 14) actually avoid a scrollbar or a lopsided empty column** — flagged inline in both tasks as review items, since `tableEx`'s row/column auto-fit behavior is a runtime layout computation this environment cannot simulate.
5. **Whether category-axis labels on the combo charts (Tasks 6, 9) truly render without excessive rotation** — depends on rendered font metrics, not inspectable statically.

### Task 17: User acceptance checklist (manual — not agent-executable)

Hand this checklist to the user once Task 15 has pushed successfully. Every item requires opening `fifa-world-cup-2026.pbip` in Power BI Desktop on Windows.

- [ ] Report opens with no error dialog; theme loads (dark green header bands visible on all 3 pages, light-gray canvas, white cards).
- [ ] Each page's header band shows only the page title, white text, left-aligned, no other content inside it.
- [ ] Filters sit in their own band directly below the header, consistently sized/aligned across all 3 pages.
- [ ] Page 1: 4 KPI cards render as plain cards (no goal/target comparison anywhere) with integers for Total de Partidas/Gols Marcados, 2 decimals for Média de Gols por Partida, and Eficiência Ofensiva showing a plausible value (not "1,00" — cross-check against `SUM(goals_for)/DISTINCTCOUNT(match_id) ≈ 2.96` type arithmetic using the actual current totals).
- [ ] Page 1: gols-por-fase chart shows 7 translated Fase labels ("Fase de Grupos", "Rodada de 32", "Oitavas de Final", "Quartas de Final", "Semifinais", "Disputa de 3º Lugar", "Final") without rotation/overlap; mapa is visually aligned to it.
- [ ] Page 1: top-10 table shows exactly 10 rows with no vertical scrollbar and no total row.
- [ ] Page 2: opens with Mexico pre-selected (not blank); selecting a different seleção updates the KPI row, both charts, and the confrontos table together.
- [ ] Page 2: confrontos table's Resultado column shows Vitória/Empate/Derrota (not WIN/DRAW/LOSS), colored accordingly; no total row.
- [ ] Page 3: Estabilidade da Escalação shows a plausible percentage (0–100%, not 7000%+) on both the KPI card and the table.
- [ ] Page 3: divergent bar chart bars are green (≥0) or red (<0) — no purple anywhere; sorted descending.
- [ ] Page 3: both scatter charts (page 2 and page 3) show axis titles; confirm whether the zero-baseline reference line is useful or should be removed (`pbir visuals reference-line remove ... --id 1`) — a true diagonal equality line was not achievable with native visuals in this environment.
- [ ] No text is cut off, no visual overlaps another, no large unexplained empty area on any of the 3 pages.
- [ ] Screenshot each page and do a side-by-side comparison against the pre-redesign PDF (if the user still has it) to confirm the intended visual identity landed as expected.
