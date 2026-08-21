# Receitas: registrar e fechar horas

Fluxos completos para as tarefas mais frequentes. Todos assumem que você já identificou o produto (veja o SKILL.md) e tem token e IDs em mãos.

Convenções usadas aqui:

```bash
TOKEN=$(op read "op://<vault>/<item>/<seção>/<campo>")
ORG=<organization_id>
WS=<workspace_id>
B2="https://focus.toggl.com/api/organizations/$ORG/workspaces/$WS"      # Toggl 2.0
BT="https://api.track.toggl.com/api/v9"                                 # Track
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

Ao inspecionar o resultado, olhe se há `project_id`. Timer rodando sem projeto está acumulando tempo que não vai aparecer em nenhuma fatura.

## Iniciar e parar o timer

```bash
# Toggl 2.0 — iniciar
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"description":"Ajuste no checkout","project_id":1234567}' \
  "$B2/tracking/start"

# Toggl 2.0 — parar
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$B2/tracking/stop"
```

```bash
# Track — iniciar (duration negativo marca "em execução")
curl -s -X POST -u "${TOKEN}:api_token" -H "Content-Type: application/json" \
  -d '{"created_with":"minha integração","description":"Ajuste no checkout","workspace_id":'"$WS"',"project_id":1234567,"duration":-1,"start":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","stop":null}' \
  "$BT/workspaces/$WS/time_entries"

# Track — parar
curl -s -X PATCH -u "${TOKEN}:api_token" \
  "$BT/workspaces/$WS/time_entries/<time_entry_id>/stop"
```

## Lançar tempo retroativo

O caso mais comum na prática: você trabalhou e esqueceu de ligar o timer.

```bash
# Toggl 2.0 — 2 horas ontem, das 14h às 16h (horário local convertido para UTC)
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "description": "Revisão de PR",
    "project_id": 1234567,
    "start": "2026-08-20T17:00:00Z",
    "duration": 7200,
    "billable": true
  }' "$B2/time-entries"
```

```bash
# Track — mesma coisa: start + duration positiva em segundos
curl -s -X POST -u "${TOKEN}:api_token" -H "Content-Type: application/json" \
  -d '{"created_with":"minha integração","description":"Revisão de PR","workspace_id":'"$WS"',"project_id":1234567,"start":"2026-08-20T17:00:00Z","duration":7200,"billable":true}' \
  "$BT/workspaces/$WS/time_entries"
```

Converta o horário local para UTC antes de montar o `start`. Errar isso desloca o lançamento para outro dia — e, perto da virada do mês, para outra fatura.

## Lançar várias horas de uma vez

Quando for reconstruir uma semana inteira, use o endpoint em lote do Toggl 2.0 (`POST ~/time-entries/bulk`) em vez de um laço de requisições. No Track não há criação em lote: respeite 1 req/s e insira um intervalo entre as chamadas, senão o 429 interrompe no meio e você fica com meia semana lançada.

## Quanto trabalhei em cada projeto neste mês

Use a via agregada — uma requisição, nomes já resolvidos. O corpo da consulta do Toggl 2.0 e a Summary do Track estão em `relatorios.md`.

Antes de entregar o número, três conferências que evitam faturar errado:

1. **Tempo sem projeto** — no Toggl 2.0 aparece como `project_id: 0` no relatório. Se houver volume relevante ali, o total por projeto está incompleto.
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
  | python3 -c "import json,sys; [print(p['id'], p['name']) for p in json.load(sys.stdin)]"

# Track
curl -s -u "${TOKEN}:api_token" "$BT/workspaces/$WS/projects" \
  | python3 -c "import json,sys; [print(p['id'], p['name']) for p in json.load(sys.stdin)]"
```

Se você vai fazer isso com frequência, guarde o mapa nome→id junto dos outros IDs em vez de listar projetos a cada execução — no Toggl 2.0 cada listagem consome parte de uma cota horária pequena.
