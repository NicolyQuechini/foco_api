# Foco API

API REST de um gerenciador de estudos. Permite **cadastrar disciplinas, atividades e avaliações** e **acompanhar o progresso**.
Feita com Node.js, Express e SQLite (o SQLite que já vem dentro do Node, sem nada para compilar).

| Requisito | Onde está |
|---|---|
| Cadastrar disciplinas | `/disciplinas` |
| Cadastrar atividades | `/atividades` |
| Cadastrar avaliações (provas, trabalhos, notas) | `/avaliacoes` |
| Acompanhar o progresso | `/progresso`, `/dashboard`, `/relatorios` |
| Registrar horas de estudo | `/sessoes`, `/metas` |

## Como rodar

1. Instale o [Node.js](https://nodejs.org) na versão **22.13 ou mais nova** (a LTS serve). Confira com `node -v`.
2. Na pasta do projeto, instale as dependências:
   ```
   npm install
   ```
3. Crie o arquivo `.env` copiando o `.env.example` e troque o `JWT_SECRET` por um texto grande qualquer.
4. Inicie a API:
   ```
   npm start
   ```
   Ela fica em `http://localhost:3000` (abre a documentação). O banco (`foco.db`) é criado sozinho na primeira execução.
   (Se existir um `foco.db` de uma versão antiga, apague-o antes.)
5. Para conferir se está tudo funcionando:
   ```
   npm run teste
   ```
   O teste liga a API sozinho (com um banco temporário), confere o fluxo principal e desliga. Não precisa do `npm start` nem do `.env`.

## Documentação interativa (Swagger)

Com a API rodando (`npm start`), abra no navegador:

```
http://localhost:3000/docs
```

A página lista as 28 rotas por assunto, mostra exemplos de envio e de resposta e permite **testar cada rota ali mesmo** (*Try it out*). A especificação em formato OpenAPI também está em `http://localhost:3000/openapi.json` (pode ser importada no Postman ou no Insomnia).

Para testar as rotas protegidas na página: crie a conta em `POST /auth/registro`, faça `POST /auth/login`, copie o `token`, clique em **Authorize** (cadeado no topo) e cole o token (sem a palavra *Bearer*).

## Como usar

1. Crie uma conta em `POST /auth/registro` e entre em `POST /auth/login`.
2. O login devolve um `token`. Nas demais rotas, envie o header:
   `Authorization: Bearer SEU_TOKEN`

## Rotas

| Método | Rota | O que faz | Auth |
|---|---|---|---|
| POST | `/auth/registro` | Cria usuário (nome, email, senha com 8+ caracteres) | não |
| POST | `/auth/login` | Devolve token JWT | não |
| GET | `/usuarios/me` | Perfil | sim |
| PUT | `/usuarios/me` | Edita nome e/ou e-mail | sim |
| GET | `/disciplinas` | Lista com minutos estudados e contagens | sim |
| POST | `/disciplinas` | Cadastra (`nome`, `cor`, `meta_horas` opcional) | sim |
| GET | `/disciplinas/:id` | Detalhe com atividades, avaliações e últimas sessões | sim |
| PUT | `/disciplinas/:id` | Edita | sim |
| DELETE | `/disciplinas/:id` | Apaga (e suas atividades, avaliações e sessões) | sim |
| GET | `/atividades?status=pendentes ou feitas&disciplina_id=` | Lista com filtros | sim |
| POST | `/atividades` | Cadastra (`titulo`, `disciplina_id`, `prazo` AAAA-MM-DD, `prioridade` baixa/media/alta, `descricao`) | sim |
| GET | `/atividades/:id` | Detalhe | sim |
| PATCH | `/atividades/:id` | Edita campos enviados (ex.: `{"concluida": true}`) | sim |
| DELETE | `/atividades/:id` | Apaga | sim |
| GET | `/avaliacoes?status=pendentes ou realizadas&disciplina_id=` | Lista por data | sim |
| POST | `/avaliacoes` | Cadastra (`titulo`, `disciplina_id`, `data` AAAA-MM-DD, `tipo` prova/trabalho/seminario/outro, `peso`, `nota_maxima`, `nota` opcional) | sim |
| GET | `/avaliacoes/:id` | Detalhe | sim |
| PATCH | `/avaliacoes/:id` | Edita campos enviados (ex.: lançar a nota: `{"nota": 8.5}`) | sim |
| DELETE | `/avaliacoes/:id` | Apaga | sim |
| GET | `/progresso` | Resumo geral + progresso de cada disciplina | sim |
| GET | `/progresso/:disciplinaId` | Progresso detalhado de uma disciplina | sim |
| POST | `/sessoes` | Salva sessão de estudo (`disciplina_id`, `minutos`, `anotacao` opcional) | sim |
| GET | `/sessoes?de=AAAA-MM-DD&ate=AAAA-MM-DD` | Histórico por período | sim |
| GET | `/metas` | Meta semanal atual | sim |
| PUT | `/metas` | Define `horas_semana` | sim |
| GET | `/dashboard` | Resumo: horas da semana, pendências, sequência, próximas atividades e avaliações | sim |
| GET | `/relatorios/semanal` | Últimos 7 dias: por dia, por disciplina e % da meta | sim |
| GET | `/relatorios/mensal?mes=AAAA-MM` | Total do mês, variação e por disciplina | sim |

## Como o progresso é calculado

Para cada disciplina:

- **% de atividades** = atividades concluídas ÷ total de atividades.
- **% de avaliações** = avaliações realizadas (que já têm nota) ÷ total de avaliações.
- **Progresso da disciplina** = média dessas duas porcentagens (se só uma existir, usa só ela).
- **Média das notas** = média ponderada pelo peso, convertida para a escala de 0 a 10. Ex.: nota 8 (peso 2) e nota 6 (peso 1) dão (8×2 + 6×1) ÷ 3 = 7,3.
- Também informa atividades atrasadas (pendentes com prazo vencido), a próxima avaliação e as horas estudadas na semana em relação à `meta_horas` da disciplina.

O `GET /progresso` traz o mesmo cálculo para todas as disciplinas, mais um resumo geral.

## Exemplos

Cadastrar uma avaliação:

```
POST /avaliacoes
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "disciplina_id": 1, "titulo": "Prova 1", "data": "2026-11-20", "peso": 2 }
```

Resposta `201`:

```
{ "id": 1, "disciplina_id": 1, "titulo": "Prova 1", "tipo": "prova", "data": "2026-11-20",
  "peso": 2, "nota": null, "nota_maxima": 10, "status": "pendente" }
```

Lançar a nota depois: `PATCH /avaliacoes/1` com `{ "nota": 8 }` (o status vira `realizada`).

Consultar o progresso (`GET /progresso/1`), resposta resumida:

```
{ "nome": "Cálculo I", "progresso_percentual": 75,
  "atividades": { "total": 4, "concluidas": 3, "atrasadas": 0, "percentual": 75 },
  "avaliacoes": { "total": 2, "realizadas": 1, "percentual": 50, "media": 8 },
  "estudo": { "minutos_semana": 150, "meta_horas": 5, "percentual_meta_semana": 50 },
  "proxima_avaliacao": { "titulo": "Trabalho", "data": "2026-12-01" } }
```

## Códigos de resposta

`200` ok · `201` criado · `204` apagado · `400` dado inválido · `401` sem login ou token inválido · `404` não encontrado · `409` e-mail já cadastrado

## Banco de dados

```
usuarios    (id, nome, email, senha_hash)
disciplinas (id, usuario_id, nome, cor, meta_horas)
atividades  (id, usuario_id, disciplina_id, titulo, descricao, prazo, prioridade, concluida)
avaliacoes  (id, usuario_id, disciplina_id, titulo, tipo, data, peso, nota, nota_maxima)
sessoes     (id, usuario_id, disciplina_id, inicio, minutos, anotacao)
metas       (usuario_id, horas_semana)
```

Apagar uma disciplina apaga junto suas atividades, avaliações e sessões (`ON DELETE CASCADE`).

## Estrutura

```
src/
  server.js           liga o Express e as rotas
  db.js               conexão SQLite e criação das tabelas
  util.js             funções auxiliares
  openapi.js          especificação OpenAPI que alimenta a página /docs
  middleware/auth.js  valida o token JWT
  routes/             um arquivo de rotas por recurso
public/docs/          visual da página /docs (tema.css, tema.js, favicon.svg)
test/smoke.js         teste automático do fluxo principal (49 verificações)
```

## Observações

- Ao criar ou mudar uma rota, atualize também `src/openapi.js`, para a documentação continuar igual ao código.
- Cada usuário só enxerga os próprios dados: toda consulta filtra por `usuario_id`.
- As datas das sessões ficam em UTC; o `FUSO` do `.env` (padrão `-3 hours`, Brasília) define em qual dia cada sessão conta nos relatórios.
- Ao iniciar, o Node pode mostrar o aviso "ExperimentalWarning: SQLite is an experimental feature". É normal e não atrapalha.
- "Semana" nos relatórios significa os últimos 7 dias, incluindo hoje.
