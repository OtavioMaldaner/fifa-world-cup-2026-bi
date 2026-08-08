# Dashboard Power BI — Copa do Mundo 2026 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the empty `fifa-world-cup-2026.SemanticModel` / `fifa-world-cup-2026.Report` PBIP project into the 3-page Copa do Mundo 2026 dashboard specified in `docs/superpowers/specs/2026-08-08-dashboard-power-bi-design.md`: a DAX measure layer, an IFRS-green theme, and Visão Geral / Perfil da Seleção / Análises Avançadas pages.

**Architecture:** Direct TMDL file authoring for the semantic model (measures, hidden/renamed objects, relationship cleanup), driven by small Python scripts where the edit is mechanical and repetitive (date-table removal touches 8 files identically; 22 measures share one generation pattern). Report pages and visuals are built exclusively through the `pbir` CLI — never hand-edited JSON. Every task ends in a state that either `pbip-validator` (model) or `pbir validate` (report) can check statically.

**Tech Stack:** Power BI Project (PBIP/TMDL/PBIR), Python 3 (stdlib only) for the two generator scripts, `pbir-cli` (installed via `uv`), git.

## Global Constraints

- All work happens in the `fifa-world-cup-2026-bi` repository. Every commit uses `git -C "BI - Semana da Informática"` — never the Databricks repo.
- **No Tabular Editor CLI (`te`), TOM, or Power BI Desktop is available in this environment** (confirmed empirically: `te`, `uv`, `pipx` all absent; platform is Linux/WSL2). Model changes go through direct TMDL editing, validated statically with the `pbip-validator` agent — not through `te mv --save` or a live AS session. Report changes go through the `pbir` CLI (installed in Task 1), which reads/writes the on-disk PBIR/TMDL files without needing Desktop.
- **Rename scope is reduced from the spec's original "all columns."** Per user decision during planning (2026-08-08), only the 7 columns actually bound by an approved visual are renamed: `dim_selecoes.team_name`, `dim_etapas.stage_name`, `dim_estadios.city`, `dim_estadios.country`, `vw_selecao_partida.opponent_name`, `vw_selecao_partida.result`, `vw_selecao_partida.match_date`. Confirmed safe: none of these 7 appear as `fromColumn`/`toColumn` in `relationships.tmdl` (verified by grep before writing this plan), so renaming them cannot break a relationship.
- **22 measures, not 21.** The spec's P3 secondary visual ("Dispersão secundária: xPts × Pontos reais") needs an "actual points" measure that the approved 21 don't include. Added `Pontos Reais (xPts) = SUM(vw_xpts_selecao_partida[actual_points])` to the `_Avançado` folder — same table, same pattern as its 5 siblings. Flagged here rather than silently added.
- Verde primário `#339645`; full 7-color palette and structural tokens (`#333333` text, `#D8D8D8` grid, `#C8C6C4` border, `#F3F3F3` background) as specified in the spec's "Identidade visual e tema" section.
- Page canvas: 1280×720, margin 24, gap 16 (confirmed in `page.json`).
- **Agent gate vs. user gate.** DAX values can only be evaluated by a running Analysis Services instance (Power BI Desktop or a published model) — neither exists here. Visual rendering can only be inspected via `pbir desktop screenshot`, which is Windows-only. Every task's "Verify" step says explicitly whether it's something this plan's executor can run (agent gate) or something only the user can run in their own Power BI Desktop (user gate). Task 14 collects every user gate into one checklist — do not claim the dashboard "works" before the user runs it.

---

## Phase A — Semantic model

### Task 1: Remove the auto-generated date tables

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/model.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/relationships.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_xpts_selecao_partida.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_pontos_recuperados.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_estabilidade_escalacao.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/ft_partidas.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/ft_estatisticas_equipe.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/ft_estatisticas_jogador.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/dim_jogadores.tmdl`
- Delete: the 13 `tables/LocalDateTable_*.tmdl` files and `tables/DateTableTemplate_3ba18799-9467-40c7-ab87-c7714c2f6086.tmdl` (14 files total)
- Script: `/tmp/remove_date_tables.py` (throwaway, not committed)

**Interfaces:**
- Produces: a model with exactly 14 tables (10 base + 4 views), zero `LocalDateTable_*`/`DateTableTemplate_*` tables, zero relationships or column `variation` blocks referencing them. Later tasks assume this table count and assume `model.tmdl`'s `annotation __PBI_TimeIntelligenceEnabled` is `0`.

- [x] **Step 1: Confirm the exact scope before touching anything**

Run from the repo root:

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^relationship " relationships.tmdl        # expect 44
grep -c "LocalDateTable" relationships.tmdl         # expect 13
grep -rl "variation Variation" tables/ | wc -l      # expect 8 files
grep -c "variation Variation" tables/*.tmdl | grep -v ":0"
```

Expected output for the last command (13 blocks total across 8 files):
```
tables/dim_jogadores.tmdl:1
tables/ft_estatisticas_equipe.tmdl:1
tables/ft_estatisticas_jogador.tmdl:1
tables/ft_partidas.tmdl:2
tables/vw_estabilidade_escalacao.tmdl:2
tables/vw_pontos_recuperados.tmdl:2
tables/vw_selecao_partida.tmdl:2
tables/vw_xpts_selecao_partida.tmdl:2
```

If any count differs, stop — the file layout has changed since this plan was written and the script below must be adjusted before running.

- [x] **Step 2: Write and run the removal script**

```python
# /tmp/remove_date_tables.py
import re
from pathlib import Path

ROOT = Path("fifa-world-cup-2026.SemanticModel/definition")

DATE_TABLE_FILES = [
    "LocalDateTable_0d324503-c34f-4524-acc5-28b1afab3dae",
    "LocalDateTable_12f77402-86e6-4d72-9d12-02eff9393790",
    "LocalDateTable_6b8ed32c-1356-401a-9638-b5ddb55bcd71",
    "LocalDateTable_7133a686-cba9-4088-81e0-500a1cd8bc4d",
    "LocalDateTable_74965fb4-82a1-4e74-8791-aaa269062309",
    "LocalDateTable_7dd71d78-dec5-4119-8983-4188cf14a36e",
    "LocalDateTable_803eef3e-4c84-40d5-880a-fe82aaefb307",
    "LocalDateTable_b7832c73-c6ac-40f6-854d-4d3c60acb2ad",
    "LocalDateTable_b86e0c8e-7aa2-4741-b584-6bf52904df4f",
    "LocalDateTable_b95570d7-d2e9-4442-95b9-0120b68f00aa",
    "LocalDateTable_d0568c74-a0cb-417b-b2a3-aea01fd807f5",
    "LocalDateTable_e62b1ceb-f6ce-478f-abbe-961dbaaecce6",
    "LocalDateTable_f6dc5d06-5aa9-479e-93d8-bec8e68d4462",
    "DateTableTemplate_3ba18799-9467-40c7-ab87-c7714c2f6086",
]

VARIATION_FILES = [
    "vw_selecao_partida", "vw_xpts_selecao_partida", "vw_pontos_recuperados",
    "vw_estabilidade_escalacao", "ft_partidas", "ft_estatisticas_equipe",
    "ft_estatisticas_jogador", "dim_jogadores",
]

VARIATION_RE = re.compile(
    r"\t\tvariation Variation\n"
    r"\t\t\tisDefault\n"
    r"\t\t\trelationship: [0-9a-f-]+\n"
    r"\t\t\tdefaultHierarchy: LocalDateTable_[0-9a-f-]+\.'Hierarquia de datas'\n"
    r"\n"
)

# 1. model.tmdl: drop the 14 `ref table` lines, disable time intelligence
model_path = ROOT / "model.tmdl"
model_text = model_path.read_text()
before = model_text
for name in DATE_TABLE_FILES:
    model_text = model_text.replace(f"ref table {name}\n", "")
model_text = model_text.replace(
    "annotation __PBI_TimeIntelligenceEnabled = 1",
    "annotation __PBI_TimeIntelligenceEnabled = 0",
)
assert model_text != before, "model.tmdl: no ref table lines were removed"
model_path.write_text(model_text)

# 2. relationships.tmdl: drop every relationship block mentioning a date table
rel_path = ROOT / "relationships.tmdl"
blocks = rel_path.read_text().split("\n\n")
kept = [b for b in blocks if "LocalDateTable" not in b]
removed = len(blocks) - len(kept)
assert removed == 13, f"expected to remove 13 relationship blocks, removed {removed}"
rel_path.write_text("\n\n".join(kept))

# 3. table files: strip the variation block from each affected column
for name in VARIATION_FILES:
    p = ROOT / "tables" / f"{name}.tmdl"
    text = p.read_text()
    new_text, n = VARIATION_RE.subn("", text)
    assert n > 0, f"{name}.tmdl: no variation block matched"
    p.write_text(new_text)

# 4. delete the 14 date-table files
for name in DATE_TABLE_FILES:
    (ROOT / "tables" / f"{name}.tmdl").unlink()

print("Date-table removal complete.")
```

