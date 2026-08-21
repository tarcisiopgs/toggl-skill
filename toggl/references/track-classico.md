# Toggl Track (`api.track.toggl.com/api/v9`)

A API tradicional do Toggl. Estável, amplamente documentada e ainda ativa — contas antigas seguem funcionando aqui mesmo depois de a organização passar a usar o Toggl 2.0.

## Base e autenticação

```
https://api.track.toggl.com/api/v9
```

HTTP Basic, com o token no lugar do usuário e a palavra literal `api_token` como senha:

```bash
curl -u "${TOKEN}:api_token" https://api.track.toggl.com/api/v9/me
```

A string `<token>:api_token` é codificada em Base64 pelo próprio curl. Autenticação por e-mail e senha também funciona (`-u email:senha`), mas token é preferível: não expira ao trocar a senha e pode ser revogado isoladamente.

Falha de autenticação aqui é **403**, não 401. Se você recebeu 401, provavelmente está no host errado — 401 é a assinatura do Toggl 2.0.

Pegue o token em `track.toggl.com/profile`. Regenerar invalida o anterior na hora.

## O campo `duration` codifica o estado

Esta é a convenção mais idiossincrática do Track, e a fonte de bug mais comum para quem chega novo.

- **Lançamento encerrado**: `duration` é a duração em segundos. `duration: 120` são dois minutos.
- **Lançamento em execução**: `duration` é **negativo**. O valor convencional ao criar é `-1`.

Ou seja, o mesmo campo carrega duração e estado. Somar `duration` ingenuamente numa lista que contém um timer rodando subtrai tempo do total — e o resultado continua sendo um número plausível, então passa despercebido.

Ao agregar horas, filtre `duration > 0` ou trate o lançamento em execução separadamente.

```bash
# inicia um lançamento agora
curl -u "${TOKEN}:api_token" -H "Content-Type: application/json" \
  -d '{"created_with":"minha integração","description":"Refatoração","workspace_id":'"$WS"',"project_id":'"$PROJ"',"duration":-1,"start":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","stop":null}' \
  -X POST "https://api.track.toggl.com/api/v9/workspaces/$WS/time_entries"
```

O `workspace_id` precisa aparecer **nos dois lugares**: no caminho da URL e no corpo. Omitir do corpo é rejeitado.

O campo `created_with` é obrigatório na criação e identifica sua integração nos logs do Toggl. Use um nome reconhecível.

## Rotas essenciais

```
GET    /me                                   perfil, id, default_workspace_id
GET    /me/organizations                     organizações e seus ids
GET    /me/time_entries                      lançamentos (start_date / end_date)
GET    /me/time_entries/current              lançamento em execução
GET    /workspaces/{ws}/projects
GET    /workspaces/{ws}/clients
GET    /workspaces/{ws}/tags
POST   /workspaces/{ws}/time_entries
PUT    /workspaces/{ws}/time_entries/{id}
PATCH  /workspaces/{ws}/time_entries/{id}/stop
DELETE /workspaces/{ws}/time_entries/{id}
```

Repare na diferença de grafia entre os produtos: aqui é `time_entries` com underscore; no Toggl 2.0 é `time-entries` com hífen. Copiar um caminho de um para o outro dá 404.

## Rate limit: 1 req/s, falha com 429

O Track usa *leaky bucket*. A janela segura documentada é **1 requisição por segundo**, aplicada por token **e** por IP — dois usuários no mesmo IP têm cotas separadas.

Ao estourar vem **429**; a orientação oficial é recuar alguns minutos antes de tentar de novo. Para operações em lote, insira um intervalo entre chamadas em vez de disparar em paralelo.

Compare com o Toggl 2.0, onde o limite é por hora e falha com 402 — os dois produtos falham de formas diferentes sob carga, e um retry pensado para um está errado para o outro.

## Outros códigos que a documentação destaca

| Código | O que fazer |
|---|---|
| 402 | o workspace precisa de upgrade para esse recurso — não repita até isso mudar |
| 410 | endpoint removido — não tente de novo |
| 429 | recue alguns minutos, depois volte a 1 req/s |

## Consistência eventual

O Track avisa na documentação que a leitura é eventualmente consistente: um recurso recém-criado pode não aparecer imediatamente numa listagem. Não trate a ausência logo após um `POST` como falha da escrita — se a criação devolveu 200 e um `id`, o registro existe.

## Relatórios

Ficam numa API à parte, sob `/reports/api/v3`, com suas próprias regras. Veja `relatorios.md`.
