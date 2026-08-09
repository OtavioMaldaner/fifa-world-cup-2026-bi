# Página "Jogadores" Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a 4th report page, "Jogadores", to `fifa-world-cup-2026.Report`, exposing the already-imported-but-unused `ft_estatisticas_jogador` table through a new DAX measure layer and 7 visuals — per `docs/superpowers/specs/2026-08-08-pagina-jogadores-design.md`.

**Architecture:** Direct TMDL authoring for the semantic model (one new relationship, 15 new measures appended to the existing `_Measures` table), driven by a throwaway Python script for the mechanical parts (measure block generation), same pattern as the original 3-page build. The report page is built exclusively through the `pbir` CLI (already installed at `/home/otaviomaldaner/.local/bin/pbir`, v0.9.28) — never hand-edited JSON. Every task ends in a state `pbip-validator` (model) or `pbir validate` (report) can check statically.

**Tech Stack:** Power BI Project (PBIP/TMDL/PBIR), Python 3 (stdlib only), `pbir-cli`, git.

## Global Constraints

- All work happens in the `fifa-world-cup-2026-bi` repository. Every commit uses `git -C "BI - Semana da Informática"` — never the Databricks repo.
- **No Tabular Editor CLI (`te`), TOM, or Power BI Desktop is available in this environment** (Linux/WSL2, confirmed empirically for the original 3-page build and still true here). Model changes go through direct TMDL editing, validated statically with the `pbip-validator` agent. Report changes go through the `pbir` CLI, which reads/writes the on-disk PBIR/TMDL files without needing Desktop.
- **Agent gate vs. user gate**, same distinction as the original plan: DAX values can only be evaluated by a running Analysis Services instance, which doesn't exist here — every DAX-correctness check in this plan is a **user gate**, collected in the final checklist (Task 9). Structural checks (TMDL syntax, field bindings resolving, no overlapping visuals) are **agent gates**, run inline.
- **Working tree has pre-existing uncommitted formatting drift** (CRLF/LF and trailing-newline differences across most report/model files, from an external editor touching the PBIP) that predates this plan and is unrelated to it. Do not `git checkout` or otherwise discard it. Every `git add` in this plan stages **only the files this plan's tasks intentionally change** — never a broad `git add -A` — so that pre-existing drift stays out of this plan's commits.
- Position codes in the data are `GK`, `DEF`, `MID`, `FWD` (confirmed by direct inspection of `player_stats.csv` and `squads_and_players.csv` during brainstorming — not `'Goalkeeper'` or similar).
- Page canvas: 1280×720, margin 24, gap 16 — same grid as the other 3 pages.
- Reuses the existing `IFRS_Feliz.json` theme unchanged. No new colors, no `Literal` hex in any `visual.json` — same propagation rule as the rest of the report, checked by the same color-grep gate.

---

## Phase A — Semantic model

### Task 1: Add the `ft_estatisticas_jogador.player_id → dim_jogadores.player_id` relationship

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/relationships.tmdl`

**Interfaces:**
- Produces: an active many-to-one relationship that every later report task (Tasks 4–6) depends on to resolve `dim_jogadores[player_name]`/`[position]` against `ft_estatisticas_jogador` rows. Without this, every ranking visual in this plan would show `player_id`, not a name.

- [x] **Step 1: Confirm the relationship doesn't already exist**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -B1 -A1 "ft_estatisticas_jogador" relationships.tmdl
cd ../../..
```

Expected: only one block, `ft_estatisticas_jogador.team_id → dim_selecoes.team_id`. If a `player_id` relationship already appears, stop — this task is already done, skip to Task 2.

- [x] **Step 2: Insert the new relationship block**

Find the existing `ft_estatisticas_jogador.team_id` block in `relationships.tmdl`:

```tmdl
relationship 99176784-8f48-4c05-8f78-cfafbbc4f5bd
	fromColumn: ft_estatisticas_jogador.team_id
	toColumn: dim_selecoes.team_id

	annotation PBI_IsFromSource = FS
```

Insert a new block immediately before it (same file, same blank-line-separated block style as every other relationship):

```tmdl
relationship f584ef8f-ad15-4efe-a83a-c9fa2ee2116d
	fromColumn: ft_estatisticas_jogador.player_id
	toColumn: dim_jogadores.player_id

	annotation PBI_IsFromSource = FS

relationship 99176784-8f48-4c05-8f78-cfafbbc4f5bd
	fromColumn: ft_estatisticas_jogador.team_id
	toColumn: dim_selecoes.team_id

	annotation PBI_IsFromSource = FS
```