Run it from the `BI - Semana da Informática` directory:

```bash
cd "BI - Semana da Informática"
python3 /tmp/remove_date_tables.py
```

Expected output: `Date-table removal complete.` with no `AssertionError`.

- [x] **Step 3: Verify the counts by hand**

```bash
cd "fifa-world-cup-2026.SemanticModel/definition"
grep -c "^relationship " relationships.tmdl                  # expect 31 (44 - 13)
grep -c "LocalDateTable\|DateTableTemplate" relationships.tmdl model.tmdl  # expect 0 in both
grep -rc "variation Variation" tables/*.tmdl | grep -v ":0"   # expect no output
ls tables/ | grep -c "LocalDateTable\|DateTableTemplate"      # expect 0
grep "__PBI_TimeIntelligenceEnabled" ../definition/model.tmdl # expect "= 0"
cd ../..
```

- [x] **Step 4: Static validation (agent gate)**

Dispatch the `pbip-validator` agent (or run its underlying check directly if invoked as a skill) against `fifa-world-cup-2026.SemanticModel`. It must report zero TMDL syntax errors and zero dangling relationship/variation references. Fix any reported issue before continuing — do not proceed to Task 2 with a validator error outstanding.

- [x] **Step 5: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/
git commit -m "$(cat <<'EOF'
feat: remove auto-generated date tables from the semantic model

A 5-week tournament has no use for automatic date hierarchies. Removes
the 13 LocalDateTable_* tables, the DateTableTemplate_* template, their
13 relationships, and the 13 column-level date variations that pointed
to them. Disables __PBI_TimeIntelligenceEnabled so Desktop does not
regenerate them on the next data load.
EOF
)"
```

---

### Task 2: Hide `ft_partidas` from the field list

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/ft_partidas.tmdl`

**Interfaces:**
- Consumes: Task 1's cleaned-up `ft_partidas.tmdl` (no `variation` blocks left).
- Produces: `ft_partidas` marked `isHidden` — later tasks (report-level filter in Task 8) still reference `ft_partidas[status]` by name; a hidden table remains fully usable by filters and DAX, it just doesn't appear in the Desktop field-list tree.

- [x] **Step 1: Add `isHidden` to the table declaration**

Read the current first lines of the file to confirm the exact declaration line, then edit:

```tmdl
// before
table ft_partidas
	lineageTag: <existing-guid>

// after
table ft_partidas
	isHidden
	lineageTag: <existing-guid>
```

- [x] **Step 2: Verify**

```bash
grep -A1 "^table ft_partidas" "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition/tables/ft_partidas.tmdl"
```

Expected: `isHidden` on the line immediately after `table ft_partidas`.

- [x] **Step 3: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/ft_partidas.tmdl
git commit -m "$(cat <<'EOF'
feat: hide ft_partidas from the field list

ft_partidas keeps its 1:N relationship to vw_selecao_partida (needed
for the report-level status filter to propagate) but every dimension
relationship off it is inactive, so a measure or slicer built directly
on it silently ignores segmentation. Hiding it stops anyone from
dragging it into a visual by accident; vw_selecao_partida is the
correct source for match-level data.
EOF
)"
```

---

### Task 3: Rename the 7 report-facing columns

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/dim_selecoes.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/dim_etapas.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/dim_estadios.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl`

**Interfaces:**
- Produces: the field names every later report task binds to. Use these exact names (with the quoting shown) in every `pbir visuals bind` / `pbir add filter` call from Task 8 onward:
  - `dim_selecoes[Seleção]` (was `team_name`)
  - `dim_etapas[Fase]` (was `stage_name`, now also `sortByColumn: stage_id`)
  - `dim_estadios[Cidade]` (was `city`)
  - `dim_estadios['País-Sede']` (was `country`)
  - `vw_selecao_partida[Adversário]` (was `opponent_name`)
  - `vw_selecao_partida[Resultado]` (was `result`; cell values stay `WIN`/`DRAW`/`LOSS` — no value translation in this plan, see note below)
  - `vw_selecao_partida['Data da Partida']` (was `match_date`)

**Why these 7 and not the full ~150:** see Global Constraints. No other column is bound by an approved visual, so renaming further would add risk for zero visible benefit.

**Known limitation, not fixed here:** renaming a column changes its *label*, not the *values* inside it. `dim_etapas[Fase]` values (`"Group Stage"`, `"Round of 32"`, …) and `vw_selecao_partida[Resultado]` values (`WIN`/`DRAW`/`LOSS`) stay in English — the Genie's own metadata flags this exact confusion risk ("Nunca chame o Round of 32 de oitavas"). Translating cell values requires a calculated column or Power Query step, which is out of scope for this plan. Axis labels on the "Gols por Fase" chart and the "Resultado" table column will show English text; call this out when presenting the report.

- [x] **Step 1: `dim_selecoes.team_name` → `Seleção`**

```tmdl
// before
	column team_name
		dataType: string
		lineageTag: f95803d4-88b9-4c10-9d86-63d3b72a113b
		summarizeBy: none
		sourceColumn: team_name

// after
	column 'Seleção'
		dataType: string
		lineageTag: f95803d4-88b9-4c10-9d86-63d3b72a113b
		summarizeBy: none
		sourceColumn: team_name
```

(`sourceColumn` stays `team_name` — that is the Power Query output column name, unrelated to the display name.)

- [x] **Step 2: `dim_etapas.stage_name` → `Fase`, with `sortByColumn`**

```tmdl
// before
	column stage_name
		dataType: string
		lineageTag: e4e7c4e2-43ee-4e4a-a9a3-73823332f256
		summarizeBy: none
		sourceColumn: stage_name

// after
	column Fase
		dataType: string
		lineageTag: e4e7c4e2-43ee-4e4a-a9a3-73823332f256
		summarizeBy: none
		sortByColumn: stage_id
		sourceColumn: stage_name
```

`stage_id` (1=Group Stage … 7=Final) is already in tournament order — confirmed against `tournament_stages.csv` before writing this plan — so this one property addition is what makes the P1 "Gols por Fase" chart sort correctly instead of alphabetically.

- [x] **Step 3: `dim_estadios.city` → `Cidade`, `country` → `'País-Sede'`**

```tmdl
// before
	column city
		dataType: string
		lineageTag: 039621f7-af57-443a-9ac0-fc89d11aa8f1
		summarizeBy: none
		sourceColumn: city

// after
	column Cidade
		dataType: string
		lineageTag: 039621f7-af57-443a-9ac0-fc89d11aa8f1
		summarizeBy: none
		sourceColumn: city
```

