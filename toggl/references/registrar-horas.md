# Receitas: registrar e fechar horas

Fluxos completos para as tarefas mais frequentes. Todos assumem que você já identificou o produto (veja o SKILL.md) e tem token e IDs em mãos.

Convenções usadas aqui: `TOKEN` e `WS` devem vir da configuração do usuário; `ORG` só é necessário no Toggl 2.0 (Focus). Nos exemplos, IDs, descrições e datas são ilustrativos; substitua-os antes de executar. Escolha `BILLABLE` explicitamente como `true` ou `false`, conforme a tarefa ou preferência já configurada. Não presuma faturamento.

```bash
: "${TOKEN:?Configure o token pelo mecanismo de credenciais escolhido}"
: "${WS:?Configure o ID do workspace}"
BT="https://api.track.toggl.com/api/v9"                                 # Track
```

Execute também este bloco apenas para Toggl 2.0 (Focus):

```bash
: "${ORG:?Configure o ID da organização para Toggl 2.0 (Focus)}"
B2="https://focus.toggl.com/api/organizations/$ORG/workspaces/$WS"
```

## Ver o que está rodando agora

Pergunta mais útil do que parece: timer esquecido rodando a noite toda é a origem clássica de lançamento absurdo no fechamento.

```bash
# Toggl 2.0
curl -s -H "Authorization: Bearer $TOKEN" "$B2/tracking/current"

# Track
curl -s -u "${TOKEN}:api_token" "$BT/me/time_entries/current"
```

No Toggl 2.0 o retorno vazio significa nada rodando. No Track, verifique se `duration` é negativo — é essa a marca de "em execução".

A marca equivalente no Toggl 2.0 é o **`duration` nulo**: um lançamento em curso
tem `start` preenchido e `duration: null`; um já encerrado traz o número de
segundos. Não procure um campo de fim — o modelo é `start` + `duration` e não
existe `stop` no payload. Quem vem do Track tende a mandar um `stop` no corpo,
que é ignorado em silêncio, e depois lê a resposta achando que o lançamento
ficou aberto.

Se a tarefa envolve faturamento por projeto, confira se há `project_id` e sinalize tempo sem projeto para revisão. Fora desse contexto, respeite o uso intencional sem projeto.

## Iniciar e parar o timer