- [x] **Step 3: Verify**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^relationship " relationships.tmdl                     # one more than before this task
grep -A1 "ft_estatisticas_jogador.player_id" relationships.tmdl # expect "toColumn: dim_jogadores.player_id"
cd ../../..
```

- [x] **Step 4: Static validation (agent gate)**

Dispatch the `pbip-validator` agent against `fifa-world-cup-2026.SemanticModel`. Zero errors expected — in particular, no dangling relationship (both `player_id` columns must exist with matching data types: `int64` on both sides, confirmed in `ft_estatisticas_jogador.tmdl` and `dim_jogadores.tmdl`).

- [x] **Step 5: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/relationships.tmdl
git commit -m "$(cat <<'EOF'
feat: relate ft_estatisticas_jogador to dim_jogadores by player_id

The model had ft_estatisticas_jogador.team_id -> dim_selecoes but never
player_id -> dim_jogadores, so no visual could resolve a player_id to a
name. The Genie agent already defines this exact join and calls it
required for "every individual ranking" (metadata/copa_mundo_2026_metadata.py)
— this brings the Power BI model in line with a join Genie already relies on.
EOF
)"
```

---

### Task 2: Add the 15 `_Jogadores` DAX measures

**Files:**
- Modify: `fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`
- Script: `/tmp/add_jogadores_measures.py` (throwaway, not committed)

**Interfaces:**
- Consumes: `ft_estatisticas_jogador[goals/assists/yellow_cards/red_cards/saves/clean_sheets/goals_conceded/minutes_played/average_rating]` (raw column names, unrenamed), `dim_jogadores[player_name/position]` via Task 1's relationship.
- Produces: 15 measures in the `_Jogadores` display folder, which Tasks 4–6 bind to by exact name:
  - `Gols (Jogador)`, `Assistências (Jogador)`, `Participações em Gols`, `Participações em Gols por 90`, `Cartões Amarelos (Jogador)`, `Cartões Vermelhos (Jogador)`, `Defesas`, `Jogos sem Sofrer Gol`, `Gols Sofridos (Jogador)`, `Nota Média (Jogador)`, `Minutos Jogados (Jogador)` (11 numeric measures)
  - `Artilheiro do Torneio`, `Garçom do Torneio`, `Melhor Goleiro (Defesas)`, `Melhor Nota Média` (4 text measures, each `"<nome> — <valor>"`, for the 4 KPI cards in Task 4)

**Naming note — broader than the spec's literal wording:** the design spec says the `(Jogador)` suffix applies "only where a same-named team-level measure already exists" (true collision case: `Gols Sofridos`). This plan applies the suffix to every aggregate measure that has a plausible team-level namesake in spirit (`Assistências`, `Cartões Amarelos`/`Vermelhos`, `Nota Média`, `Minutos Jogados`) even where today's model has no literal collision — so the `_Jogadores` folder reads consistently without the viewer needing to remember which of the 11 names happen to collide. `Defesas`, `Jogos sem Sofrer Gol`, `Participações em Gols`, and `Participações em Gols por 90` keep the Genie's exact names unsuffixed, since those have no team-level equivalent at all.

- [x] **Step 1: Confirm none of these 15 names already exist**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^\tmeasure " tables/_Measures.tmdl   # expect 22 (the existing count)
grep "measure 'Gols (Jogador)'\|measure Defesas\|measure 'Artilheiro do Torneio'" tables/_Measures.tmdl
cd ../../..
```

Expected: 22 for the count, no output for the name grep (nothing named this way exists yet). If the count isn't 22, stop and re-read the file before continuing — something changed since this plan was written.

- [x] **Step 2: Write the measure-insertion script**

Appends the 15 new measures right after the last existing measure (`Pontos Médios por Seleção`, `lineageTag: 00000000-0000-0000-0000-000000000404`) and before the `column _dummy` block, following the exact same `measure / formatString / displayFolder / lineageTag` shape as the 22 existing measures.

```python
# /tmp/add_jogadores_measures.py
from pathlib import Path

PATH = Path("fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl")
text = PATH.read_text()

MARKER = "\tcolumn _dummy\n"
assert MARKER in text, "expected marker 'column _dummy' not found — file layout changed"

# (name, dax, format_string, lineage_suffix)
MEASURES = [
    ("Gols (Jogador)", "SUM(ft_estatisticas_jogador[goals])", "#,##0", "501"),
    ("Assistências (Jogador)", "SUM(ft_estatisticas_jogador[assists])", "#,##0", "502"),
    ("Participações em Gols", "[Gols (Jogador)] + [Assistências (Jogador)]", "#,##0", "503"),
    ("Participações em Gols por 90",
     "DIVIDE(90 * [Participações em Gols], SUM(ft_estatisticas_jogador[minutes_played]))",
     "#,##0.00", "504"),
    ("Cartões Amarelos (Jogador)", "SUM(ft_estatisticas_jogador[yellow_cards])", "#,##0", "505"),
    ("Cartões Vermelhos (Jogador)", "SUM(ft_estatisticas_jogador[red_cards])", "#,##0", "506"),
    ("Defesas", "SUM(ft_estatisticas_jogador[saves])", "#,##0", "507"),
    ("Jogos sem Sofrer Gol", "SUM(ft_estatisticas_jogador[clean_sheets])", "#,##0", "508"),
    ("Gols Sofridos (Jogador)", "SUM(ft_estatisticas_jogador[goals_conceded])", "#,##0", "509"),
    ("Nota Média (Jogador)", "AVERAGE(ft_estatisticas_jogador[average_rating])", "0.0", "510"),
    ("Minutos Jogados (Jogador)", "SUM(ft_estatisticas_jogador[minutes_played])", "#,##0", "511"),
]