```tmdl
// before
	column country
		dataType: string
		lineageTag: ee6ace45-d2bf-4703-89a2-8ad1f236c499
		summarizeBy: none
		sourceColumn: country

// after
	column 'País-Sede'
		dataType: string
		lineageTag: ee6ace45-d2bf-4703-89a2-8ad1f236c499
		summarizeBy: none
		sourceColumn: country
```

- [x] **Step 4: `vw_selecao_partida.opponent_name` → `Adversário`, `.result` → `Resultado`, `.match_date` → `'Data da Partida'`**

```tmdl
// before
	column opponent_name
		dataType: string
		lineageTag: eb3c03bc-8c77-4b72-b515-41a608f164f7
		summarizeBy: none
		sourceColumn: opponent_name

// after
	column Adversário
		dataType: string
		lineageTag: eb3c03bc-8c77-4b72-b515-41a608f164f7
		summarizeBy: none
		sourceColumn: opponent_name
```

```tmdl
// before
	column result
		dataType: string
		lineageTag: 72821297-1f23-4d87-b72e-8dc8ccf3d194
		summarizeBy: none
		sourceColumn: result

// after
	column Resultado
		dataType: string
		lineageTag: 72821297-1f23-4d87-b72e-8dc8ccf3d194
		summarizeBy: none
		sourceColumn: result
```

```tmdl
// before (note: the variation block under this column was already removed in Task 1)
	column match_date
		dataType: dateTime
		formatString: Long Date
		lineageTag: 2fbbeeed-be57-4d11-84a6-ca18ca542fea
		summarizeBy: none
		sourceColumn: match_date

		annotation SummarizationSetBy = Automatic

		annotation UnderlyingDateTimeDataType = Date

// after
	column 'Data da Partida'
		dataType: dateTime
		formatString: Long Date
		lineageTag: 2fbbeeed-be57-4d11-84a6-ca18ca542fea
		summarizeBy: none
		sourceColumn: match_date

		annotation SummarizationSetBy = Automatic

		annotation UnderlyingDateTimeDataType = Date
```

- [x] **Step 5: Confirm none of the 7 renamed columns appear in `relationships.tmdl`**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -E "team_name|stage_name|\.city|\.country|opponent_name|\.result|match_date" relationships.tmdl
cd ../../..
```

Expected: no output. If anything matches, stop — a relationship references one of these columns by its old name and must be updated in the same commit, or the model will fail to load.

- [x] **Step 6: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` again. Zero errors expected.

- [x] **Step 7: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/dim_selecoes.tmdl \
        fifa-world-cup-2026.SemanticModel/definition/tables/dim_etapas.tmdl \
        fifa-world-cup-2026.SemanticModel/definition/tables/dim_estadios.tmdl \
        fifa-world-cup-2026.SemanticModel/definition/tables/vw_selecao_partida.tmdl
git commit -m "$(cat <<'EOF'
feat: rename the 7 columns bound by the report to PT-BR business names

Scoped down from the spec's "rename everything" per a planning-time
decision: renaming ~150 columns with relationships.tmdl referencing
columns by name, and no te/TOM/Desktop available to validate the
cascade, was the highest-risk operation in the whole plan for no
visible gain on columns no visual touches. These 7 are exactly what
Total de Partidas, the confrontos table, and the two charts bind to.
EOF
)"
```

---

### Task 4: Add the 22 DAX measures

**Files:**
- Create: `fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`
- Modify: `fifa-world-cup-2026.SemanticModel/definition/model.tmdl` (add `ref table _Measures`)
- Script: `/tmp/generate_measures.py` (throwaway, not committed)

**Interfaces:**
- Consumes: `vw_selecao_partida[goals_for/goals_against/xg_for/xg_against/points/result/match_id/team_id]` (raw column names — these were not renamed), `vw_xpts_selecao_partida[expected_points/points_above_expected/actual_points]`, `vw_pontos_recuperados[recovered_points/was_behind]`, `vw_estabilidade_escalacao[lineup_stability_pct]`, `dim_selecoes` (for `REMOVEFILTERS`).
- Produces: 22 measures every later report task binds to by exact name (folder in parentheses):
  - `_Torneio`: `Total de Partidas`, `Seleções`, `Gols Marcados`, `Gols Sofridos`, `Saldo de Gols`, `Média de Gols/Partida`
  - `_Desempenho`: `Pontos Conquistados`, `Vitórias`, `Aproveitamento (%)`, `xG a Favor`, `xG Contra`, `Eficiência Ofensiva (Gols/xG)`
  - `_Avançado`: `Pontos Esperados (xPts)`, `Pontos Acima do Esperado`, `Pontos Reais (xPts)`, `Pontos Recuperados`, `Jogos Atrás no Placar`, `Estabilidade da Escalação (%)`
  - `_Referência`: `Aproveitamento (%) Médio`, `Média Gols/Partida Torneio`, `Eficiência Média Torneio`, `Pontos Médios por Seleção`

- [x] **Step 1: Write the measure-generation script**

Measures live in a new empty calculated table `_Measures` (SQLBI convention: a table that holds only measures, so the field list doesn't force every measure onto whichever fact table it happens to aggregate). Generating all 22 from one Python list avoids hand-typing 22 TMDL blocks and guarantees unique `lineageTag`s.

```python
# /tmp/generate_measures.py
import uuid
from pathlib import Path

ROOT = Path("fifa-world-cup-2026.SemanticModel/definition")

# (name, dax, format_string, folder)
MEASURES = [
    # _Torneio
    ("Total de Partidas", "DISTINCTCOUNT(vw_selecao_partida[match_id])", "#,##0", "_Torneio"),
    ("Seleções", "DISTINCTCOUNT(vw_selecao_partida[team_id])", "#,##0", "_Torneio"),
    ("Gols Marcados", "SUM(vw_selecao_partida[goals_for])", "#,##0", "_Torneio"),
    ("Gols Sofridos", "SUM(vw_selecao_partida[goals_against])", "#,##0", "_Torneio"),
    ("Saldo de Gols", "[Gols Marcados] - [Gols Sofridos]", "#,##0;-#,##0", "_Torneio"),
    ("Média de Gols/Partida", "DIVIDE([Gols Marcados], [Total de Partidas])", "#,##0.00", "_Torneio"),
    # _Desempenho
    ("Pontos Conquistados", "SUM(vw_selecao_partida[points])", "#,##0", "_Desempenho"),
    ("Vitórias", 'CALCULATE([Total de Partidas], vw_selecao_partida[result] = "WIN")', "#,##0", "_Desempenho"),
    ("Aproveitamento (%)", "DIVIDE([Pontos Conquistados], [Total de Partidas] * 3)", "0.0%", "_Desempenho"),
    ("xG a Favor", "SUM(vw_selecao_partida[xg_for])", "#,##0.00", "_Desempenho"),
    ("xG Contra", "SUM(vw_selecao_partida[xg_against])", "#,##0.00", "_Desempenho"),
    ("Eficiência Ofensiva (Gols/xG)", "DIVIDE([Gols Marcados], [xG a Favor])", "0.00", "_Desempenho"),
    # _Avançado
    ("Pontos Esperados (xPts)", "SUM(vw_xpts_selecao_partida[expected_points])", "#,##0.00", "_Avançado"),
    ("Pontos Acima do Esperado", "SUM(vw_xpts_selecao_partida[points_above_expected])", "#,##0.00;-#,##0.00", "_Avançado"),
    ("Pontos Reais (xPts)", "SUM(vw_xpts_selecao_partida[actual_points])", "#,##0", "_Avançado"),
    ("Pontos Recuperados", "SUM(vw_pontos_recuperados[recovered_points])", "#,##0", "_Avançado"),
    ("Jogos Atrás no Placar", "CALCULATE(COUNTROWS(vw_pontos_recuperados), vw_pontos_recuperados[was_behind] = TRUE())", "#,##0", "_Avançado"),
    ("Estabilidade da Escalação (%)", "AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct])", "0.0%", "_Avançado"),
    # _Referência
    ("Aproveitamento (%) Médio", "CALCULATE([Aproveitamento (%)], REMOVEFILTERS(dim_selecoes))", "0.0%", "_Referência"),
    ("Média Gols/Partida Torneio", "CALCULATE([Média de Gols/Partida], REMOVEFILTERS(dim_selecoes))", "#,##0.00", "_Referência"),
    ("Eficiência Média Torneio", "CALCULATE([Eficiência Ofensiva (Gols/xG)], REMOVEFILTERS(dim_selecoes))", "0.00", "_Referência"),
    ("Pontos Médios por Seleção", "CALCULATE(DIVIDE([Pontos Conquistados], [Seleções]), REMOVEFILTERS(dim_selecoes))", "#,##0.0", "_Referência"),
]

