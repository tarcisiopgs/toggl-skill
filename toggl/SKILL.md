---
name: toggl
description: Integração com as APIs do Toggl para controle e registro de horas — iniciar e parar timer, lançar tempo retroativo, somar horas por projeto ou cliente, gerar relatório de faturamento e submeter timesheet. Cobre os DOIS produtos, que são APIs distintas e incompatíveis entre si - Toggl Track (`api.track.toggl.com`, Basic auth) e Toggl 2.0 (`focus.toggl.com`, Bearer). Use sempre que aparecer "Toggl", `toggl_sk_`, `api_token`, `focus.toggl.com`, `api.track.toggl.com`, time entry, workspace_id, ou quando a tarefa envolver registrar horas trabalhadas, apontar tempo em projeto, fechar horas do mês, calcular quanto faturar por cliente ou descobrir onde o tempo foi parar — mesmo que o Toggl não seja nomeado. Use também antes de consultar a documentação do Toggl por WebFetch ou curl, e para decidir em qual dos dois produtos uma credencial funciona.
---

# Toggl

Existem **dois produtos Toggl com APIs separadas**, e eles não conversam. Errar qual deles você está usando é de longe a falha mais cara aqui, porque ela não se parece com o que é: você recebe 401 ou 403 e conclui que a credencial morreu, quando na verdade a credencial está viva e o host é que está errado.

Comece sempre por identificar em qual dos dois você está. O resto da skill depende disso.

## A bifurcação

|  | Toggl Track (clássico) | Toggl 2.0 (interno: "Focus") |
|---|---|---|
| Host | `https://api.track.toggl.com/api/v9` | `https://focus.toggl.com/api` |
| Autenticação | Basic `<token>:api_token` | `Authorization: Bearer toggl_sk_...` |
| Formato do token | 32 caracteres hexadecimais | `toggl_sk_` + 32 hex |
| Falha de auth | **403** | **401** |
| Limite | 1 req/s (leaky bucket), estoura em **429** | cota por hora, estoura em **402** |
| Documentação | `engineering.toggl.com/docs/track/` | `engineering.toggl.com/docs/focus/` |

**O prefixo do token diz o produto.** Se começa com `toggl_sk_`, é Toggl 2.0 e só funciona em `focus.toggl.com`. Se são 32 hex puros, é Track e só funciona em `api.track.toggl.com`. Cruzar os dois falha em todas as variações de auth que você tentar — Bearer, Basic, header customizado — e cada tentativa parece confirmar que o token é inválido.

Um teste de dois segundos resolve a dúvida antes de você gastar meia hora:

```bash
# Toggl 2.0
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  https://focus.toggl.com/api/users/me/settings

# Toggl Track
curl -s -o /dev/null -w '%{http_code}\n' -u "${TOKEN}:api_token" \
  https://api.track.toggl.com/api/v9/me
```

`200` em um dos dois responde a pergunta. Repare nas chaves em `${TOKEN}:api_token` — em zsh, `$TOKEN:a` aciona o modificador de expansão `:a` e destrói o valor silenciosamente, o que produz um 403 que parece problema de credencial.

## Antes de responder: consulte a documentação real

As duas APIs mudam e sua memória sobre elas envelhece — especialmente sobre o Toggl 2.0, que é recente e ainda está se mexendo. Não responda de memória sobre payloads, campos ou nomes de endpoint.

A especificação OpenAPI completa de cada produto está publicada e é barata de baixar. Os índices ficam em `engineering.toggl.com/docs/track/openapi/` e `engineering.toggl.com/docs/focus/openapi/`; cada página aponta para um `.json` versionado por hash. Baixe a spec e consulte-a localmente em vez de adivinhar caminhos — a do Toggl 2.0 tem 277 rotas e cerca de 400 operações, muito além do que cabe em contexto ou em memória.

A spec do Toggl 2.0 é **Swagger 2.0**, não OpenAPI 3: os schemas ficam em `definitions` (não `components.schemas`) e o corpo da requisição é um parâmetro com `in: body` (não `requestBody`). Um leitor escrito para OpenAPI 3 estoura com `KeyError` nela.

## Credenciais

Guarde o token num gerenciador de segredos e leia de lá na hora do uso, em vez de deixá-lo em `.env` versionado ou no histórico do shell. Com 1Password:

```bash
TOKEN=$(op read "op://<vault>/<item>/<seção>/<campo>")
```

Duas coisas específicas do Toggl que valem saber antes de mexer:

**No Toggl 2.0 existe apenas uma API key ativa por usuário.** Criar uma nova revoga a anterior em silêncio, e a key só é exibida uma vez, no momento da criação, em `focus.toggl.com/settings`. Não há como manter duas em paralelo, então "rotacionar para testar" quebra toda integração que já usava a antiga. Planeje a troca antes de gerar.

**No Track a rotação é igualmente destrutiva**: regenerar o token em `track.toggl.com/profile` invalida o anterior imediatamente.

Se o nome da seção contiver um ponto — "Toggl 2.0" é o caso óbvio — escape ao gravar com o CLI do 1Password, porque a sintaxe é `[<seção>.]<campo>[[tipo]]=<valor>` e o ponto do "2.0" vira um segundo separador:

```bash
op item edit "<item>" 'Toggl 2\.0.API Key[password]=toggl_sk_...'
```

## Descobrir os IDs

Praticamente toda rota exige IDs que não estão no token. Descubra-os uma vez e guarde junto da credencial em vez de redescobrir a cada execução.

No **Track**, `GET /api/v9/me` devolve `id` (usuário) e `default_workspace_id`; `GET /api/v9/me/organizations` devolve o `organization_id`.

No **Toggl 2.0**, `GET /api/users/me/settings` devolve `current_workspace_id`. O `organization_id` não é exposto de forma direta — se a conta também tiver Track ativo, o caminho mais curto é pegá-lo por lá, já que os IDs de organização e workspace são compartilhados entre os dois produtos.

Cuidado com uma pegadinha de identidade: o `toggl_user_id` que aparece nos payloads do Toggl 2.0 **não é** o mesmo número do `id` de usuário do Track. São espaços de identificadores diferentes; não use um no lugar do outro.

## Qual referência ler

Leia só a que corresponde à tarefa — cada uma é autossuficiente.

| Arquivo | Quando ler |
|---|---|
| `references/toggl-2.md` | Trabalhando em `focus.toggl.com`: rotas, timer, lançamentos, o modelo planejado-vs-realizado, cota e erros |
| `references/track-classico.md` | Trabalhando em `api.track.toggl.com/api/v9`: time entries, o campo `duration` negativo, rate limit |
| `references/relatorios.md` | Somar horas, faturar por cliente, exportar CSV — a engine de query do 2.0 e a Reports API v3 do Track |
| `references/registrar-horas.md` | Receitas prontas: iniciar/parar timer, lançar hora retroativa, fechar o mês, submeter timesheet |

## O erro que não estoura

Os dois produtos aceitam lançamento de tempo **sem projeto**. A requisição retorna 200, o tempo é contabilizado no total geral e some do relatório por projeto — que é justamente o relatório usado para faturar. Ninguém percebe até o fechamento do mês, quando as horas não batem.

Ao criar ou iniciar um lançamento, trate projeto ausente como um problema a levantar, não como um detalhe opcional. E ao somar horas para faturamento, confira quanto tempo caiu em "sem projeto" antes de entregar o número — se houver volume relevante ali, o total por projeto está incompleto e a conta vai sair menor do que deveria.