# Text measures follow a shared TOPN/ADDCOLUMNS pattern: rank dim_jogadores[player_name]
# by a numeric measure within the current filter context (page filter + slicers already
# applied upstream — see Task 3's page-level minutes_played >= 270 filter), take row 1,
# concatenate name and value. "Melhor Goleiro" additionally pins position = "GK" inside
# CALCULATETABLE so it always names a goalkeeper regardless of the page's Posição slicer.
TEXT_MEASURES = [
    ("Artilheiro do Torneio", '''
	VAR RankedPlayers = ADDCOLUMNS(VALUES(dim_jogadores[player_name]), "@V", [Gols (Jogador)])
	VAR TopRow = TOPN(1, RankedPlayers, [@V], DESC)
	VAR TopName = MAXX(TopRow, dim_jogadores[player_name])
	VAR TopValue = MAXX(TopRow, [@V])
	RETURN IF(ISBLANK(TopName), "Sem dados", TopName & " — " & TopValue & " gols")''', "512"),
    ("Garçom do Torneio", '''
	VAR RankedPlayers = ADDCOLUMNS(VALUES(dim_jogadores[player_name]), "@V", [Assistências (Jogador)])
	VAR TopRow = TOPN(1, RankedPlayers, [@V], DESC)
	VAR TopName = MAXX(TopRow, dim_jogadores[player_name])
	VAR TopValue = MAXX(TopRow, [@V])
	RETURN IF(ISBLANK(TopName), "Sem dados", TopName & " — " & TopValue & " assist.")''', "513"),
    ("Melhor Goleiro (Defesas)", '''
	VAR RankedPlayers =
		CALCULATETABLE(
			ADDCOLUMNS(VALUES(dim_jogadores[player_name]), "@V", [Defesas]),
			dim_jogadores[position] = "GK"
		)
	VAR TopRow = TOPN(1, RankedPlayers, [@V], DESC)
	VAR TopName = MAXX(TopRow, dim_jogadores[player_name])
	VAR TopValue = MAXX(TopRow, [@V])
	RETURN IF(ISBLANK(TopName), "Sem dados", TopName & " — " & TopValue & " defesas")''', "514"),
    ("Melhor Nota Média", '''
	VAR RankedPlayers = ADDCOLUMNS(VALUES(dim_jogadores[player_name]), "@V", [Nota Média (Jogador)])
	VAR TopRow = TOPN(1, RankedPlayers, [@V], DESC)
	VAR TopName = MAXX(TopRow, dim_jogadores[player_name])
	VAR TopValue = MAXX(TopRow, [@V])
	RETURN IF(ISBLANK(TopName), "Sem dados", TopName & " — " & FORMAT(TopValue, "0.0"))''', "515"),
]

def quote(name: str) -> str:
    return f"'{name}'" if any(c in name for c in " %()/áçõêíã") else name

block = ""
for name, dax, fmt, suffix in MEASURES:
    block += f"\tmeasure {quote(name)} = {dax}\n"
    block += f"\t\tformatString: {fmt}\n"
    block += f"\t\tdisplayFolder: _Jogadores\n"
    block += f"\t\tlineageTag: 00000000-0000-0000-0000-000000000{suffix}\n\n"

for name, dax_body, suffix in TEXT_MEASURES:
    block += f"\tmeasure {quote(name)} ={dax_body}\n"
    block += f"\t\tdisplayFolder: _Jogadores\n"
    block += f"\t\tlineageTag: 00000000-0000-0000-0000-000000000{suffix}\n\n"

new_text = text.replace(MARKER, block + MARKER, 1)
assert new_text != text
PATH.write_text(new_text)
print(f"Inserted {len(MEASURES) + len(TEXT_MEASURES)} measures into {PATH}")
```

- [x] **Step 3: Run it**

```bash
cd "BI - Semana da Informática"
python3 /tmp/add_jogadores_measures.py
```

Expected output: `Inserted 15 measures into fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl`

- [x] **Step 4: Spot-check the result**

```bash
cd "BI - Semana da Informática/fifa-world-cup-2026.SemanticModel/definition"
grep -c "^\tmeasure " tables/_Measures.tmdl               # expect 37 (22 + 15)
grep -c "displayFolder: _Jogadores" tables/_Measures.tmdl  # expect 15
grep -A6 "measure 'Artilheiro do Torneio'" tables/_Measures.tmdl
cd ../../..
```

Confirm the `Artilheiro do Torneio` block's `VAR`/`RETURN` lines are indented with tabs, matching the surrounding file, and that `column _dummy` still immediately follows the last inserted measure's blank line.

- [x] **Step 5: Static validation (agent gate)** — found and fixed a real blocker: the 4 text measures' VAR/RETURN lines were under-indented (same depth as `measure`), which TMDL's parser would read as stray top-level keywords rather than expression body. Re-indented to 3 tabs; re-validated clean (0 errors).

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel`. It checks TMDL syntax and referential integrity but **cannot evaluate DAX** — whether `TOPN`/`ADDCOLUMNS`/`CALCULATETABLE` produce the right player name, and whether `Melhor Goleiro (Defesas)` really always resolves to a goalkeeper, are **user gates**, checked in Task 9.