def quote(name: str) -> str:
    return f"'{name}'" if any(c in name for c in " %()/áçõêíã") else name

lines = [
    "table _Measures",
    f"\tlineageTag: {uuid.uuid4()}",
    "",
    "\tcolumn _dummy",
    "\t\tdataType: int64",
    "\t\tisHidden",
    f"\t\tlineageTag: {uuid.uuid4()}",
    "\t\tsummarizeBy: none",
    "\t\tsourceColumn: _dummy",
    "",
]

for name, dax, fmt, folder in MEASURES:
    lines.append(f"\tmeasure {quote(name)} = {dax}")
    lines.append(f"\t\tformatString: {fmt}")
    lines.append(f"\t\tdisplayFolder: {folder}")
    lines.append(f"\t\tlineageTag: {uuid.uuid4()}")
    lines.append("")

lines += [
    "\tpartition _Measures = calculated",
    "\t\tmode: import",
    '\t\tsource = ROW("_dummy", 0)',
    "",
]

(ROOT / "tables" / "_Measures.tmdl").write_text("\n".join(lines))
print(f"Wrote _Measures.tmdl with {len(MEASURES)} measures.")
```

- [x] **Step 2: Run it**

```bash
cd "BI - Semana da Informática"
python3 /tmp/generate_measures.py
```

Expected output: `Wrote _Measures.tmdl with 22 measures.`

- [x] **Step 3: Register the new table in `model.tmdl`**

Add one line to the `ref table` list (any position is fine; TMDL doesn't require alphabetical order — match the existing style by adding it near the top):

```tmdl
ref table _Measures
ref table vw_xpts_selecao_partida
```

- [x] **Step 4: Spot-check the generated file**

```bash
head -30 "fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl"
grep -c "^\tmeasure " "fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl"   # expect 22
```

Confirm indentation is tabs (not spaces) and every `measure` line has a matching `formatString`/`displayFolder`/`lineageTag` triplet below it.

- [x] **Step 5: Static validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel`. It checks TMDL syntax and referential integrity (e.g., that `_dummy` the column matches `_dummy` in the `ROW(...)` source) but **cannot evaluate the DAX** — `DIVIDE`, `CALCULATE`, `REMOVEFILTERS` syntax correctness and the actual numeric output are a **user gate**, checked in Task 14.

- [x] **Step 6: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl \
        fifa-world-cup-2026.SemanticModel/definition/model.tmdl
git commit -m "$(cat <<'EOF'
feat: add the 22-measure DAX layer in a dedicated _Measures table

21 measures from the approved spec, plus Pontos Reais (xPts) — needed
by the P3 "xPts × Pontos reais" scatter, which the spec's 21 don't
cover. All team/match measures read from vw_selecao_partida and the 3
analytical views, never from ft_partidas, per the architecture
decision in the design spec (ft_partidas' dimension relationships are
inactive).
EOF
)"
```

---

## Phase B — Theme

### Task 5: Build `IFRS_Feliz.json` from the Tramontina theme

**Files:**
- Create: `fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json`
- Script: `/tmp/build_theme.py` (throwaway, not committed)
- Read (source, unmodified): `fifa-world-cup-2026.Report/StaticResources/RegisteredResources/Tramontina_Layout13807985785810117.json`

**Interfaces:**
- Produces: `IFRS_Feliz.json`, referenced by name in Task 6's `report.json` edit and validated by the color-grep gate in Task 12.

- [ ] **Step 1: Write the theme-transform script**

Every hex color's exact JSON path in the source theme was mapped before writing this plan (`visualStyles.*` wildcard walk). The transform: rename, truncate `dataColors` to the 7 curated greens, remap every blue/navy hex to its green equivalent, leave the `kpi` bad-status red untouched (legitimate semantic color, not a brand color), fix the white-on-white `filterCard` bug, disable `dropShadow`, delete the Tramontina icons.

```python
# /tmp/build_theme.py
import json
from pathlib import Path

SRC = Path("fifa-world-cup-2026.Report/StaticResources/RegisteredResources/Tramontina_Layout13807985785810117.json")
DST = Path("fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json")

theme = json.loads(SRC.read_text())

# 1. Name
theme["name"] = "IFRS Feliz"

# 2. Curated 7-color palette (spec order: primário, escuro, azulado, oliva, ardósia, violeta, ameixa)
theme["dataColors"] = [
    "#339645", "#1A4C23", "#307E6E", "#537D36",
    "#5F6B6D", "#7B43A3", "#914678",
]

# 3. Remove the Tramontina icon set entirely
theme.pop("icons", None)

# 4. Remap every blue/navy hex found in visualStyles to its green equivalent.
#    Semantic red (#D64554, kpi bad-status) is intentionally NOT in this map —
#    it is a legitimate negative-value indicator, not a brand color.
HEX_MAP = {
    "#003087": "#1A4C23",  # navy brand -> verde escuro (banner bg, accent bars, dark text)
    "#004578": "#339645",  # mid blue -> verde primário (slicer selected states)
    "#005A9E": "#339645",  # mid blue -> verde primário (slicer selected states)
    "#1696D2": "#339645",  # bright blue -> verde primário (table/matrix grid outline)
    "#607EB4": "#5F6B6D",  # blue -> cinza ardósia (decorative group-box border, not data)
    "#CFE3F5": "#D7EBDB",  # light blue tint -> light green tint (slicer fillCustom)
    "#E5F1FB": "#EAF5EC",  # very light blue tint -> very light green tint (fills)
}