Os dois produtos aceitam `billable` no início do timer. No Focus, isso está definido em `timeentry.StartTrackingPayload` na [OpenAPI oficial](https://engineering.toggl.com/docs/focus/openapi/). Defina `BILLABLE` antes de iniciar; os parênteses limitam a validação à receita, preservando o shell se o valor for inválido.

```bash
# Toggl 2.0 — iniciar. `type` é obrigatório: "activity" ou "break".
(
case "${BILLABLE:-}" in true|false) ;; *) echo "Configure BILLABLE como true ou false" >&2; exit 1 ;; esac
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"type":"activity","description":"Ajuste no checkout","project_id":1234567,"billable":'"$BILLABLE"'}' \
  "$B2/tracking/start"
)
```

Execute a parada somente quando quiser encerrar o timer:

```bash
# Toggl 2.0 — parar. O corpo é obrigatório e leva o instante do fim em `end`.
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"end":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"}' \
  "$B2/tracking/stop"
```

```bash
# Track — iniciar (duration negativo marca "em execução")
(
case "${BILLABLE:-}" in true|false) ;; *) echo "Configure BILLABLE como true ou false" >&2; exit 1 ;; esac
curl -s -X POST -u "${TOKEN}:api_token" -H "Content-Type: application/json" \
  -d '{"created_with":"minha integração","description":"Ajuste no checkout","workspace_id":'"$WS"',"project_id":1234567,"duration":-1,"start":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","stop":null,"billable":'"$BILLABLE"'}' \
  "$BT/workspaces/$WS/time_entries"
)
```

Para encerrar um lançamento identificado:

```bash
# Track — parar
curl -s -X PATCH -u "${TOKEN}:api_token" \
  "$BT/workspaces/$WS/time_entries/<time_entry_id>/stop"
```

## Lançar tempo retroativo

O caso mais comum na prática: você trabalhou e esqueceu de ligar o timer.

```bash
# Toggl 2.0 — exemplo de 2 horas em uma data ilustrativa, em UTC.
# Um lançamento é `start` + `duration`; não existe campo de fim aqui.
# Defina BILLABLE como true ou false antes de executar.
(
case "${BILLABLE:-}" in true|false) ;; *) echo "Configure BILLABLE como true ou false" >&2; exit 1 ;; esac
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "type": "activity",
    "description": "Revisão de PR",
    "project_id": 1234567,
    "start": "2026-08-20T17:00:00Z",
    "duration": 7200,
    "billable": '"$BILLABLE"'
  }' "$B2/time-entries"
)
```

O `type` é obrigatório nos dois casos, e omiti-lo devolve um 400 cuja mensagem
aponta para um schema interno em vez do campo — `CreateTasklessPayload.Payload.
PayloadWithoutDuration.Type`, para o POST de lançamento. Lendo rápido, parece
problema de payload malformado; é só o `type` faltando.

```bash
# Track — mesma coisa: start + duration positiva em segundos
(
case "${BILLABLE:-}" in true|false) ;; *) echo "Configure BILLABLE como true ou false" >&2; exit 1 ;; esac
curl -s -X POST -u "${TOKEN}:api_token" -H "Content-Type: application/json" \
  -d '{"created_with":"minha integração","description":"Revisão de PR","workspace_id":'"$WS"',"project_id":1234567,"start":"2026-08-20T17:00:00Z","duration":7200,"billable":'"$BILLABLE"'}' \
  "$BT/workspaces/$WS/time_entries"
)
```

Use o fuso escolhido pelo usuário para converter o horário local para UTC antes de montar o `start`. Errar isso desloca o lançamento para outro dia — e, perto da virada do mês, para outra fatura.

## Lançar várias horas de uma vez

Quando for reconstruir uma semana inteira, use o endpoint em lote do Toggl 2.0 (`POST ~/time-entries/bulk`) em vez de um laço de requisições. No Track não há criação em lote: respeite 1 req/s e insira um intervalo entre as chamadas, senão o 429 interrompe no meio e você fica com meia semana lançada.

## Quanto trabalhei em cada projeto neste mês

Use a via agregada — uma requisição, nomes já resolvidos. O corpo da consulta do Toggl 2.0 e a Summary do Track estão em `relatorios.md`.

Antes de entregar o número, confira estes pontos; a revisão para faturamento só se aplica quando esse for o objetivo:

1. **Tempo sem projeto** — no Toggl 2.0 aparece como `project_id: 0` no relatório. Apresente-o separadamente; no contexto de faturamento, confira se precisa ser atribuído a um projeto.
2. **Lançamento em execução** — no Track, `duration` negativo contamina somas ingênuas. Filtre ou trate à parte.
3. **Planejado versus realizado** — no Toggl 2.0, registros com `calendar_event_id` e sem `start` são compromissos de agenda que nunca viraram trabalho. Contá-los infla o total.

## Submeter horas para aprovação

Só no Toggl 2.0, e só se a organização tiver timesheets configurados:

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "$B2/timesheets/{setup_id}/{start_date}/{user_account_id}/submit"
```

`GET ~/timesheets` lista o que está visível para você e revela os `setup_id` disponíveis. Para conferir o total antes de submeter, `GET ~/timesheets/{setup_id}/{start_date}/{user_account_id}/hours`.

Submeter é reversível enquanto ninguém aprovou: `.../withdraw` retira a submissão. Depois de aprovado, a retirada depende de permissão de aprovador.

## Descobrir o id de um projeto pelo nome

Você quase sempre tem o nome e precisa do id.

```bash
# Toggl 2.0
curl -s -H "Authorization: Bearer $TOKEN" "$B2/projects" \
  | python3 -c "import json,sys; [print(p['id'], p['name']) for p in json.load(sys.stdin)['data']]"

# Track
curl -s -u "${TOKEN}:api_token" "$BT/workspaces/$WS/projects" \
  | python3 -c "import json,sys; [print(p['id'], p['name']) for p in json.load(sys.stdin)]"
```

O exemplo Focus lê uma página: a resposta contém `data`, `page`, `per_page` e `total`. Consulte as páginas restantes quando necessário antes de concluir que um projeto não existe. Formato conferido na [OpenAPI oficial do Focus](https://engineering.toggl.com/docs/focus/openapi/), em setembro de 2026.

Se você vai fazer isso com frequência, guarde o mapa nome→id junto dos outros IDs em vez de listar projetos a cada execução — no Toggl 2.0 cada listagem consome parte de uma cota horária pequena.