- [x] **Step 6: Commit**

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.SemanticModel/definition/tables/_Measures.tmdl
git commit -m "$(cat <<'EOF'
feat: add the 15-measure _Jogadores DAX layer

11 aggregate measures over ft_estatisticas_jogador (goals, assists,
G+A, G+A per 90, cards, saves, clean sheets, goals conceded, rating,
minutes) plus 4 TOPN-based text measures that name the tournament's
top scorer, top assister, best goalkeeper, and best-rated player for
the KPI row of the new Jogadores page. Names reuse the Genie glossary
1:1 where it exists (Participações em Gols, Participações em Gols por
90, Jogos sem Sofrer Gol); the rest get a "(Jogador)" suffix only
where a team-level measure of the same root name already exists, to
keep every measure name unique across the model.
EOF
)"
```

---

## Phase B — Report page

### Task 3: Page shell — title, slicers, page-level minute filter

**Files:**
- Create: `fifa-world-cup-2026.Report/definition/pages/<new-page-id>/page.json` (via `pbir add page`)
- Create: slicer/title visuals under that page's `visuals/`

**Interfaces:**
- Consumes: `dim_selecoes[Seleção]`, `dim_jogadores[position]`, `_Jogadores` measures (Task 2) not yet used here but the page filter references `ft_estatisticas_jogador[minutes_played]` directly (raw column, unrenamed).
- Produces: a page named `Jogadores`, header band (title + 2 slicers) at `y=24, h=48` matching the other 3 pages' position, and a page-scoped filter `ft_estatisticas_jogador[minutes_played] >= 270` that Tasks 4–6's visuals all inherit. **Does not** add the Fase slicer — deliberate, see spec "Achado 2" (this table has no `match_id`/`stage_id`, so a Fase slicer here would be visibly present but never filter anything).

- [x] **Step 1: Add the page** — used `-n`/`--display-name` (the plan's `--name` doesn't exist in pbir 0.9.28) and a page path `"fifa-world-cup-2026.Report/Jogadores.Page"` (the plan's report-only path errors: "'add page' needs a page path, not a report path"). Page id `17f5a29a0a0d84a7`, appended as pageOrder[3] automatically.

```bash
cd "BI - Semana da Informática"
pbir add page "fifa-world-cup-2026.Report" --name "Jogadores" --width 1280 --height 720
pbir pages json "fifa-world-cup-2026.Report/Jogadores.Page"
```

Confirm `width: 1280, height: 720` in the output.

- [x] **Step 2: Order the new page after "Análises Avançadas"** — no-op needed: `pbir add page` already appended it as the 4th `pageOrder` entry, `activePageName` untouched.

```bash
pbir pages order --help
```

Use whatever `--help` shows to append `Jogadores` as the 4th entry in `pageOrder` (after the existing 3), without disturbing `activePageName`.

- [x] **Step 3: Add the page-level minute filter** — actual syntax: `pbir add filter ft_estatisticas_jogador minutes_played -p "fifa-world-cup-2026.Report/Jogadores.Page" --type Advanced --operator GreaterThanOrEqual --values 270`, then `pbir filters hide`/`pbir filters lock` on the resulting filter path. Confirmed `GreaterThanOrEqual` is a valid enum value (CLI lists valid operators on error).

```bash
pbir add filter --help
```

Add a basic filter, `ft_estatisticas_jogador[minutes_played] >= 270`, scoped to this page only (not report-wide — the other 3 pages don't touch this table). Use the exact flag names `--help` printed — same discovery-first approach as the original plan's Task 8. Hide it from the filter pane and lock it, same as the report-level status filter:

```bash
pbir add filter "fifa-world-cup-2026.Report/Jogadores.Page" --field "ft_estatisticas_jogador[minutes_played]" --type basic --operator ">=" --values 270 --scope page
pbir filters pane-hide "fifa-world-cup-2026.Report/Jogadores.Page/filter:<FilterName>"
pbir filters lock "fifa-world-cup-2026.Report/Jogadores.Page/filter:<FilterName>"
```

- [x] **Step 4: Add the page title** — `pbir add page` already auto-creates a "Jogadores" title textbox (at 20,20,480,120); repositioned it with `pbir visuals position ... --x 24 --y 24 --width 500 --height 48` to match the *actual* precedent on pages 1-3 (all three use width 500, not the plan's assumed 300 — verified by inspecting their visual.json files).

- [x] **Step 5: Add the Seleção slicer** (sync dropped — see note below) — `pbir add visual slicer ... -n slicer_selecao -x 660 -y 24 -w 300 -h 48`, then `pbir visuals bind ... --add "Values:dim_selecoes.Seleção" --type Column` (plan's `--field` flag doesn't exist; bind uses `--add role:field`).

```bash
pbir schema describe slicer syncGroup
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" slicer --name slicer_selecao --x 660 --y 24 --width 300 --height 48
pbir visuals bind "fifa-world-cup-2026.Report/Jogadores.Page/slicer_selecao.Visual" --field "dim_selecoes[Seleção]"
```

~~Apply the same `syncGroup` name used for `slicer_selecao` on the other 3 pages~~ — **this step is unimplementable as written and is dropped.** Verified empirically: `grep -rli sync fifa-world-cup-2026.Report/` returns zero hits anywhere in the report tree, the original 3-page plan's Task 8 Step 6 (which was supposed to create this sync group) was left unchecked (`[ ]`), and `pbir schema describe slicer syncGroup` errors with `'syncGroup' is not a valid container for 'slicer'`. Slicer sync is report-level state in Power BI (Exibir → Sincronizar Segmentações in Desktop), not per-visual JSON — `pbir` operates on per-visual files and has no surface for it. Pages 1–3 never had sync; this page's `slicer_selecao` is added exactly like `slicer_posicao` below, page-local, with no sync group. If cross-page sync is wanted, it's a one-time manual toggle in Desktop, out of scope for this plan.

- [x] **Step 6: Add the page-local Posição slicer** — same corrected `add visual`/`bind` syntax as Step 5, bound to `dim_jogadores.position`.

```bash
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" slicer --name slicer_posicao --x 976 --y 24 --width 280 --height 48
pbir visuals bind "fifa-world-cup-2026.Report/Jogadores.Page/slicer_posicao.Visual" --field "dim_jogadores[position]"
```

No `syncGroup` on this one — it's specific to this page, the other 3 pages have no player-position concept.

- [x] **Step 7: Validate (agent gate)** — passed: "Pages: 4, Visuals: 34, Fields: 105 (model loaded)", Valid (1 info).

```bash
pbir validate "fifa-world-cup-2026.Report" --fields
```

Expected: no errors — confirms `dim_selecoes[Seleção]` and `dim_jogadores[position]` resolve, and that the page-level filter's field reference is valid.

- [x] **Step 8: Commit** — `63d610c`.

```bash
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: add the Jogadores page shell