def remap(obj):
    if isinstance(obj, dict):
        return {k: remap(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [remap(v) for v in obj]
    if isinstance(obj, str):
        return HEX_MAP.get(obj, obj)
    return obj

theme["visualStyles"] = remap(theme["visualStyles"])

# 5. Fix the white-on-white filterCard "Available" bug
vs = theme["visualStyles"]["*"]["*"]
for card in vs["filterCard"]:
    if card.get("$id") == "Available":
        card["fontColor"]["solid"]["color"] = "#333333"

# 6. Disable drop shadows (accessibility: vestibular issues; also fights grid alignment)
for shadow in vs["dropShadow"]:
    shadow["show"] = False

DST.write_text(json.dumps(theme, ensure_ascii=False, indent=2))
print(f"Wrote {DST} ({DST.stat().st_size:,} bytes, source was {SRC.stat().st_size:,} bytes)")
```

- [ ] **Step 2: Run it**

```bash
cd "BI - Semana da Informática"
python3 /tmp/build_theme.py
```

Expected: a size reduction of well over 100 KB (the source is 161 KB, dominated by base64 icon data that Step 1 deletes).

- [ ] **Step 3: Verify the transform**

```bash
cd "fifa-world-cup-2026.Report/StaticResources/RegisteredResources"
python3 -c "
import json, re
t = json.load(open('IFRS_Feliz.json'))
assert t['name'] == 'IFRS Feliz'
assert t['dataColors'] == ['#339645','#1A4C23','#307E6E','#537D36','#5F6B6D','#7B43A3','#914678']
assert 'icons' not in t
s = json.dumps(t)
assert '#003087' not in s and '#607EB4' not in s and '#1696D2' not in s
assert '#D64554' in s, 'the legitimate bad-status red should still be present'
available = [c for c in t['visualStyles']['*']['*']['filterCard'] if c.get('\$id') == 'Available'][0]
assert available['fontColor']['solid']['color'] != available['backgroundColor']['solid']['color']
assert all(sh['show'] is False for sh in t['visualStyles']['*']['*']['dropShadow'])
print('All checks passed.')
"
cd ../../../..
```

- [ ] **Step 4: Color grep gate (agent gate, early check)**

```bash
grep -oE '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json" | sort -u
```

Every hex printed here should be one you recognize from the palette or an intentionally-preserved neutral/semantic color (`#000000`, `#333333`, `#666666`, `#6A6A6A`, `#C8C6C4`, `#D0D0D0`, `#E6E6E6`, `#E7E7E7`, `#EEEEEE`, `#F3F3F3`, `#F4F4F4`, `#FFFFFF`, `#D64554`). If an unfamiliar hex appears, the remap missed a path — go back to Step 1.

- [ ] **Step 5: Commit**

```bash
cd "BI - Semana da Informática"
git add "fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json"
git commit -m "$(cat <<'EOF'
feat: build IFRS_Feliz theme from the Tramontina base

Curates dataColors from 480 auto-generated entries down to the 7
verified-contrast greens from the design spec, remaps every fixed
blue/navy hex inside visualStyles to its green equivalent, drops the
Tramontina name and 8 branded icons, fixes a white-text-on-white-
background bug in the "Available" filter card, and disables drop
shadows. Old theme file is untouched — swapped out of report.json in
the next commit so this one stays revertible on its own.
EOF
)"
```

---

### Task 6: Wire the new theme into the report, remove the old one

**Files:**
- Modify: `fifa-world-cup-2026.Report/definition/report.json`
- Delete: `fifa-world-cup-2026.Report/StaticResources/RegisteredResources/Tramontina_Layout13807985785810117.json`

**Interfaces:**
- Consumes: `IFRS_Feliz.json` from Task 5.
- Produces: a report whose only custom theme is IFRS Feliz — required before Task 12's color-grep gate can mean anything (a visual could otherwise still cite the old theme's colors by name).

- [ ] **Step 1: Replace the `customTheme` and `resourcePackages` entries**

In `report.json`, change:

```json
"customTheme": {
  "name": "Tramontina_Layout13807985785810117.json",
  "reportVersionAtImport": { "visual": "2.11.0", "report": "3.4.0", "page": "2.3.1" },
  "type": "RegisteredResources"
}
```
to:
```json
"customTheme": {
  "name": "IFRS_Feliz.json",
  "reportVersionAtImport": { "visual": "2.11.0", "report": "3.4.0", "page": "2.3.1" },
  "type": "RegisteredResources"
}
```

and in `resourcePackages`, change the `RegisteredResources` item:

```json
{
  "name": "Tramontina_Layout13807985785810117.json",
  "path": "Tramontina_Layout13807985785810117.json",
  "type": "CustomTheme"
}
```
to:
```json
{
  "name": "IFRS_Feliz.json",
  "path": "IFRS_Feliz.json",
  "type": "CustomTheme"
}
```

Use `pbir get "fifa-world-cup-2026.Report"` first to confirm the live property path (installed in Task 7 — if Task 7 hasn't run yet, do this edit directly in `report.json` since `report.json`-level theme wiring is not covered by a documented `pbir theme` subcommand for *selecting which registered theme is active*; `pbir theme` commands operate on an already-active theme's colors/formatting, not on swapping which file is registered as active. Confirm this with `pbir theme --help` once installed — if a direct command exists, prefer it and update this step).

- [ ] **Step 2: Delete the old theme file**

```bash
cd "BI - Semana da Informática"
rm "fifa-world-cup-2026.Report/StaticResources/RegisteredResources/Tramontina_Layout13807985785810117.json"
```

- [ ] **Step 3: Verify no reference to the old file remains**

```bash
grep -rl "Tramontina" "fifa-world-cup-2026.Report/" 2>/dev/null
```

Expected: no output.

- [ ] **Step 4: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: switch the report to the IFRS Feliz theme

report.json now registers IFRS_Feliz.json as the custom theme instead
of the Tramontina one, which is deleted. This is the one-file swap
the spec's whole theme design exists to make possible for a future
color change.
EOF
)"
```

---

## Phase C — Report pages

### Task 7: Install `pbir-cli` and confirm it reads this report

**Files:** none (tooling setup)

**Interfaces:**
- Produces: a working `pbir` command on PATH, used by every remaining task.

- [ ] **Step 1: Install `uv`**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source "$HOME/.local/bin/env" 2>/dev/null || export PATH="$HOME/.local/bin:$PATH"
uv --version
```

- [ ] **Step 2: Install `pbir-cli`**

```bash
uv tool install pbir-cli
pbir --version
```

If this fails because of no network access or a locked-down environment, fall back to the portable build per the `pbir-cli` skill (`bin/fetch.sh`) — do not hand-edit PBIR JSON as a substitute; stop and report the gap instead.

- [ ] **Step 3: Confirm it finds and validates the report**

```bash
cd "BI - Semana da Informática"
pbir ls
pbir ls "fifa-world-cup-2026.Report"
pbir model "fifa-world-cup-2026.Report" -d -t _Measures
```

Expected: the last command lists the 22 measures from Task 4 with their `displayFolder` values, confirming `pbir` reads the on-disk TMDL model directly (no Desktop/AS instance needed for this).

- [ ] **Step 4: Run baseline validation**

```bash
pbir validate "fifa-world-cup-2026.Report"
```

Note the starting state (one blank page, no visuals) so later validation runs can be compared against it.

No commit — tooling only, nothing in the repo changes.

---

### Task 8: Report-level filter, universal slicers, and the Page 1 shell

**Files:**
- Modify: `fifa-world-cup-2026.Report/definition/report.json` (report-level filter)
- Modify: `fifa-world-cup-2026.Report/definition/pages/f976a7b441542f698fbc/page.json` (rename to "Visão Geral do Torneio", resize confirmation)
- Create: slicer visuals under `fifa-world-cup-2026.Report/definition/pages/f976a7b441542f698fbc/visuals/`

**Interfaces:**
- Consumes: `_Measures` table (Task 4), `dim_selecoes[Seleção]` and `dim_etapas[Fase]` (Task 3), `ft_partidas[status]` (unchanged name, table hidden but still filterable — Task 2).
- Produces: a page named `Visão Geral do Torneio` with a header band (title + 2 slicers) at `y=24, h=48`, and a `syncGroup` on both slicers that Tasks 9–10 reuse on pages 2 and 3.

- [ ] **Step 1: Discover the report-level filter command**

```bash
cd "BI - Semana da Informática"
pbir add filter --help
```

- [ ] **Step 2: Add the status filter at report level**

