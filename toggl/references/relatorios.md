# Relatórios e faturamento

Somar horas é a pergunta que mais chega ao Toggl, e nos dois produtos existe um caminho agregado que resolve isso melhor do que paginar lançamentos crus e somar na mão. Prefira sempre a via agregada: ela é uma requisição em vez de dezenas, resolve nomes de projeto para você e não exige que você reimplemente a semântica de duração.

## Toggl 2.0: engine de query

```
POST /reports/workspaces/{workspace_id}/query
```

Note que esta rota **não** leva `organization_id`. Aceita `?response_format=json_row|table|hierarchical` e `?include_dicts=true`.

O corpo descreve uma consulta analítica — filtros, agregações, agrupamentos, ordenações, transformações, moeda e fuso. Todos os campos de primeiro nível são obrigatórios na estrutura, mesmo quando vazios; omitir um array resulta em 400.

Horas por projeto num período. Datas e IDs abaixo são ilustrativos. Substitua `currency` pela moeda escolhida para o relatório; `USD` é apenas um exemplo. Configure o fuso do período conforme o usuário. Use `view_in_my_timezone: true` somente quando o fuso do perfil corresponder ao desejado; confira a semântica vigente na documentação antes de usar outro recorte.

```json
{
  "period": {"from": "2026-08-01", "to": "2026-08-31", "preset": ""},
  "filters": [],
  "aggregations": [{"function": "sum", "property": "duration"}],
  "groupings": [{"property": "project_id", "show_empty": false}],
  "aggregation_filters": [],
  "attributes": [],
  "ordinations": [],
  "transformations": [],
  "pagination": {"page": 1, "per_page": 50},
  "limit": 100,
  "currency": "USD",
  "conversion_date": null,
  "modifiers": {},
  "view_in_my_timezone": true
}
```

A resposta em formato `table` vem como matriz, com a primeira linha servindo de cabeçalho:

```json
{
  "data_table": [
    ["sum_duration", "project_id"],
    [4954, 1234567],
    [27209, 7654321]
  ],
  "dictionaries": {
    "projects": {"1234567": {"name": "Nome do Projeto", "color": "#e36a00"}},
    "tasks": {}, "status": {}, "users": {}
  }
}
```

Com `include_dicts=true` você recebe `dictionaries` para traduzir os IDs em nomes sem chamadas adicionais — é o que evita um N+1 de `GET /projects/{id}`. Durações vêm em **segundos**; divida por 3600 para horas.

Repare na linha com `project_id: 0` quando ela aparecer: é o tempo **sem projeto**. Reporte esse tempo separadamente quando relevante. Ele pode ser intencional; no contexto de faturamento por projeto, confira sua atribuição antes de fechar os valores.

Há também `POST /reports/workspaces/{ws}/profitability` (rentabilidade com projeção) e `POST /reports/workspaces/{ws}/workload` (carga de trabalho).

**Exportar arquivo** é um header, não um parâmetro: `X-Toggl-Report-Export: csv|xlsx|pdf`, com o mesmo corpo JSON. Cada formato depende de um direito do plano — sem ele a resposta é 403, não um arquivo vazio.

## Toggl Track: Reports API v3

API separada, sob a mesma base `https://api.track.toggl.com`, em `/reports/api/v3`. Autenticação idêntica à do Track (Basic com `:api_token`).

Cinco tipos de relatório:

| Tipo | Serve para |
|---|---|
| Summary | total agregado por categoria — o caminho curto para faturamento |
| Detailed | cada lançamento individual, ideal para exportar e revisar |
| Weekly | blocos de 7 dias agrupados por pessoa e projeto, em duração ou valor |
| Saved | relatórios salvos e compartilháveis, inclusive com quem não tem conta — indisponível no plano Free |
| Insights | rentabilidade de projetos e de pessoas |

**Relatórios não cruzam workspaces.** É preciso especificar um, e não existe consulta consolidada entre eles. Com vários workspaces, some no seu lado.

**Detailed é paginado de 50 em 50**, e a paginação é por cursor via headers de resposta:

```
X-Next-ID           id do próximo lançamento
X-Next-Row-Number   número da próxima linha
```

Passe o valor de `X-Next-Row-Number` como `first_row_number` na requisição seguinte. Como não há um total antecipado, o fim da coleção é sinalizado pela ausência desses headers — pare por aí, não por contagem estimada.

Relatórios têm visibilidade pública ou privada; acessar um privado sem ser dono ou admin do workspace devolve 403.

A V2 continua suportada e sem prazo de migração anunciado. Encontrar código em V2 não é motivo suficiente para reescrevê-lo.

## Escolhendo o período

Use o fuso escolhido pelo usuário para definir as bordas do período; não assuma um país ou um offset fixo. Um lançamento perto da meia-noite pode pertencer a dias ou meses diferentes em UTC e no fuso local. No Toggl 2.0, confira se o fuso do perfil usado por `view_in_my_timezone` corresponde ao recorte solicitado. No Track, monte o intervalo a partir das bordas locais convertidas, considerando mudanças de horário de verão quando aplicáveis.
