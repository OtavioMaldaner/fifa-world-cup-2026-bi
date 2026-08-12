# Do Campo Aos Dados: Dashboard Power BI — Copa do Mundo 2026

Relatório Power BI da oficina "Do Campo Aos Dados", etapa final da jornada dos dados: consome a camada gold preparada no workspace Databricks do repositório irmão [`oficina-semana-informatica-2026`](https://github.com/OtavioMaldaner/oficina-semana-informatica-2026) e transforma dimensões, fatos e views analíticas em um dashboard interativo com identidade visual de transmissão esportiva.

## Estrutura do repositório

```
fifa-world-cup-2026.pbip              Arquivo de projeto Power BI (PBIP)
fifa-world-cup-2026.Report/           Definição do relatório: páginas, visuais, tema
fifa-world-cup-2026.SemanticModel/    Modelo semântico: tabelas, relacionamentos, medidas DAX
```

## Modelo semântico

Modo de conexão **Import**, via conector `DatabricksMultiCloud`, apontando para o mesmo SQL Warehouse do workspace Databricks da oficina. As 14 tabelas do modelo espelham a camada gold do Databricks:

- **Dimensões**: `dim_selecoes`, `dim_estadios`, `dim_etapas`, `dim_jogadores`, `dim_arbitros`
- **Fatos**: `ft_partidas`, `ft_eventos`, `ft_estatisticas_equipe`, `ft_estatisticas_jogador`, `ft_escalacoes`
- **Views analíticas**: `vw_selecao_partida`, `vw_pontos_recuperados`, `vw_estabilidade_escalacao`, `vw_xpts_selecao_partida`

(ver o [repositório do Databricks](https://github.com/OtavioMaldaner/oficina-semana-informatica-2026) para o detalhamento de cada tabela)

Uma tabela auxiliar `_Measures` concentra as medidas DAX do relatório — métricas de seleção (gols, saldo, aproveitamento, xG, xPts, pontos recuperados, estabilidade de escalação), métricas agregadas do torneio e métricas de jogador (artilharia, assistências, cartões, nota média, além dos destaques "Artilheiro do Torneio", "Garçom do Torneio" e "Melhor Goleiro").

## Páginas do relatório

| Página | Conteúdo |
|---|---|
| **Visão Geral do Torneio** | KPIs gerais (partidas, gols marcados, média de gols, eficiência), gráfico de gols por fase, mapa das sedes, tabela top 10 e filtros por seleção/fase |
| **Perfil da Seleção** | KPIs por seleção (pontos, aproveitamento, saldo, eficiência), tabela de confrontos e dispersão de gols x xG por partida |
| **Análises Avançadas** | Métricas avançadas: pontos esperados (xPts), pontos recuperados e estabilidade de escalação |
| **Jogadores** | Artilharia, melhores notas, ranking de goleiros, tabela de jogadores com filtro por posição/seleção, e cards de destaque (artilheiro, garçom, melhor goleiro) |

## Identidade visual

O relatório usa um tema customizado com paleta inspirada em transmissões esportivas, registrado em `fifa-world-cup-2026.Report/StaticResources/RegisteredResources/IFRS_Feliz.json` sobre a base `CY26SU05`.

## Pré-requisitos

- **Power BI Desktop** (Windows) — [download](https://www.microsoft.com/en-us/power-platform/products/power-bi/desktop)
- Camada gold já publicada no workspace Databricks da oficina — ver [`oficina-semana-informatica-2026`](https://github.com/OtavioMaldaner/oficina-semana-informatica-2026)

## Como abrir

1. Prepare a camada gold no Databricks seguindo o repositório [`oficina-semana-informatica-2026`](https://github.com/OtavioMaldaner/oficina-semana-informatica-2026).
2. Abra `fifa-world-cup-2026.pbip` no Power BI Desktop.
3. Na primeira abertura, o Power BI pede autenticação no SQL Warehouse do Databricks — use suas credenciais do workspace.
4. As tabelas são carregadas em modo Import; para atualizar os dados, use **Atualizar** no Power BI Desktop.

## Referências

- [Download do Power BI Desktop](https://www.microsoft.com/en-us/power-platform/products/power-bi/desktop)
- [Power BI Project (PBIP)](https://learn.microsoft.com/power-bi/developer/projects/projects-overview)