New 4th page with the same header-band signature as the other 3
(title + Seleção slicer, synced report-wide) plus a page-local Posição
slicer and a hidden/locked page filter (minutes_played >= 270,
matching the Genie's own "Minutagem Mínima" cutoff) that every visual
in Tasks 4-6 inherits. No Fase slicer here by design — this table has
no match/stage grain for it to filter.
EOF
)"
```

---

### Task 4: KPI row — 4 spotlight cards

**Files:**
- Create: visuals under `fifa-world-cup-2026.Report/definition/pages/Jogadores/visuals/`

**Interfaces:**
- Consumes: `_Jogadores[Artilheiro do Torneio]`, `[Garçom do Torneio]`, `[Melhor Goleiro (Defesas)]`, `[Melhor Nota Média]` (Task 2).
- Produces: the completed KPI row, checked by Task 7's full validation pass.

Grid (same as the other 3 pages): `y=88`, 4 × 296w × 100h, `x = 24 / 336 / 648 / 960`.

- [x] **Step 1: Add the 4 cards** — card's data role is `Values` (not `Fields`); confirmed `pbir schema describe card` has no display-units/label-precision property that would garble a text measure's raw string, so no override was needed.

```bash
pbir schema describe card
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" card --name kpi_artilheiro --x 24 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_artilheiro.Visual" --field "_Measures[Artilheiro do Torneio]"

pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" card --name kpi_garcom --x 336 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_garcom.Visual" --field "_Measures[Garçom do Torneio]"

pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" card --name kpi_melhor_goleiro --x 648 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_melhor_goleiro.Visual" --field "_Measures[Melhor Goleiro (Defesas)]"

pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" card --name kpi_melhor_nota --x 960 --y 88 --width 296 --height 100
pbir visuals bind ".../kpi_melhor_nota.Visual" --field "_Measures[Melhor Nota Média]"
```

(All 4 measures return text, not a number — confirm `card` renders a text measure's raw string rather than trying to apply `formatString`/unit abbreviation to it; if `pbir schema describe card` in Step 1's first command shows a `displayUnits`/`labelPrecision` property that would garble text, explicitly disable it on all 4 visuals.)

- [x] **Step 2: Validate (agent gate)** — only the pre-existing `$schema` baseline error, no new warnings on this page.

```bash
pbir validate "fifa-world-cup-2026.Report/Jogadores.Page" --qa
```

- [x] **Step 3: Commit** — `a5b504a`.

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: add the Jogadores page KPI row

4 spotlight cards naming the tournament's top scorer, top assister,
best goalkeeper by saves, and best-rated player — each a single text
measure from Task 2, so the card shows "Nome — valor" instead of a
bare number.
EOF
)"
```

