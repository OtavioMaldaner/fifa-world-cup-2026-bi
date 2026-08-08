# Dashboard Power BI — Copa do Mundo 2026 (IFRS Câmpus Feliz)

Data: 2026-08-08

## Objetivo

Construir um relatório Power BI (PBIP) de 3 páginas sobre a Copa do Mundo 2026,
que exiba — com camada de medidas DAX própria — os mesmos dados e a mesma
narrativa de negócio já documentados no agente Genie (`metadata/copa_mundo_2026_metadata.py`).
O relatório é um **artefato pronto**, construído antes e apresentado funcionando
na oficina, ilustrando a jornada Databricks → Genie → Power BI para alunos do
Ensino Médio no IFRS – Câmpus Feliz.

## Contexto

O projeto já tem:

- Um modelo semântico importado (`BI - Semana da Informática/fifa-world-cup-2026.SemanticModel`),
  com 10 tabelas gold + 4 views analíticas, todas em modo *Import*, sem nenhuma
  medida DAX.
- Um agente Genie (`Databricks/oficina-semana-informatica-2026/metadata/copa_mundo_2026_metadata.py`)
  que documenta nomes de negócio em PT-BR, regras de desambiguação e 4 medidas
  (só sobre `ft_partidas`).
- Um relatório PBIP com uma página em branco e um tema customizado importado
  (`Tramontina_Layout13807985785810117.json`), que o usuário pediu para
  substituir por um tema com a identidade visual do IFRS.

Este documento cobre **modelo semântico + relatório**: sem medidas, o
relatório se resumiria a contagens de linha e somas de colunas cruas.

### Dois repositórios

O projeto vive em dois repositórios Git separados, cada um versionando sua
própria pasta:

