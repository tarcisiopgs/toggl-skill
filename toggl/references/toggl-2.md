# Toggl 2.0 (`focus.toggl.com`)

O produto novo do Toggl, chamado "Focus" internamente — o nome aparece na URL da documentação, nos schemas (`dto.QueryRequest`) e em mensagens de erro, mas nunca na interface. É uma API distinta do Track, não uma versão nova dela.

## Base e autenticação

```
https://focus.toggl.com/api
Authorization: Bearer toggl_sk_...
Content-Type: application/json
```

Quase toda rota fica sob `/organizations/{organization_id}/workspaces/{workspace_id}/`. As exceções, que dispensam o `organization_id`, são: relatórios (`/reports/workspaces/{ws}/...`), clientes, milestones, statuses, tags, saved-views, permissões, moeda e configurações de workspace.

Datas e horas são ISO 8601 / RFC 3339, armazenadas em UTC e devolvidas no fuso do perfil do usuário.

## O modelo de dados difere do Track

Esta é a diferença conceitual que mais confunde quem vem do Track: no Toggl 2.0 um lançamento carrega **duas dimensões**.

| Dimensão | Campos | Origem |
|---|---|---|
| Planejado | `planned_start`, `planned_duration`, `calendar_event_id` | evento do Google Calendar sincronizado |
| Realizado | `start`, `stop`, `duration` | tempo efetivamente rastreado |

Um compromisso de agenda entra como lançamento **planejado** e só ganha `start`/`stop` quando vira trabalho de fato. Na prática a maioria dos registros de uma conta com calendário sincronizado nunca é executada.

Isso importa porque somar tudo infla o total de horas — você acaba faturando reunião que não aconteceu. **Para "quanto foi realmente trabalhado", filtre por `start` presente.** Se o que você quer é o inverso — comparar previsto contra realizado — as duas dimensões no mesmo registro são exatamente o que torna isso possível, e é o motivo de o modelo ser assim.

Os campos variam entre registros do mesmo retorno: um lançamento planejado simplesmente não traz `start`. Código que assume a presença dos campos do primeiro item quebra no meio da lista.

## Rotas essenciais

Abaixo, `~` abrevia `/organizations/{org}/workspaces/{ws}`.

### Timer

```
GET   ~/tracking/current                    lançamento em execução
POST  ~/tracking/start                      inicia
POST  ~/tracking/start-from-description     inicia a partir de um texto
POST  ~/tracking/stop                       encerra
```

### Lançamentos

```
GET    ~/time-entries                    lista (date_from e date_to obrigatórios)
POST   ~/time-entries                    cria lançamento sem task
POST   ~/tasks/{task_id}/time-entries    cria lançamento vinculado a uma task
PATCH  ~/time-entries/{id}               atualização parcial
DELETE ~/time-entries/{id}
POST   ~/time-entries/bulk               criação em lote
PATCH  ~/time-entries/bulk-edit          edição em lote
GET    ~/time-entries/stream             lista via streaming, para volumes grandes
```

### Projetos, tasks e apoio

```
GET   ~/projects            GET ~/projects/{id}       POST ~/projects
GET   ~/tasks               POST ~/tasks
GET   /workspaces/{ws}/clients        GET /workspaces/{ws}/tags
GET   /workspaces/{ws}/context        panorama do workspace numa chamada
GET   /workspaces/{ws}/permissions    o que o usuário atual pode fazer aqui
```

### Timesheets (aprovação de horas)

Ciclo completo de submissão e aprovação, útil quando as horas passam por revisão antes de virar fatura:

```
GET   ~/timesheets
POST  ~/timesheets/{setup_id}/{start_date}/{user_account_id}/submit
POST  ~/timesheets/{setup_id}/{start_date}/{user_account_id}/approve
POST  ~/timesheets/{setup_id}/{start_date}/{user_account_id}/request-changes
POST  ~/timesheets/{setup_id}/{start_date}/{user_account_id}/withdraw
GET   ~/timesheets/{setup_id}/{start_date}/{user_account_id}/hours
POST  ~/timesheets/bulk/approve
```

Há ainda faixas de custo e faturamento por projeto e por pessoa (`~/billable-rates/...`, `~/labor-costs/...`), que alimentam o relatório de rentabilidade.

## Cota: falha com 402, não 429

O limite é **por hora, por usuário, por organização**, e é definido pelo plano da organização:

| Plano | Requisições/hora |
|---|---|
| Free | 30 |
| Starter | 240 |
| Premium | 600 |

Ao estourar, a resposta é **HTTP 402 (Payment Required)**, não 429. A escolha é deliberada — a cota está atrelada à assinatura, então o Toggl sinaliza "seu plano acabou" em vez de "desacelere". Na prática isso engana: parece problema de cobrança e não de volume de chamadas.

Dois headers permitem se antecipar, quando presentes:

```
X-Toggl-Quota-Remaining     requisições restantes na janela
X-Toggl-Quota-Resets-In     segundos até a janela reiniciar
```

Trinta requisições por hora é pouco. Evite varrer endpoints por exploração: baixe a spec OpenAPI e leia-a localmente, que isso não consome cota. Prefira uma consulta de relatório agregada a paginar lançamentos crus.

## Armadilhas de requisição

**Datas em query param exigem RFC 3339 completo.** `date_from=2026-08-07` devolve 400 com `parsing time "2026-08-07" as "2006-01-02T15:04:05Z07:00"`. Use `2026-08-07T00:00:00Z`. Curiosamente, no *corpo* de uma consulta de relatório o campo `period.from` aceita `YYYY-MM-DD` — a regra não é uniforme, então na dúvida mande RFC 3339 completo.

**`per_page` tem teto.** 200 é recusado; 100 passa.

**Respostas de lista vêm embrulhadas.** O formato é `{"page": 1, "per_page": 100, "data": [...]}` — os registros estão em `data`, não na raiz.

**Os erros são do validador Go e dizem exatamente o que corrigir**, o que torna o debug quase determinístico:

```json
{
  "error": "validation",
  "error_description": "Key: 'Pagination.PerPage' Error:Field validation for 'PerPage' failed on the 'max' tag",
  "trace_id": "2cd9a01304b5d6362f6c282dfd21914f"
}
```

Leia o campo e a tag violada antes de tentar de novo — vale mais que qualquer chute, e cada tentativa cega custa cota. O `trace_id` é o que o suporte do Toggl pede.

## Códigos de resposta

| Código | Significado |
|---|---|
| 400 | payload ou query inválidos — a mensagem nomeia o campo |
| 401 | token inválido **ou host errado** (token do Track aqui) |
| 402 | cota horária estourada |
| 403 | o plano não tem direito ao recurso (exportação de relatório, por exemplo) |
| 422 | conversão de moeda pedida sem taxa de câmbio disponível |
