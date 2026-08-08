# Página "Jogadores" — 4ª página do dashboard Power BI

Data: 2026-08-08

## Objetivo

Adicionar uma 4ª página ao relatório (`fifa-world-cup-2026.Report`) dedicada a
desempenho individual: artilharia, criação de jogo, goleiros, disciplina e
nota média. A tabela `ft_estatisticas_jogador` já está importada no modelo
semântico desde a construção original do relatório, mas não tem nenhuma
medida DAX nem visual — este documento cobre modelo semântico + página de
relatório para tirá-la do papel.

## Contexto: por que isso estava fora de escopo e por que muda agora

A spec original do dashboard (`2026-08-08-dashboard-power-bi-design.md`)
descartou explicitamente uma página de jogadores, em favor de 3 páginas
focadas em seleção/torneio. Na época, o agente Genie
(`Databricks/oficina-semana-informatica-2026/metadata/copa_mundo_2026_metadata.py`)
tinha só 4 medidas, todas sobre `ft_partidas`.

O Genie foi atualizado desde então e agora tem um glossário maduro para
estatística de jogador: medidas `Participações em Gols` e `Participações em
Gols por 90`, filtros `Somente Goleiros` / `Somente Jogadores de Linha` /
`Minutagem Mínima (3 jogos)`, e a expressão `Idade na Abertura do Torneio`.
Ele também define explicitamente o join
`ft_estatisticas_jogador.player_id = dim_jogadores.player_id` como
obrigatório em "todo ranking individual". Isso muda o cálculo: replicar essa
página no Power BI deixou de ser reinventar uma regra de negócio e passou a
ser aplicar uma decisão que o Genie já tomou, mantendo os dois artefatos com
o mesmo glossário — mesma motivação já registrada no achado 4 da spec
original (alinhamento de nomenclatura Genie ↔ Power BI).

**Repositório:** este documento e a página que ele descreve vivem em
`fifa-world-cup-2026-bi` (pasta `BI - Semana da Informática`), commitados com
`git -C "BI - Semana da Informática"` — nunca no repositório de Databricks,
seguindo a regra já registrada em memória de projeto (`project-repo-split-bi-dashboard`).

## Achados que definem a arquitetura

### 1. Relação `player_id → dim_jogadores` está ausente

Hoje `relationships.tmdl` só tem `ft_estatisticas_jogador.team_id →
dim_selecoes.team_id`. Sem a relação por `player_id`, nenhum visual consegue
resolver o nome do jogador — um ranking mostraria `player_id`, não
`player_name`. **Decisão:** adicionar
`ft_estatisticas_jogador.player_id → dim_jogadores.player_id` (many-to-one,
ativa), replicando o `join_spec` que o Genie já define.

### 2. `ft_estatisticas_jogador` é total-por-torneio, sem `match_id`/`stage_id`