| Repositório | Pasta | Conteúdo |
|---|---|---|
| `oficina-semana-informatica-2026` (existente) | `Databricks/oficina-semana-informatica-2026` | pipelines, assets, metadata do Genie |
| [`fifa-world-cup-2026-bi`](https://github.com/OtavioMaldaner/fifa-world-cup-2026-bi) (criado em 2026-08-08, público) | `BI - Semana da Informática` | projeto PBIP (relatório + modelo semântico), **e esta spec** |

Consequência para o plano de implementação: todo commit de medida DAX, tema e
página de relatório acontece no repositório `fifa-world-cup-2026-bi`, com
`git -C "BI - Semana da Informática"`. A pasta raiz `oficina-semana-informatica-2026`
(um nível acima dos dois repositórios) não é, ela própria, um repositório —
não há commit "guarda-chuva" que abranja as duas pastas.

**Local dos documentos de processo:** esta spec (e o plano de implementação
que a sucede) vive em `BI - Semana da Informática/docs/superpowers/specs/` —
não no repositório de Databricks. Regra geral daqui em diante: documento de
processo mora no repositório do artefato que ele descreve. Como o BI passou
a ter repositório próprio nesta sessão, esta spec foi movida do repositório
de Databricks (onde nasceu, por ser o único repositório versionável antes de
2026-08-08) para cá.

### Restrição de projeto: legibilidade didática vence

Mesma regra de desempate do refactor de pipelines (`2026-08-05-refactor-pipeline-declarativa-design.md`):
quando prática de produção conflitar com o que é explicável em poucos minutos
para um aluno do Ensino Médio, a legibilidade vence. Por isso o design evita
soluções DAX avançadas (`USERELATIONSHIP` duplicado, visual customizado só para
uma linha de referência) quando existe alternativa mais direta.

## Descoberta que define a arquitetura: `ft_partidas` está desativado

Em `relationships.tmdl`, toda relação de `ft_partidas` para dimensão está
`isActive: false` (`stage_id`, `venue_id`, `home_team_id`, `away_team_id`), e
`dim_jogadores.team_id → dim_selecoes` também. Uma medida DAX escrita
ingenuamente sobre `ft_partidas` ignoraria silenciosamente qualquer
segmentação de seleção, fase ou estádio — o pior tipo de erro porque não gera
mensagem nenhuma, só números errados. Isso é uma armadilha específica do
modelo do Power BI: o Genie não sofre disso, porque compila para SQL direto
contra as tabelas do Databricks, sem depender de relações do modelo tabular.

A tabela `vw_selecao_partida` (adicionada ao repositório durante este
brainstorm, arquivo `data-engineering/transformations/vw_selecao_partida.py`)
resolve isso: granularidade de 1 linha por seleção-partida, com `goals_for`,
`goals_against`, `xg_for`, `xg_against`, `points`, `result`, `opponent_name`
já desnormalizados, e relações **ativas** para `dim_selecoes`, `dim_etapas`,
`dim_estadios` e `dim_arbitros`.

**Decisão de arquitetura:** todas as medidas de partida nascem em
`vw_selecao_partida` (ou nas outras 3 views analíticas, para as métricas
avançadas). `ft_partidas` permanece no modelo só como tabela-ponte (é o lado
"1" das relações ativas) e fica **oculta** do painel de campos para que
ninguém a arraste para um visual por engano.

Duas abordagens alternativas foram descartadas: ativar as relações de
`ft_partidas` exigiria `USERELATIONSHIP` duplicado (mandante + visitante) em
toda medida de seleção — reconstruindo em DAX o que a view já entrega pronto,
e didaticamente pesado. Ativar só uma relação por vez (ex.: `venue_id` e
`stage_id`) criaria caminhos de filtro ambíguos entre `dim_etapas` e as views
analíticas.

## Achados de dados que mudam o comportamento das medidas

### 1. `result` e `points` ignoram disputa de pênaltis

`vw_selecao_partida.py` calcula `points`/`result` comparando só `home_score`/
`away_score` — o placar antes da disputa por pênaltis. Times que avançam nos
pênaltis contam como **empate, 1 ponto**, não vitória. Isso é documentado no
comentário da própria view (regra de negócio, não bug), mas afeta
`Aproveitamento %` e a comparação com xPts na página 3. **Nota de rodapé
obrigatória na página 3** avisando sobre essa convenção.

### 2. 4 partidas "fantasma" com placar em branco

`matches.csv` tem 104 linhas, mas só 100 têm `status = "Completed"`. As 4
`Scheduled` (semifinais 101/102, terceiro lugar 103, final 104) têm placar
vazio. As duas semifinais (101, 102) têm `home_team_id`/`away_team_id`
preenchidos e passam pela expectativa `selecoes_distintas` de
`vw_selecao_partida`; a cascata `F.when(...).otherwise(...)` sem nenhuma
condição verdadeira as classifica como `points = 0, result = "LOSS"` — uma
derrota fantasma para as 4 seleções semifinalistas (33, 29, 45, 37).

**Decisão:** filtro de nível de relatório em `ft_partidas[status] = "Completed"`.
Como `ft_partidas` tem relação ativa 1:N com `vw_selecao_partida`, o filtro
propaga para as páginas 1 e 2 sem precisar de mudança em Databricks.

As outras 3 views analíticas **já excluem essas partidas por construção** e
não precisam do filtro:
- `vw_xpts_selecao_partida.py:49` — `.filter(home_xg.isNotNull() & away_xg.isNotNull())`
- `vw_pontos_recuperados.py` — join *inner* com `ft_eventos`; partida sem gol registrado não entra
- `vw_estabilidade_escalacao.py` — parte de `ft_escalacoes`, que só tem as 100 partidas jogadas (5200 linhas = 100 × 52)

**Valores de aceitação** (conferidos em `matches.csv`, só `status = Completed`):

| medida | valor esperado |
|---|---|
| Total de Partidas | 100 |
| Gols Marcados | 292 |
| Média de Gols/Partida | 2,92 |
| Eficiência Ofensiva (Gols/xG) | 1,100 |
| Partidas decididas nos pênaltis | 4 |

### 3. `dim_estadios` sem latitude/longitude/altitude

`venues.csv` tem `latitude`, `longitude`, `elevation_meters`, mas
`dim_estadios.py` descarta as três colunas no `select`. O Genie promete
análise de altitude que a tabela gold não sustenta hoje. **Fora de escopo por
decisão do usuário**: o mapa da página 1 usa `city`/`country` e deixa o Power
BI geocodificar por nome (decisão aceita: sem altitude, sem coordenada fixa,
com dependência de geocodificação online no momento da renderização — ver
Riscos).

### 4. Genie atualizado (commit `610a8f3`, 2026-08-08): agora conhece as 4 views — e confirma a arquitetura

Este achado estava desatualizado e foi corrigido nesta revisão. O Genie
passou a documentar `vw_selecao_partida`, `vw_xpts_selecao_partida`,
`vw_pontos_recuperados` e `vw_estabilidade_escalacao` como fontes de dados de
primeira classe, com `column_configs` e `join_specs` próprios, e ganhou 20
medidas (eram 4). A página 3 do relatório deixa de ser uma análise que só o
dashboard sabe fazer — o Genie responde as mesmas perguntas hoje.

Mais relevante: a instrução da medida "Saldo de Gols" do Genie diz, quase
literalmente, a mesma coisa que a seção anterior deste documento — *"Não
calcule saldo a partir de `ft_partidas.home_score` menos `away_score`, porque
isso só enxerga o lado mandante... prefira `vw_selecao_partida`"*. A decisão
de arquitetura deste documento (medidas de seleção nascem em
`vw_selecao_partida`) não é uma preferência isolada do relatório — é a mesma
conclusão a que o autor do Genie chegou, de forma independente, para o
SQL. Isso eleva a confiança na decisão.

**Alinhamento de nomenclatura** — como o objetivo declarado da faxina de
colunas é Genie e Power BI compartilharem um glossário único, as medidas DAX
que correspondem 1:1 a uma medida do Genie usam o `display_name` dele,
substituindo o nome provisório proposto neste documento:

| nome provisório (spec) | nome final (glossário do Genie) |
|---|---|
| `Partidas` | `Total de Partidas` |
| `Pontos` | `Pontos Conquistados` |
| `Aproveitamento %` | `Aproveitamento (%)` |
| `Eficiência Gols/xG` | `Eficiência Ofensiva (Gols/xG)` |
| `Estabilidade Média Escalação` | `Estabilidade da Escalação (%)` |

As 16 medidas restantes da lista original mantêm o nome já definido — ou
porque o Genie usa o mesmo nome (`Saldo de Gols`, `Pontos Esperados (xPts)`,
`Pontos Acima do Esperado`, `Pontos Recuperados`), ou porque não têm
equivalente direto no Genie (`Gols Marcados`, `Gols Sofridos`, `Vitórias`,
`xG a Favor`, `xG Contra`, as 4 medidas de referência).

O Genie também ganhou 6 medidas sem equivalente no relatório hoje: `Média de
Gols Sofridos por Jogo`, `xG Diferencial`, `Gols Acima do Esperado`,
`Participação no xG da Partida` (todas sobre `vw_selecao_partida` — mesma
tabela das medidas `_Desempenho` já planejadas), mais `Índice de Zebra`
(métrica de upset, cruzando `points`/`team_rank`/`opponent_rank`) e duas
métricas sobre `ft_estatisticas_equipe`/`ft_estatisticas_jogador` (posse de
bola, precisão de finalização, participações em gols) que pertencem às
páginas de estatística de equipe e de jogador — ambas fora de escopo por
decisão já registrada (ver "Fora de escopo"). **Não incorporadas ao modelo
nesta revisão** — ver decisão pendente ao final desta seção.

## Camada semântica

### Faxina do modelo

1. Desligar *Opções → Carregar Dados → Inteligência de Tempo automática* e
   recarregar — remove as 14 tabelas `LocalDateTable_*` e as 27 relações de
   data associadas. Um torneio de 5 semanas não usa hierarquia de data
   automática.
2. Ocultar `ft_partidas` do painel de campos (permanece como tabela-ponte).
3. Renomear colunas para os nomes de negócio em PT-BR usando os campos
   `display_name` de `metadata.py` como fonte única — Genie e Power BI passam
   a compartilhar o mesmo glossário.

### Medidas (21, em 3 pastas de exibição)

**`_Torneio`** (sobre `vw_selecao_partida`, respeitando o filtro de status da
seção anterior):

```dax
Total de Partidas     = DISTINCTCOUNT(vw_selecao_partida[match_id])
Seleções              = DISTINCTCOUNT(vw_selecao_partida[team_id])
Gols Marcados         = SUM(vw_selecao_partida[goals_for])
Gols Sofridos         = SUM(vw_selecao_partida[goals_against])
Saldo de Gols         = [Gols Marcados] - [Gols Sofridos]
Média de Gols/Partida = DIVIDE([Gols Marcados], [Total de Partidas])
```

**`_Desempenho`** (sobre `vw_selecao_partida`):

```dax
Pontos Conquistados        = SUM(vw_selecao_partida[points])
Vitórias                   = CALCULATE([Total de Partidas], vw_selecao_partida[result] = "WIN")
Aproveitamento (%)         = DIVIDE([Pontos Conquistados], [Total de Partidas] * 3)
xG a Favor                 = SUM(vw_selecao_partida[xg_for])
xG Contra                  = SUM(vw_selecao_partida[xg_against])
Eficiência Ofensiva (Gols/xG) = DIVIDE([Gols Marcados], [xG a Favor])
```

**`_Avançado`** (uma por view analítica, sem filtro de status — desnecessário,
ver seção anterior):

```dax
Pontos Esperados (xPts)      = SUM(vw_xpts_selecao_partida[expected_points])
Pontos Acima do Esperado     = SUM(vw_xpts_selecao_partida[points_above_expected])
Pontos Recuperados           = SUM(vw_pontos_recuperados[recovered_points])
Jogos Atrás no Placar        = CALCULATE(COUNTROWS(vw_pontos_recuperados), vw_pontos_recuperados[was_behind] = TRUE())
Estabilidade da Escalação (%) = AVERAGE(vw_estabilidade_escalacao[lineup_stability_pct])
```

**Medidas de referência** (média do torneio, para dar contexto aos KPIs —
`REMOVEFILTERS(dim_selecoes)` em vez de `ALL()` porque a referência deve
ignorar a seleção escolhida mas **respeitar** o filtro de fase: olhando só o
mata-mata, a meta é a média do mata-mata, não a do torneio inteiro):

```dax
Aproveitamento (%) Médio   = CALCULATE([Aproveitamento (%)], REMOVEFILTERS(dim_selecoes))
Média Gols/Partida Torneio = CALCULATE([Média de Gols/Partida], REMOVEFILTERS(dim_selecoes))
Eficiência Média Torneio   = CALCULATE([Eficiência Ofensiva (Gols/xG)], REMOVEFILTERS(dim_selecoes))
Pontos Médios por Seleção  = CALCULATE(DIVIDE([Pontos Conquistados], [Seleções]), REMOVEFILTERS(dim_selecoes))
```

**Decisão pendente — 4 medidas novas do Genie sobre `vw_selecao_partida`:**
`Média de Gols Sofridos por Jogo`, `xG Diferencial`, `Gols Acima do Esperado`
e `Participação no xG da Partida` usam a mesma tabela das medidas
`_Desempenho` acima e custariam pouco para replicar em DAX. Não foram
incluídas nesta revisão porque adicioná-las muda o número de medidas
aprovado (21 → 25) e nenhuma tem visual designado no layout já aprovado — a
decisão de expandir o modelo ou não fica para o plano de implementação, como
um passo opcional a confirmar com o usuário antes de executar. `Índice de
Zebra` e as medidas de `ft_estatisticas_equipe`/`ft_estatisticas_jogador`
ficam de fora por dependerem de páginas fora de escopo (ver "Fora de
escopo").

## Identidade visual e tema

**Tom: `corporate`** — paleta curada, no máximo 2 acentos por página,
saturação moderada, linhas de grade fracas, rótulos de dados só onde a
leitura exige. Cabe ao contexto de apresentação institucional.

**Assinatura (única, repetida nas 3 páginas):** faixa de cabeçalho de 64px
com título à esquerda e até 2 segmentações à direita, na mesma posição em
toda página — é o que faz o relatório ler como um artefato único.

**Verde institucional:** `#339645` (fornecido pelo usuário como cor primária
do IFRS). Paleta derivada e verificada por contraste (WCAG, mínimo 3:1 contra
o fundo `#F3F3F3` para elementos gráficos, 4,5:1 para texto):

| # | papel | hex | contraste vs. fundo |
|---|---|---|---|
| 1 | verde primário (marca, série principal) | `#339645` | 3,39 |
| 2 | verde escuro (títulos, texto de destaque) | `#1A4C23` | 9,00 |
| 3 | verde azulado | `#307E6E` | 4,37 |
| 4 | oliva | `#537D36` | 4,35 |
| 5 | cinza ardósia (séries secundárias, "outros") | `#5F6B6D` | 4,97 |
| 6 | violeta (polo negativo do par divergente) | `#7B43A3` | 5,96 |
| 7 | ameixa | `#914678` | 5,65 |

Estruturais: texto `#333333` (11,39), grade `#D8D8D8`, borda `#C8C6C4`, fundo
`#F3F3F3`. Um verde-médio intermediário (`#44A756`) foi descartado por medir
2,74 de contraste — reprova o mínimo de 3:1.

**Par divergente:** verde ↔ violeta, não verde ↔ vermelho. O eixo
verde-vermelho é o pior caso possível para deuteranopia e protanopia, os dois
tipos mais comuns de daltonismo — e cairia justo no visual-âncora da página 3
(Pontos Acima do Esperado). Cor codifica significado (acima/abaixo do
esperado) só nesse visual; em todo o resto, cor identifica série, não
sentimento.

### Arquivo de tema: `IFRS_Feliz.json`

O tema atual (`Tramontina_Layout13807985785810117.json`, 161 KB) tem 5
problemas que impedem reaproveitamento direto:

1. `dataColors` com 480 entradas auto-geradas por rotação de matiz — ruído
   depois da oitava cor.
2. 23 valores hex fixos dentro de `visualStyles` (eixos, bordas, realces),
   incluindo o navy `#003087` — não bastaria trocar só `dataColors`.
3. `"name": "Tramontina Layout"` e 8 ícones em base64 de formatação
   condicional (`TrianguloCertoVerde`, `FlechaSubiuVermelha`, ...) — marca
   errada embutida, e os ícones não são usados por este design.
4. **Bug de contraste:** `filterCard` "Available" tem `fontColor: #FFFFFF`
   sobre `backgroundColor: #FFFFFF` — texto branco em fundo branco, os
   filtros disponíveis ficam invisíveis.
5. `dropShadow.show: true` em todos os visuais (6px de distância) — contraria
   a diretriz de acessibilidade (questão vestibular) e enfraquece o
   alinhamento de grade.

**Decisão:** derivar `IFRS_Feliz.json` a partir do arquivo atual, mantendo o
que funciona (Segoe UI em tudo, `valueAxis.show: false`, fundo `#F3F3F3`,
bordas de raio 3) e corrigindo os 5 pontos acima — paleta reduzida a 7 cores
curadas, hex fixos substituídos pela paleta verde, ícones e nome Tramontina
removidos, contraste do `filterCard` corrigido, sombra desligada. O arquivo
antigo sai do `resourcePackages` e do `themeCollection` em `report.json`.

**Regra de propagação:** nenhuma cor em `visual.json` — toda cor referenciada
via `ThemeDataColor`, nunca `Literal` hex. É o que garante que uma futura
troca de paleta seja edição de um arquivo só. Verificado no portão de
validação (seção seguinte).

## Layout

Grade compartilhada pelas 3 páginas — página 1280×720 (confirmado em
`page.json`), margem 24, espaçamento 16, toda posição derivada por
aritmética:

```
y=24   faixa (título + segmentações)          h=48
y=88   linha de KPIs — 4 × 296w               h=100   x = 24 / 336 / 648 / 960
y=204  linha analítica — 2 × 608w             h=238   x = 24 / 648
y=458  linha de detalhe — 1 × 1232w           h=238   x = 24
                                              (458 + 238 + 24 = 720 ✓)
```

Segmentações universais, sincronizadas nas 3 páginas: `dim_selecoes[team_name]`
e `dim_etapas[stage_name]`. `dim_estadios` e `dim_arbitros` só alcançam
`vw_selecao_partida` — podem segmentar as páginas 1 e 2, nunca a 3.

### Página 1 — Visão Geral do Torneio (forma: `summary`)

- KPIs: Total de Partidas · Gols Marcados · Média de Gols/Partida (vs.
  referência do torneio) · Eficiência Ofensiva (Gols/xG) (vs. referência do
  torneio)
- Gols por Fase — coluna + linha de média, ordenado pela ordem do torneio
  (fase de grupos → final), não descendente
- Mapa das sedes — bolha por cidade/país, tamanho = Total de Partidas
  (geocodificação por nome, sem coordenada fixa)
- Tabela Top 10 seleções — Pontos Conquistados, Saldo, Gols, Aproveitamento
  (%) (barras de dados na coluna de Pontos Conquistados)

### Página 2 — Perfil da Seleção (forma: `exploration`)

- KPIs: Pontos Conquistados · Aproveitamento (%) (vs. referência) · Saldo de
  Gols · Eficiência Ofensiva (Gols/xG) (vs. referência)
- Gols × xG por partida — combo coluna + linha
- Dispersão xG a favor × xG contra, por partida
- Tabela de confrontos: Adversário · Fase · Placar · xG · Resultado

### Página 3 — Análises Avançadas (forma: `narrative`)

- KPIs: xPts · Pontos Acima do Esperado · Pontos Recuperados · Estabilidade
  da Escalação (%)
- **Visual-âncora:** barras divergentes de "Pontos Acima do Esperado" por
  seleção (verde = acima, violeta = abaixo). Preferido a um scatter com linha
  identidade porque o scatter nativo do Power BI só oferece linha de razão
  (não y=x) — exigiria visual customizado, evitado pela restrição de
  legibilidade didática.
- Dispersão secundária: xPts × Pontos reais
- Tabela de estabilidade de escalação por seleção, com nota de rodapé sobre a
  convenção de pênaltis em `points`/`result` (achado 1 acima)

## Validação

1. **Estrutural** — `pbir` valida a cada mutação; nenhum visual sem vínculo
   de campo, nenhum campo inexistente no modelo. Agente
   `pbip-validator` roda ao final.
2. **Cor** — `grep -rE '#[0-9A-Fa-f]{6}' definition/pages/` deve retornar
   vazio. Teste objetivo de que toda cor vem do tema.
3. **Propagação de filtro** — escolher uma seleção na segmentação e confirmar
   que todo visual das 3 páginas reage; repetir com a segmentação de fase.
   Qualquer visual que não reagir está lendo `ft_partidas` diretamente e é
   bug (deveria ler as views).
4. **Números** — conferir os 4 valores de aceitação da seção "Achados de
   dados" (Total de Partidas = 100, Gols = 292, Média = 2,92, Eficiência = 1,100).
5. **Visual** — captura de tela das 3 páginas via Desktop Bridge (skill
   `pbir-cli`) e revisão humana; o JSON pode estar correto e a página ainda
   assim ilegível.
6. **Design gate** — checklist de fechamento da skill `pbi-report-design`
   (identidade propagada, uma intenção por página, espaçamento e margens
   iguais e na grade, chamadas apoiadas em dado do modelo, acessibilidade).
7. **Publicação** — commit e push para `fifa-world-cup-2026-bi` (branch
   `main`) só depois dos 6 portões acima passarem. O repositório é público;
   nada com credencial ou caminho de máquina local deve entrar em um
   `visual.json` ou `.pbi/localSettings.json` (já coberto pelo `.gitignore`
   existente).

## Riscos e decisões aceitas

- **Mapa por geocodificação online:** depende de conectividade no momento da
  renderização. Mitigação: renderizar e conferir antes da oficina; se falhar
  no dia, ter uma versão em imagem estática como plano B (fora de escopo
  deste documento, decisão de execução).
- **Pênaltis fora de `points`/`result`:** aceito como está, com nota de
  rodapé visível na página 3.
- **Genie não conhece as views analíticas:** diferença de escopo entre o que
  o Genie responde e o que o dashboard mostra, sem correção prevista aqui.
- **`dim_estadios` sem altitude/coordenada:** decisão do usuário — mapa por
  nome de cidade, sem corrigir a gold.

## Fora de escopo

- Publicação em workspace Fabric/Power BI Service (decisão do usuário:
  Desktop local).
- Página de Jogadores (artilheiros, assistências, cartões) — descartada em
  favor de 3 páginas focadas.
- Correção de `dim_estadios` para incluir latitude/longitude/elevação.
- Qualquer medida ou visual sobre `team_role` (mandante/visitante) — sedes
  neutras, o campo é só cadastral.