Add a basic filter, `ft_partidas[status] = "Completed"`, scoped to the report (applies to all 3 pages; harmless no-op on page 3, whose visuals don't touch `ft_partidas` or `vw_selecao_partida`). Use the exact flag names `pbir add filter --help` printed in Step 1 — do not guess syntax. The filter should be locked (not user-editable) and hidden from the filter pane, since it's a data-correctness fix, not an analytical choice for the viewer:

```bash
pbir add filter "fifa-world-cup-2026.Report" --field "ft_partidas[status]" --type basic --values "Completed" --scope report
```

If `--scope report` isn't the actual flag name, use whatever `--help` showed for "apply to whole report" instead. Then hide and lock it (again, check exact flags before running):

```bash
pbir filters pane-hide "fifa-world-cup-2026.Report/filter:<FilterName>"
pbir filters lock "fifa-world-cup-2026.Report/filter:<FilterName>"
```

- [ ] **Step 3: Rename Page 1 and confirm its size**

```bash
pbir pages rename "fifa-world-cup-2026.Report/f976a7b441542f698fbc.Page" "Visão Geral do Torneio"
pbir pages json "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page"
```

Confirm `width: 1280, height: 720` in the output — every position in this plan assumes that canvas.

- [ ] **Step 4: Add the page title**

```bash
pbir add title "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" "Visão Geral do Torneio" --width 500 --x 24 --y 24 --height 48
```

- [ ] **Step 5: Discover slicer visual properties, then add the two universal slicers**

```bash
pbir schema describe slicer
```

Add the Seleção slicer (dropdown-style, positioned to the right of the title inside the same 48px band) and the Fase slicer next to it:

```bash
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" slicer --name slicer_selecao --x 660 --y 24 --width 300 --height 48
pbir visuals bind "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page/slicer_selecao.Visual" --field "dim_selecoes[Seleção]"

pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" slicer --name slicer_fase --x 976 --y 24 --width 280 --height 48
pbir visuals bind "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page/slicer_fase.Visual" --field "dim_etapas[Fase]"
```

(660 + 300 + 16 = 976; 976 + 280 = 1256 = 1280 - 24, matching the page's right margin — same grid arithmetic as the rest of the layout.)

- [ ] **Step 6: Create the sync group so pages 2 and 3 can reuse these two slicers**

```bash
pbir schema describe slicer syncGroup
```

Apply a `syncGroup` (e.g. `sgSelecao` / `sgFase`) to both slicers with sync-values and sync-filter enabled, following the exact property shape `schema describe` returned. Record the group names used here — Tasks 9 and 10 apply the *same* `syncGroup` names to their own copies of these two slicers.

- [ ] **Step 7: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report" --fields
```

Expected: no errors. `--fields` specifically confirms `dim_selecoes[Seleção]` and `dim_etapas[Fase]` resolve against the model — i.e., that Task 3's renames and this task's bindings agree.

- [ ] **Step 8: Commit**

```bash
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: report-level status filter and Page 1 header band

Filters out the 4 unplayed matches (2 semifinals with a phantom LOSS,
final, third-place) at report level via ft_partidas[status], which
propagates to vw_selecao_partida through their active relationship.
Renames Page 1 to "Visão Geral do Torneio" and adds the header-band
signature (title + Seleção/Fase slicers) that pages 2 and 3 repeat
identically.
EOF
)"
```

---

### Task 9: Page 1 visuals — KPIs, Gols por Fase, mapa, tabela

**Files:**
- Create: visuals under `fifa-world-cup-2026.Report/definition/pages/Visão Geral do Torneio/visuals/`

**Interfaces:**
- Consumes: `_Measures` (`Total de Partidas`, `Gols Marcados`, `Média de Gols/Partida`, `Eficiência Ofensiva (Gols/xG)`, `Média Gols/Partida Torneio`, `Eficiência Média Torneio`, `Pontos Conquistados`, `Saldo de Gols`, `Aproveitamento (%)`), `dim_etapas[Fase]`, `dim_estadios[Cidade]`/`['País-Sede']`, `dim_selecoes[Seleção]`.
- Produces: the completed Page 1 canvas, checked by Task 12's full validation pass.

Grid for this page (from the spec, all pages share it):
```
y=88   KPI row   — 4 × 296w × 100h   x = 24 / 336 / 648 / 960
y=204  analytic row — 2 × 608w × 238h   x = 24 / 648
y=458  detail row — 1 × 1232w × 238h   x = 24
```

- [ ] **Step 1: KPI 1 and 2 — plain cards (no reference measure exists for these two)**

```bash
pbir schema describe card
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" card --name kpi_total_partidas --x 24 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_total_partidas.Visual" --field "_Measures[Total de Partidas]"

pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" card --name kpi_gols_marcados --x 336 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_gols_marcados.Visual" --field "_Measures[Gols Marcados]"
```

- [ ] **Step 2: KPI 3 and 4 — `kpi` visuals with a reference-measure target**

```bash
pbir schema describe kpi
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" kpi --name kpi_media_gols --x 648 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_media_gols.Visual" --field "_Measures[Média de Gols/Partida]" --role Value
pbir visuals bind ".../kpi_media_gols.Visual" --field "_Measures[Média Gols/Partida Torneio]" --role TargetValue

pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" kpi --name kpi_eficiencia --x 960 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_eficiencia.Visual" --field "_Measures[Eficiência Ofensiva (Gols/xG)]" --role Value
pbir visuals bind ".../kpi_eficiencia.Visual" --field "_Measures[Eficiência Média Torneio]" --role TargetValue
```

(Confirm the exact role names — `Value`/`TargetValue` or whatever `pbir schema describe kpi` printed in Step 1's first command — before running the `bind` calls.)

- [ ] **Step 3: Gols por Fase — column + line combo, sorted by `Fase`'s `sortByColumn`**

```bash
pbir schema describe lineClusteredColumnComboChart
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" lineClusteredColumnComboChart --name chart_gols_fase --x 24 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_gols_fase.Visual" --field "dim_etapas[Fase]" --role Category
pbir visuals bind ".../chart_gols_fase.Visual" --field "_Measures[Gols Marcados]" --role ColumnValue
pbir visuals bind ".../chart_gols_fase.Visual" --field "_Measures[Média de Gols/Partida]" --role LineValue
```

Sort must follow `Fase`'s `sortByColumn` (`stage_id`) automatically since that's a model-level property set in Task 3 — do not add a visual-level sort override unless the rendered chart (Task 14, user gate) shows it sorted alphabetically instead.

- [ ] **Step 4: Mapa das sedes — bubble map**

```bash
pbir schema describe map
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" map --name map_sedes --x 648 --y 204 --width 608 --height 238
pbir visuals bind ".../map_sedes.Visual" --field "dim_estadios[Cidade]" --role Location
pbir visuals bind ".../map_sedes.Visual" --field "dim_estadios['País-Sede']" --role Location
pbir visuals bind ".../map_sedes.Visual" --field "_Measures[Total de Partidas]" --role Size
```

- [ ] **Step 5: Tabela Top 10 seleções**

```bash
pbir schema describe tableEx
pbir add visual "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" tableEx --name table_top10 --x 24 --y 458 --width 1232 --height 238
pbir visuals bind ".../table_top10.Visual" --field "dim_selecoes[Seleção]"
pbir visuals bind ".../table_top10.Visual" --field "_Measures[Pontos Conquistados]"
pbir visuals bind ".../table_top10.Visual" --field "_Measures[Saldo de Gols]"
pbir visuals bind ".../table_top10.Visual" --field "_Measures[Gols Marcados]"
pbir visuals bind ".../table_top10.Visual" --field "_Measures[Aproveitamento (%)]"
```

Sort descending by `Pontos Conquistados`, top 10 rows, and add a data bar on the `Pontos Conquistados` column — check `pbir visuals cf --help` and `pbir pages json` sort syntax before applying; both are documented `pbir` capabilities per the skill (`references/sort-visuals.md`, `pbir visuals cf`).

- [ ] **Step 6: Alignment and spacing check (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Visão Geral do Torneio.Page" --qa
```

`--qa` checks overlap/overflow. Fix any reported issue before continuing.

- [ ] **Step 7: Full validation (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report" --all
```

- [ ] **Step 8: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: build Page 1 (Visão Geral do Torneio) visuals

4 KPIs (2 plain cards with no natural reference, 2 kpi visuals against
the _Referência measures), Gols por Fase combo chart sorted in
tournament order via Fase's sortByColumn, a bubble map by
Cidade/País-Sede sized by match count, and a Top 10 seleções table.
EOF
)"
```

---

### Task 10: Page 2 — Perfil da Seleção

**Files:**
- Create: `fifa-world-cup-2026.Report/definition/pages/Perfil da Seleção/` (new page, plus its `visuals/`)

**Interfaces:**
- Consumes: `_Measures` (`Pontos Conquistados`, `Aproveitamento (%)`, `Aproveitamento (%) Médio`, `Saldo de Gols`, `Eficiência Ofensiva (Gols/xG)`, `Eficiência Média Torneio`, `xG a Favor`, `xG Contra`), `vw_selecao_partida[Adversário]`, `[Resultado]`, `['Data da Partida']`, `dim_etapas[Fase]`, the two `syncGroup`s from Task 8.
- Produces: Page 2 canvas.

- [ ] **Step 1: Create the page and header band**

```bash
cd "BI - Semana da Informática"
pbir add page "fifa-world-cup-2026.Report" --name "Perfil da Seleção" --width 1280 --height 720
pbir add title "fifa-world-cup-2026.Report/Perfil da Seleção.Page" "Perfil da Seleção" --width 500 --x 24 --y 24 --height 48
```

- [ ] **Step 2: Add the same two slicers, joined to the Task 8 sync groups**

```bash
pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" slicer --name slicer_selecao --x 660 --y 24 --width 300 --height 48
pbir visuals bind ".../slicer_selecao.Visual" --field "dim_selecoes[Seleção]"
# apply the same syncGroup name used in Task 8 Step 6 for the Seleção slicer

pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" slicer --name slicer_fase --x 976 --y 24 --width 280 --height 48
pbir visuals bind ".../slicer_fase.Visual" --field "dim_etapas[Fase]"
# apply the same syncGroup name used in Task 8 Step 6 for the Fase slicer
```

- [ ] **Step 3: 4 KPIs**

```bash
pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" kpi --name kpi_pontos --x 24 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_pontos.Visual" --field "_Measures[Pontos Conquistados]" --role Value
pbir visuals bind ".../kpi_pontos.Visual" --field "_Measures[Pontos Médios por Seleção]" --role TargetValue

pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" kpi --name kpi_aproveitamento --x 336 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_aproveitamento.Visual" --field "_Measures[Aproveitamento (%)]" --role Value
pbir visuals bind ".../kpi_aproveitamento.Visual" --field "_Measures[Aproveitamento (%) Médio]" --role TargetValue

pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" card --name kpi_saldo --x 648 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_saldo.Visual" --field "_Measures[Saldo de Gols]"

pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" kpi --name kpi_eficiencia --x 960 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_eficiencia.Visual" --field "_Measures[Eficiência Ofensiva (Gols/xG)]" --role Value
pbir visuals bind ".../kpi_eficiencia.Visual" --field "_Measures[Eficiência Média Torneio]" --role TargetValue
```

`Saldo de Gols` is a plain `card`: `Saldo de Gols` can be negative, and the "Médio" reference measures were designed for ratios/percentages, not for a signed differential — no natural target exists for it, matching the spec's page-1 pattern (2 of 4 KPIs are plain cards).

- [ ] **Step 4: Gols × xG por partida — combo chart**

```bash
pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" lineClusteredColumnComboChart --name chart_gols_xg --x 24 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_gols_xg.Visual" --field "vw_selecao_partida['Data da Partida']" --role Category
pbir visuals bind ".../chart_gols_xg.Visual" --field "_Measures[Gols Marcados]" --role ColumnValue
pbir visuals bind ".../chart_gols_xg.Visual" --field "_Measures[xG a Favor]" --role LineValue
```

- [ ] **Step 5: Dispersão xG a favor × xG contra**

```bash
pbir schema describe scatterChart
pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" scatterChart --name chart_dispersao_xg --x 648 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_dispersao_xg.Visual" --field "_Measures[xG a Favor]" --role X
pbir visuals bind ".../chart_dispersao_xg.Visual" --field "_Measures[xG Contra]" --role Y
pbir visuals bind ".../chart_dispersao_xg.Visual" --field "vw_selecao_partida[match_id]" --role Details
```

`match_id` here only forces one point per match (a technical grain key in the "Details" role, never shown to the viewer as a label) — it was intentionally left out of Task 3's rename scope for exactly this reason.

- [ ] **Step 6: Tabela de confrontos**

```bash
pbir add visual "fifa-world-cup-2026.Report/Perfil da Seleção.Page" tableEx --name table_confrontos --x 24 --y 458 --width 1232 --height 238
pbir visuals bind ".../table_confrontos.Visual" --field "vw_selecao_partida[Adversário]"
pbir visuals bind ".../table_confrontos.Visual" --field "dim_etapas[Fase]"
pbir visuals bind ".../table_confrontos.Visual" --field "_Measures[Gols Marcados]"
pbir visuals bind ".../table_confrontos.Visual" --field "_Measures[Gols Sofridos]"
pbir visuals bind ".../table_confrontos.Visual" --field "_Measures[xG a Favor]"
pbir visuals bind ".../table_confrontos.Visual" --field "vw_selecao_partida[Resultado]"
```

Numeric columns bind to the `_Measures` versions (not raw `goals_for`/`goals_against`/`xg_for` columns) — at this table's per-match grain they're numerically identical, and reusing the measures avoids renaming 3 more columns for a one-visual gain.

- [ ] **Step 7: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Perfil da Seleção.Page" --qa
pbir validate "fifa-world-cup-2026.Report" --all
```

- [ ] **Step 8: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: build Page 2 (Perfil da Seleção)

4 KPIs (Saldo de Gols as a plain card — no meaningful average
reference for a signed differential), Gols × xG combo chart by match
date, xG a favor × xG contra scatter, and a confrontos table reusing
_Measures for its numeric columns instead of renaming 3 more raw
columns. Seleção/Fase slicers joined to Page 1's sync groups.
EOF
)"
```

---

### Task 11: Page 3 — Análises Avançadas

**Files:**
- Create: `fifa-world-cup-2026.Report/definition/pages/Análises Avançadas/` (new page, plus its `visuals/`)

**Interfaces:**
- Consumes: `_Measures` (`Pontos Esperados (xPts)`, `Pontos Acima do Esperado`, `Pontos Reais (xPts)`, `Pontos Recuperados`, `Estabilidade da Escalação (%)`), `dim_selecoes[Seleção]`.
- Produces: Page 3 canvas, including the mandatory pênaltis footnote.

- [ ] **Step 1: Create the page and header band**

```bash
cd "BI - Semana da Informática"
pbir add page "fifa-world-cup-2026.Report" --name "Análises Avançadas" --width 1280 --height 720
pbir add title "fifa-world-cup-2026.Report/Análises Avançadas.Page" "Mérito ou Sorte?" --width 500 --x 24 --y 24 --height 48
```

Per the spec, this page reaches `dim_selecoes` and `dim_etapas` but not `dim_estadios`/`dim_arbitros` — add the same 2 universal slicers, same sync groups, no others:

```bash
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" slicer --name slicer_selecao --x 660 --y 24 --width 300 --height 48
pbir visuals bind ".../slicer_selecao.Visual" --field "dim_selecoes[Seleção]"
# same syncGroup as Task 8/10

pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" slicer --name slicer_fase --x 976 --y 24 --width 280 --height 48
pbir visuals bind ".../slicer_fase.Visual" --field "dim_etapas[Fase]"
# same syncGroup as Task 8/10
```

- [ ] **Step 2: 4 KPIs — all plain cards (no `_Referência` measure exists for any of the four)**

```bash
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" card --name kpi_xpts --x 24 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_xpts.Visual" --field "_Measures[Pontos Esperados (xPts)]"

pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" card --name kpi_pontos_acima --x 336 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_pontos_acima.Visual" --field "_Measures[Pontos Acima do Esperado]"

pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" card --name kpi_recuperados --x 648 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_recuperados.Visual" --field "_Measures[Pontos Recuperados]"

pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" card --name kpi_estabilidade --x 960 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_estabilidade.Visual" --field "_Measures[Estabilidade da Escalação (%)]"
```

- [ ] **Step 3: Visual-âncora — barras divergentes de Pontos Acima do Esperado**

```bash
pbir schema describe barChart
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" barChart --name chart_pontos_acima --x 24 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_pontos_acima.Visual" --field "dim_selecoes[Seleção]" --role Category
pbir visuals bind ".../chart_pontos_acima.Visual" --field "_Measures[Pontos Acima do Esperado]" --role Y
```

Color this visual with conditional formatting on the measure sign: verde primário (`ThemeDataColor` index 0) for positive, violeta (index 5, `#7B43A3`) for negative. Check `pbir visuals cf --help` for the exact rule-based CF syntax before applying — this is the one visual in the whole report where color is allowed to encode meaning (above/below expected), per the spec's explicit exception.

- [ ] **Step 4: Dispersão secundária — xPts × Pontos reais**

```bash
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" scatterChart --name chart_xpts_reais --x 648 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_xpts_reais.Visual" --field "_Measures[Pontos Esperados (xPts)]" --role X
pbir visuals bind ".../chart_xpts_reais.Visual" --field "_Measures[Pontos Reais (xPts)]" --role Y
pbir visuals bind ".../chart_xpts_reais.Visual" --field "dim_selecoes[Seleção]" --role Details
```

This is the visual that needed the `Pontos Reais (xPts)` measure added in Task 4.

- [ ] **Step 5: Tabela de estabilidade de escalação, with footnote**

```bash
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" tableEx --name table_estabilidade --x 24 --y 458 --width 1232 --height 200
pbir visuals bind ".../table_estabilidade.Visual" --field "dim_selecoes[Seleção]"
pbir visuals bind ".../table_estabilidade.Visual" --field "_Measures[Estabilidade da Escalação (%)]"
```

Add the mandatory footnote below the table (spec Achado 1 requires this — the page's KPIs and xPts comparisons treat a penalty-shootout advance as a 1-point draw, not a win):

```bash
pbir schema describe textbox
pbir add visual "fifa-world-cup-2026.Report/Análises Avançadas.Page" textbox --name footnote_penaltis --x 24 --y 666 --width 1232 --height 30
```

Set its text (via `pbir set` or the textbox-specific command `schema describe textbox` surfaces) to:

> Partidas decididas nos pênaltis contam como empate (1 ponto) neste painel — o placar antes da disputa é o que define Pontos, Resultado e a comparação com xPts.

- [ ] **Step 6: Validate (agent gate)**

```bash
pbir validate "fifa-world-cup-2026.Report/Análises Avançadas.Page" --qa
pbir validate "fifa-world-cup-2026.Report" --all
```

- [ ] **Step 7: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: build Page 3 (Análises Avançadas)

4 plain-card KPIs, the divergent-bar Pontos Acima do Esperado anchor
(verde/violeta conditional formatting — the one visual in the report
where color is allowed to carry meaning), the xPts × Pontos reais
scatter that Task 4's extra measure exists for, an estabilidade table,
and the mandatory pênaltis-convention footnote from the design spec.
EOF
)"
```

---

### Task 12: Full validation pass and push

**Files:** none (validation + git operations only)

**Interfaces:**
- Consumes: everything from Tasks 1–11.
- Produces: a pushed `main` branch on `fifa-world-cup-2026-bi`, or a documented list of failures if something doesn't pass.

- [ ] **Step 1: Model validation (agent gate)**

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` one more time on the final state. Zero errors expected.

- [ ] **Step 2: Report validation (agent gate)**

```bash
cd "BI - Semana da Informática"
pbir validate "fifa-world-cup-2026.Report" --all
```

Zero errors expected. `--all` includes the `--fields` check (every binding resolves against the model) and the `--qa` check (no overlapping/overflowing visuals) across all 3 pages at once.

- [ ] **Step 3: Color gate (agent gate)**

```bash
grep -rE '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/definition/pages/" 2>/dev/null
```

Expected: **no output**. Any hit means a visual has a hardcoded color instead of a `ThemeDataColor` reference — find it with `pbir get` on the visual named in the grep output and fix it before continuing.

- [ ] **Step 4: Design gate checklist**

Walk the `pbi-report-design` skill's closing checklist against the 3 built pages: identity propagated (header band on all 3, same position), one intent per page (`summary` / `exploration` / `narrative` as specced), equal spacing (all positions in this plan derive from margin=24/gap=16 — confirm no visual was placed off that grid), KPIs backed by a target where one exists. Note any deviation found; fix inline if small, otherwise list it for the user in Task 14.

- [ ] **Step 5: Push**

```bash
git push
git log --oneline -15
```

- [ ] **Step 6: If any gate in Steps 1–4 failed**

Do not push. Fix the specific failure, re-run only the gate that failed (not the whole sequence), then retry Step 5.

---

### Task 13: (informational — no checkbox) What this plan does not verify

Everything through Task 12 is checkable without Power BI Desktop. Two things genuinely cannot be:

1. **Do the 22 DAX measures evaluate to the right numbers?** `pbip-validator` checks TMDL syntax, not DAX semantics. `DIVIDE`, `CALCULATE`, `REMOVEFILTERS` could all be syntactically valid TMDL and still be logically wrong, and nothing in Tasks 1–12 would catch it.
2. **Does the report actually render — are visuals legible, correctly positioned, correctly colored?** `pbir validate --qa` catches overlap and overflow from the JSON's numbers; it cannot render a screenshot. `pbir desktop screenshot` (the tool that could) is Windows-only and this plan was written and is meant to be executed in a Linux/WSL environment with no Desktop available.

Task 14 is the checklist for the user to run these two checks themselves.

---

### Task 14: User acceptance checklist (manual — not agent-executable)

Hand this checklist to the user once Task 12 has pushed successfully. Every item here requires opening the file in Power BI Desktop on Windows.

- [ ] Open `fifa-world-cup-2026.pbip` in Power BI Desktop. It should load with no error dialog. (If it errors, the most likely cause is a TMDL mistake `pbip-validator` didn't catch — report the exact error text back.)
- [ ] Field list check: `ft_partidas` should not appear. The 14 `LocalDateTable_*`/`DateTableTemplate_*` tables should not appear. `_Measures` should appear with 22 measures across 4 folders (`_Torneio`, `_Desempenho`, `_Avançado`, `_Referência`).
- [ ] On Page 1, confirm the 4 acceptance numbers from the design spec, unfiltered:

  | Card | Expected |
  |---|---|
  | Total de Partidas | 100 |
  | Gols Marcados | 292 |
  | Média de Gols/Partida | 2,92 |
  | Eficiência Ofensiva (Gols/xG) | 1,10 |

  If any number is off, the DAX (not the report layout) is the likely culprit — check the measure's formula in `_Measures.tmdl` against the spec's acceptance table.
- [ ] Select a seleção in the Seleção slicer on Page 1. Confirm every visual on **all 3 pages** reacts — including Page 3, which reaches `dim_selecoes` through the 3 analytical views, not through `vw_selecao_partida`. A visual that doesn't react is reading a table the slicer can't reach and is a bug.
- [ ] Repeat with the Fase slicer.
- [ ] Read the "Gols por Fase" chart's X axis on Page 1: it should read Group Stage → Round of 32 → Round of 16 → Quarter-finals → Semi-finals → Third-place match → Final, in that order (not alphabetical). If it's alphabetical, `Fase`'s `sortByColumn: stage_id` (Task 3) didn't take effect — check for a competing visual-level sort override.
- [ ] Screenshot all 3 pages (Desktop's own export, or `pbir desktop screenshot` if running `pbir` from the Windows side) and do a visual pass: legible KPI numbers, no visual overlapping another, header band identical across all 3 pages, footnote visible on Page 3.
- [ ] Confirm the map on Page 1 renders bubbles for the venue cities — this needs live internet access for Bing geocoding at render time (spec's documented risk).
- [ ] Close and reopen the file once. This confirms `__PBI_TimeIntelligenceEnabled = 0` (Task 1) actually stops Desktop from regenerating the date tables on reload — the single best test that the flag setting worked.