---

### Task 5: Analytical row — artilharia/assistências e ranking de goleiros

**Files:**
- Create: `chart_artilharia` and `chart_goleiros` visuals under `fifa-world-cup-2026.Report/definition/pages/Jogadores/visuals/`

**Interfaces:**
- Consumes: `dim_jogadores[player_name]`, `_Jogadores[Gols (Jogador)]`, `[Assistências (Jogador)]`, `[Defesas]`, `[Jogos sem Sofrer Gol]`, `[Gols Sofridos (Jogador)]`.
- Produces: the 2 analytical-row visuals, checked by Task 7's full validation pass.

Grid: `y=204`, 2 × 608w × 238h, `x = 24 / 648`.

- [x] **Step 1: Top 10 Artilharia + Assistências — clustered bar chart** — roles are `Category` (Column, single) and `Y` (Measure, multiple), confirmed via `pbir visuals bind --list-roles`. Sort: `pbir visuals sort <path> --field "_Measures.Gols (Jogador)" --direction Descending` (a real, previously-undiscovered subcommand — not `visuals cf`). Top 10: `pbir add filter <table> <field> -v <visual path> --type TopN --n 10 --by-table _Measures --by-field "Gols (Jogador)" --direction Top`.

```bash
pbir schema describe clusteredBarChart
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" clusteredBarChart --name chart_artilharia --x 24 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_artilharia.Visual" --field "dim_jogadores[player_name]" --role Category
pbir visuals bind ".../chart_artilharia.Visual" --field "_Measures[Gols (Jogador)]" --role Y
pbir visuals bind ".../chart_artilharia.Visual" --field "_Measures[Assistências (Jogador)]" --role Y
```