A tabela tem uma linha por jogador com os totais acumulados do torneio — não
tem granularidade de partida nem de fase. Consequência: a segmentação
universal de **Fase** (`dim_etapas[stage_name]`), usada nas 3 páginas atuais,
não tem caminho de filtro até esta tabela. Colocá-la na página 4 violaria o
portão de validação 3 do relatório ("todo visual deve reagir ao slicer de
fase") — o slicer ficaria visível mas mudo.

**Decisão:** a página 4 mantém apenas o slicer de **Seleção**
(`dim_selecoes[team_name]`, que funciona via `team_id`) e ganha um slicer
próprio de **Posição** (`dim_jogadores[position]`, valores confirmados em
`player_stats.csv`: `GK`, `DEF`, `MID`, `FWD`), necessário porque estatística
de goleiro e de jogador de linha não se comparam na mesma régua.

### 3. Corte de minutagem mínima

Sem corte, um jogador com 1 gol em 5 minutos de bola rolando lidera o ranking
de artilharia — o mesmo problema que o Genie já resolveu com o filtro
"Minutagem Mínima (3 jogos)" (`minutes_played >= 270`). **Decisão:** aplicar
`minutes_played >= 270` como filtro de nível de página (não por visual), com
nota de rodapé visível — mesmo padrão já usado na página 3 para a convenção
de pênaltis em `points`/`result`.

### 4. Códigos de posição confirmados

`player_stats.csv` e `squads_and_players.csv` gravam posição como `GK`,
`DEF`, `MID`, `FWD` (confirmado por inspeção direta dos CSVs) — o filtro do
Genie `dim_jogadores.position = 'GK'` é válido como está; a ressalva que o
próprio Genie registra ("se não retornar linhas, confira os códigos") não se
aplica aqui.

## Camada semântica

### Nova relação

```
ft_estatisticas_jogador.player_id → dim_jogadores.player_id   (many-to-one, ativa)
```

### Medidas novas — pasta `_Jogadores` (15 medidas)

Nomes reaproveitados do glossário do Genie onde há correspondência 1:1
(`Participações em Gols`, `Participações em Gols por 90`, `Jogos sem Sofrer
Gol`). Nas demais, sufixo `(Jogador)` só onde já existe um nome parecido de
seleção no modelo (ex.: `Gols Marcados` já existe para seleção em
`_Torneio`, então o equivalente de jogador é `Gols (Jogador)`), para evitar
ambiguidade no painel de campos — medidas precisam de nome único no modelo
inteiro, não só na tabela.

```dax
Gols (Jogador)                = SUM(ft_estatisticas_jogador[goals])
Assistências (Jogador)        = SUM(ft_estatisticas_jogador[assists])
Participações em Gols         = [Gols (Jogador)] + [Assistências (Jogador)]
Participações em Gols por 90  = DIVIDE(90 * [Participações em Gols], SUM(ft_estatisticas_jogador[minutes_played]))
Cartões Amarelos (Jogador)    = SUM(ft_estatisticas_jogador[yellow_cards])
Cartões Vermelhos (Jogador)   = SUM(ft_estatisticas_jogador[red_cards])
Defesas                       = SUM(ft_estatisticas_jogador[saves])
Jogos sem Sofrer Gol          = SUM(ft_estatisticas_jogador[clean_sheets])
Gols Sofridos (Jogador)       = SUM(ft_estatisticas_jogador[goals_conceded])
Nota Média (Jogador)          = AVERAGE(ft_estatisticas_jogador[average_rating])
Minutos Jogados (Jogador)     = SUM(ft_estatisticas_jogador[minutes_played])
```

Mais 4 medidas de texto, uma por cartão de destaque do topo da página — cada
uma usa `TOPN(1, ..., <medida>, DESC)` sobre `dim_jogadores`/`ft_estatisticas_jogador`
filtrado pelo corte de 270 minutos, concatenando nome e valor:

```dax
Artilheiro do Torneio      = "<nome> — N gols"      (TOPN por [Gols (Jogador)])
Garçom do Torneio          = "<nome> — N assist."    (TOPN por [Assistências (Jogador)])
Melhor Goleiro (Defesas)   = "<nome> — N defesas"    (TOPN por [Defesas], position = "GK")
Melhor Nota Média          = "<nome> — X,X"          (TOPN por [Nota Média (Jogador)])
```

### Filtro de página

`ft_estatisticas_jogador[minutes_played] >= 270`, aplicado a todos os visuais
da página 4 (nível de página, não de visual individual).

## Layout

Mesma grade das 3 páginas existentes — 1280×720, margem 24, espaçamento 16:

```
y=24   faixa: título "Jogadores" + slicers Seleção/Posição       h=48
y=88   linha de KPIs — 4 cartões de texto (nome — valor)         h=100   x = 24 / 336 / 648 / 960
       Artilheiro do Torneio · Garçom do Torneio ·
       Melhor Goleiro (Defesas) · Melhor Nota Média
y=204  linha analítica — 2 × 608w                                 h=238   x = 24 / 648
       Top 10 Artilharia + Assistências (barras clusterizadas,
       ordenado por Gols (Jogador) desc)
       Ranking de Goleiros — Defesas · Jogos sem Sofrer Gol ·
       Gols Sofridos (Jogador), filtro de visual position = "GK"
y=458  linha de detalhe — 1 × 1232w                                h=238   x = 24
       Tabela completa: Jogador · Seleção · Posição · Gols (Jogador) ·
       Assistências (Jogador) · Cartões Amarelos/Vermelhos (Jogador) ·
       Nota Média (Jogador) — ordenável, nota de rodapé com o corte
       de 270 minutos
                                              (458 + 238 + 24 = 720 ✓)
```

Disciplina (cartões) e nota média entram como colunas da tabela-detalhe, não
como visuais próprios — mantém a página em 4 KPIs + 2 gráficos + 1 tabela,
igual às 3 páginas existentes, em vez de acumular 6+ visuais numa página só
(contra a diretriz de "uma intenção por página" da skill `pbi-report-design`).

**Alternativa descartada:** slicer de posição com bookmarks trocando o
conjunto de colunas visíveis (visão "linha" vs. "goleiro"). Mais polido, mas
introduz uma técnica (bookmarks) que nenhuma das 3 páginas existentes usa —
pesa contra a regra de desempate já registrada na spec original: "quando
prática de produção conflitar com o que é explicável em poucos minutos para
um aluno do Ensino Médio, a legibilidade vence."

### Identidade visual

Reaproveita o tema `IFRS_Feliz.json` já existente (mesma paleta verde, mesma
faixa de cabeçalho de 64px com título + segmentações) — nenhuma cor nova,
nenhum ajuste de tema necessário. Toda cor referenciada via `ThemeDataColor`,
nunca `Literal` hex, mesma regra de propagação já em vigor.

## Validação

Mesmos 7 portões já usados nas 3 páginas (ver spec original, seção
"Validação"), aplicados também à página 4:

1. Estrutural — `pbir` valida a cada mutação, agente `pbip-validator` ao
   final.
2. Cor — `grep -rE '#[0-9A-Fa-f]{6}' definition/pages/` vazio.
3. Propagação de filtro — escolher uma seleção no slicer de Seleção e
   confirmar que os 3 visuais de estatística reagem; escolher uma posição no
   slicer de Posição e confirmar que a tabela de goleiros zera quando a
   posição escolhida não é `GK`. **Não testar propagação do slicer de Fase
   nesta página — ele não existe aqui, por design (achado 2).**
4. Números — conferir manualmente o artilheiro/garçom/goleiro/nota do topo
   contra uma consulta direta em `ft_estatisticas_jogador` (ou uma pergunta
   equivalente ao Genie) antes de aceitar os cartões de destaque.
5. Visual — captura de tela da página 4 via Desktop Bridge (skill
   `pbir-cli`) e revisão humana.
6. Design gate — checklist de fechamento da skill `pbi-report-design`.
7. Publicação — commit e push para `fifa-world-cup-2026-bi` (branch `main`)
   só depois dos 6 portões acima passarem.

## Riscos e decisões aceitas

- **Corte de 270 minutos oculta jogadores revelação com poucos jogos:** aceito
  como está, por consistência com o Genie — mesmo trade-off que ele já
  assumiu, comunicado via nota de rodapé.
- **Slicer de Fase ausente nesta página, diferente das outras 3:** decisão
  deliberada (achado 2), não uma omissão — documentado para não ser
  "corrigido" por engano numa revisão futura.
- **Medidas de texto (TOPN) são mais pesadas que `SUM`/`DIVIDE`:** aceitável
  para 4 cartões sobre uma tabela de ~1000 linhas (jogadores × torneio); sem
  otimização adicional prevista.

## Fora de escopo

- Página separada só para goleiros (mantido como um recorte dentro da página
  4, via slicer de Posição, não como página própria).
- Bookmarks ou parâmetros de campo para alternar visão linha/goleiro (ver
  "Alternativa descartada" em Layout).
- Qualquer medida sobre idade (`Idade na Abertura do Torneio`) ou faixa
  etária — o Genie já tem essas expressões, mas nenhum visual desta página
  as usa; fica para uma iteração futura se houver pedido explícito.
