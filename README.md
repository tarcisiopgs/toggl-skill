# toggl-skill

Agent skill for the two [Toggl](https://toggl.com) time-tracking APIs — Toggl Track and Toggl 2.0. Written in Portuguese, matching the rest of these skills.

Skill para agentes de IA sobre as APIs do Toggl — registrar horas, iniciar e parar timer, somar tempo por projeto, gerar relatório de faturamento e submeter timesheet.

## Instalação

```bash
npx skills add tarcisiopgs/toggl-skill@toggl -g -y
```

O `-g` instala globalmente (nível de usuário), disponível em todos os agentes compatíveis — Claude Code, Cursor, Codex, Gemini CLI, entre outros.

Para instalar apenas no projeto atual, remova o `-g`.

## O problema que a skill resolve

Existem **dois produtos Toggl com APIs separadas e incompatíveis**, e a documentação deles vive lado a lado sem deixar isso evidente:

| | Toggl Track (clássico) | Toggl 2.0 ("Focus") |
|---|---|---|
| Host | `api.track.toggl.com/api/v9` | `focus.toggl.com/api` |
| Autenticação | Basic `<token>:api_token` | `Authorization: Bearer toggl_sk_...` |
| Falha de auth | 403 | 401 |
| Limite | 1 req/s → 429 | cota por hora → **402** |

Cruzar host com token falha em todas as variações de autenticação que você tentar, e cada tentativa reforça a conclusão errada de que a credencial morreu. A skill começa justamente por aí: identificar o produto antes de qualquer outra coisa.

## O que a skill cobre

| Tema | Conteúdo |
|---|---|
| **A bifurcação** | Como saber em qual produto você está pelo prefixo do token e pelo código de erro |
| **Credenciais** | Leitura via gerenciador de segredos; a key única por usuário do Toggl 2.0, que se auto-revoga |
| **IDs** | Onde achar organização, workspace e usuário — e por que o `toggl_user_id` do 2.0 não é o `id` do Track |
| **Toggl 2.0** | Rotas de timer, lançamentos, projetos e timesheets; o modelo planejado-vs-realizado; cota que falha com 402 |
| **Track clássico** | O campo `duration` negativo que codifica estado; rate limit por token e IP; consistência eventual |
| **Relatórios** | Engine de query do 2.0 e Reports API v3 do Track; paginação por cursor; exportação por header |
| **Receitas** | Iniciar/parar timer, lançar hora retroativa, fechar o mês, submeter timesheet |

O foco é o que a documentação não deixa óbvio à primeira leitura: as convenções que divergem entre os dois produtos e os erros que não estouram — passam, e aparecem depois como hora que sumiu da fatura do cliente.

## Estrutura

```
toggl/
├── SKILL.md
└── references/
    ├── toggl-2.md
    ├── track-classico.md
    ├── relatorios.md
    └── registrar-horas.md
```

O `SKILL.md` carrega o essencial e a decisão de qual produto usar; as referências são lidas sob demanda conforme o tema da tarefa.

## Fontes

Todo o conteúdo é derivado da documentação oficial em [engineering.toggl.com](https://engineering.toggl.com) e das especificações OpenAPI publicadas pelo Toggl, verificado contra as APIs reais em agosto de 2026. Este é um projeto independente, sem vínculo com o Toggl.

Encontrou algo desatualizado ou incorreto? Abra uma issue ou um PR.

## Licença

MIT