(Confirm exact role names via Step 1's `schema describe` output — `Y`/`Values` or similar — before running the `bind` calls, same discovery-first pattern used throughout this report.)

Sort descending by `Gols (Jogador)`, limit to the top 10 players — check `pbir visuals cf --help` / `pbir pages json` sort syntax first (same commands the original plan's `table_top10` step used).

- [x] **Step 2: Ranking de Goleiros — clustered bar chart, filtered to GK** — visual-level filter `dim_jogadores.position` type Categorical, value `GK`; sort descending by `Defesas`; TopN 10 same pattern as Step 1.

```bash
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" clusteredBarChart --name chart_goleiros --x 648 --y 204 --width 608 --height 238
pbir visuals bind ".../chart_goleiros.Visual" --field "dim_jogadores[player_name]" --role Category
pbir visuals bind ".../chart_goleiros.Visual" --field "_Measures[Defesas]" --role Y
pbir visuals bind ".../chart_goleiros.Visual" --field "_Measures[Jogos sem Sofrer Gol]" --role Y
pbir visuals bind ".../chart_goleiros.Visual" --field "_Measures[Gols Sofridos (Jogador)]" --role Y
pbir add filter --help
```

Add a **visual-level** filter (not page-level — the artilharia chart on the left must still include goalkeepers who happen to have an assist) `dim_jogadores[position] = "GK"` scoped to `chart_goleiros` only. Sort descending by `Defesas`, limit to top 10.

- [x] **Step 3: Alignment and spacing check (agent gate)** — only the pre-existing baseline `$schema` error, no new warnings.

```bash
pbir validate "fifa-world-cup-2026.Report/Jogadores.Page" --qa
```

- [x] **Step 4: Commit** — `b585e89`.

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: add the Jogadores page analytical row

Top 10 Artilharia + Assistências (clustered bars, all positions) and
a Ranking de Goleiros (Defesas / Jogos sem Sofrer Gol / Gols Sofridos,
visual-level filtered to position = GK so it doesn't compete with the
page's Posição slicer for report readers who want to see everyone).
EOF
)"
```

---

### Task 6: Detail table — tabela completa de jogadores, with footnote

**Files:**
- Create: `table_jogadores` and `footnote_minutagem` visuals under `fifa-world-cup-2026.Report/definition/pages/Jogadores/visuals/`

**Interfaces:**
- Consumes: `dim_jogadores[player_name/position]`, `dim_selecoes[Seleção]`, `_Jogadores[Gols (Jogador)]`, `[Assistências (Jogador)]`, `[Cartões Amarelos (Jogador)]`, `[Cartões Vermelhos (Jogador)]`, `[Nota Média (Jogador)]`.
- Produces: the completed page, checked by Task 7's full validation pass.

Grid: `y=458`, 1 × 1232w × 238h, `x = 24` (458 + 238 + 24 = 720, same arithmetic as every other page).

- [x] **Step 1: Add the table** — tableEx's data role is `Values` (schema describe's "Columns" label is a display label, not the bind key), same gotcha as `card`. Height 208 (not the Grid section's "238h", which conflicts with the arithmetic for the footnote below it — 208 is what the plan's own command example used, and it's the value consistent with `458 + 208 + 8-gap + 22 (footnote) + 24 = 720`).

```bash
pbir schema describe tableEx
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" tableEx --name table_jogadores --x 24 --y 458 --width 1232 --height 208
pbir visuals bind ".../table_jogadores.Visual" --field "dim_jogadores[player_name]"
pbir visuals bind ".../table_jogadores.Visual" --field "dim_selecoes[Seleção]"
pbir visuals bind ".../table_jogadores.Visual" --field "dim_jogadores[position]"
pbir visuals bind ".../table_jogadores.Visual" --field "_Measures[Gols (Jogador)]"
pbir visuals bind ".../table_jogadores.Visual" --field "_Measures[Assistências (Jogador)]"
pbir visuals bind ".../table_jogadores.Visual" --field "_Measures[Cartões Amarelos (Jogador)]"
pbir visuals bind ".../table_jogadores.Visual" --field "_Measures[Cartões Vermelhos (Jogador)]"
pbir visuals bind ".../table_jogadores.Visual" --field "_Measures[Nota Média (Jogador)]"
```

Sort descending by `Gols (Jogador)` by default (viewer can re-sort any column by clicking its header — no row limit here, unlike the two ranking charts, since this is the full-detail view).

- [x] **Step 2: Footnote about the minutage cutoff** — used `pbir add title` (not `pbir add visual textbox -t`), then repositioned + `pbir set ...text.fontSize 10` to match the existing footnote pattern on page 2's `Title_2` visual (body-paragraph textbox, header hidden). `add visual textbox --title` writes into the visual-header title property instead, which would have rendered with a visible header bar, inconsistent with every other textbox in the report.

```bash
pbir schema describe textbox
pbir add visual "fifa-world-cup-2026.Report/Jogadores.Page" textbox --name footnote_minutagem --x 24 --y 674 --width 1232 --height 22
```

Set its text to:

> Exibe apenas jogadores com 270 minutos ou mais em campo no torneio (equivalente a 3 partidas completas), mesmo corte usado pelo agente Genie — evita que jogadores com poucos minutos distorçam os rankings.

- [x] **Step 3: Validate (agent gate)** — still only the 1 pre-existing baseline error; warnings grew from 56 to 67, all in the same 5 codes already present on pages 1-3 (not a new category), proportional to this page's own header slicers/KPI cards/visual filters/sub-floor textboxes.

```bash
pbir validate "fifa-world-cup-2026.Report/Jogadores.Page" --qa
pbir validate "fifa-world-cup-2026.Report" --all
```

- [x] **Step 4: Commit** — `2d932b9`.

```bash
cd "BI - Semana da Informática"
git add fifa-world-cup-2026.Report/
git commit -m "$(cat <<'EOF'
feat: add the Jogadores page detail table and footnote

Full sortable player table (nome, seleção, posição, gols,
assistências, cartões, nota média) plus the mandatory footnote
documenting the 270-minute cutoff shared with the Genie agent's
"Minutagem Mínima" filter — completes the Jogadores page.
EOF
)"
```

---

### Task 7: Full validation pass and push

**Files:** none (validation + git operations only)

**Interfaces:**
- Consumes: everything from Tasks 1–6.
- Produces: a pushed `main` branch on `fifa-world-cup-2026-bi`, or a documented list of failures if something doesn't pass.

- [x] **Step 1: Model validation (agent gate)** — clean pass, no regression: `validate_pbip.py` (0/0), `tmdl-validate` directory mode (0/0), and single-file mode on all 19 definition files (0/0 each). New relationship and all 15 measures confirmed present.

Dispatch `pbip-validator` against `fifa-world-cup-2026.SemanticModel` one more time on the final state. Zero errors expected.

- [x] **Step 2: Report validation (agent gate)** — **not literally zero errors**, deviating from the plan's expectation, but the 1 error is pre-existing and out of this plan's scope: `fifa-world-cup-2026.SemanticModel/definition.pbism` missing `$schema`, unchanged since the initial commit (`768c202`), confirmed via `git log`. Field-reference validation for the new page is clean (144 fields loaded, 0 reference errors). Warning count grew from the pre-plan baseline of 56 to 67 — all in the same 5 codes already present on pages 1-3 (`SCHEMA_DEGRADED`/`VISUAL_UNDERSIZED`/`VISUAL_LEVEL_FILTERS`/`TEXTBOX_HEIGHT_BELOW_FLOOR`/`TOO_MANY_FIELDS`), proportional to this page's own slicers/cards/filters — not a new problem category.

```bash
cd "BI - Semana da Informática"
pbir validate "fifa-world-cup-2026.Report" --all
```

Zero errors expected across all 4 pages now.

- [x] **Step 3: Color gate (agent gate)** — 2 hex-literal hits, both pre-existing (in `chart_pontos_acima` and `table_top10`, both from commit `2648340`, before this plan). Zero hits inside the Jogadores page's own files.

```bash
grep -rE '#[0-9A-Fa-f]{6}' "fifa-world-cup-2026.Report/definition/pages/" 2>/dev/null
```

Expected: no output — same gate as the original 3-page build, now covering the 4th page too.

- [x] **Step 4: Slicer/filter propagation spot-check (agent gate, structural only)** — confirmed `0`.

```bash
pbir pages json "fifa-world-cup-2026.Report/Jogadores.Page" | grep -c "dim_etapas"
```

Expected: `0`. Confirms the deliberate absence of a Fase slicer/filter on this page (spec "Achado 2") wasn't accidentally added by any `pbir` default.

- [x] **Step 5: Design gate checklist** — header band present and positioned identically to pages 1-3 (y=24, h=48); one intent (individual-performance rankings); shared grid margins/gaps preserved throughout. One open risk, not fixable without rendering: KPI cards are 296px wide, and a long player name (e.g. "Cristiano Ronaldo dos Santos Aveiro — 12 gols", 45 chars) may wrap or truncate — flagged to Task 9 for the user's visual pass in Desktop, since this environment can't render a screenshot to confirm either way.

Walk the `pbi-report-design` skill's closing checklist against the new page: identity propagated (header band present, same `y=24, h=48` position as pages 1–3), one intent for the page ("melhores desempenhos individuais"), spacing on the shared grid (margin 24 / gap 16, same as the other 3 pages), KPIs are legible as text (not truncated — check card width against the longest plausible name, e.g. a 20+ character player name plus " — 12 gols"). Note any deviation found; fix inline if small, otherwise list it in Task 9.

- [x] **Step 6: Push**

```bash
git push
git log --oneline -10
```

- [ ] **Step 7: If any gate in Steps 1–5 failed** — n/a, no fresh gate failed (the 1 Report-validation error and the 2 color hits both pre-date this plan).

Do not push. Fix the specific failure, re-run only the gate that failed, then retry Step 6.

---

### Task 8: (informational — no checkbox) What this plan does not verify

Same two gaps as the original 3-page plan, now extended to this page:

1. **Do the 15 new measures evaluate to the right values?** `pbip-validator` checks TMDL syntax, not DAX semantics. The 4 `TOPN`/`ADDCOLUMNS`/`CALCULATETABLE` text measures are the highest-risk DAX in this plan — nothing in Tasks 1–7 confirms `Artilheiro do Torneio` actually names the player with the most goals, or that `Melhor Goleiro (Defesas)` really always resolves to a `GK` even when the Posição slicer is set to something else.
2. **Does the page render correctly?** `pbir validate --qa` catches overlap/overflow from the JSON's numbers; it cannot render a screenshot or confirm the two bar charts are legibly sorted top-10. `pbir desktop screenshot` is Windows-only.

Task 9 is the checklist for the user to run these two checks themselves.

---

### Task 9: User acceptance checklist (manual — not agent-executable)

Hand this checklist to the user once Task 7 has pushed successfully. Every item requires opening the file in Power BI Desktop on Windows.

- [ ] Open `fifa-world-cup-2026.pbip` in Power BI Desktop. It should load with no error dialog.
- [ ] Field list check: `_Measures` should now show 37 measures across 5 folders (`_Torneio`, `_Desempenho`, `_Avançado`, `_Referência`, `_Jogadores`).
- [ ] Open the "Jogadores" page. Confirm the 4 KPI cards show readable text like `"<nome> — N gols"`, not `#ERROR`, blank, or a bare number.
- [ ] Cross-check `Artilheiro do Torneio` against a direct query: sort `ft_estatisticas_jogador` by `goals` descending (via a Genie question, e.g. "quem tem mais gols no torneio, com pelo menos 270 minutos jogados?", or a manual look at `player_stats.csv`) and confirm the name and number match the card.
- [ ] Confirm `Melhor Goleiro (Defesas)` names an actual goalkeeper (`position = 'GK'` in `player_stats.csv`) even after selecting a non-GK value in the page's Posição slicer — this is the specific case the `CALCULATETABLE` override in Task 2 exists for.
- [ ] Select a seleção in the Jogadores page's Seleção slicer. Confirm it narrows this page's 2 charts, KPI cards, and detail table to that seleção's players. (Cross-page sync with pages 1–3 does **not** exist — never did, on any of the 4 pages; it's a manual Desktop toggle, Exibir → Sincronizar Segmentações, out of scope for this plan.)
- [ ] Select a posição in the page's own Posição slicer (e.g. `GK`). Confirm the artilharia chart, goleiros chart, and detail table all narrow accordingly, and that the KPI cards stay stable (they compute over their own filter context, not the ambient slicer, except where deliberately overridden).
- [ ] Confirm the two analytical-row charts show exactly 10 bars each, sorted descending.
- [ ] Screenshot the Jogadores page (Desktop's own export, or `pbir desktop screenshot` from the Windows side) and do a visual pass: legible KPI text, no visual overlapping another, header band identical in position to pages 1–3, footnote visible below the table.
- [ ] Confirm the deliberate absence of a Fase slicer on this page reads as intentional, not as a missing feature — the footnote and page title should be enough context; if reviewers keep asking "where's the fase filter", consider adding a short explanatory note (out of scope for this plan, flag back to the user if it comes up).
