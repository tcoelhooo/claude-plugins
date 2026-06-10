---
name: analista-contratos
description: Analista de contratos da PTA via Autentique. Use SEMPRE que o contexto envolver contratos de assinatura, status de assinatura, painel de contratos, cobrança de contratos pendentes, taxa de assinatura por turma, ou acompanhamento das pós-graduações na Autentique. Termos que acionam: "contratos", "Autentique", "assinaturas", "painel de contratos", "quem não assinou", "contratos pendentes", "taxa de assinatura", "cobrança de contrato". Conecta na API da Autentique, analisa os documentos dos últimos meses e gera um painel HTML (macro mensal, mês atual semana a semana e turma a turma com a lista de alunos).
---

# Analista de Contratos · PTA

Esta skill é a inteligência para analisar os contratos da PTA na **Autentique** e gerar o **painel de assinaturas**. Você (Claude) executa a análise — conecta na API, agrega os dados e preenche o painel.

## Token

O token da Autentique vem da variável de ambiente `CLAUDE_PLUGIN_OPTION_AUTENTIQUE_TOKEN` (configurada na instalação do plugin). Se estiver vazia, peça o token ao usuário ou avise — **nunca invente dados**.

## Como conectar na Autentique

API GraphQL: `POST https://api.autentique.com.br/v2/graphql`
Header: `Authorization: Bearer <TOKEN>` e `Content-Type: application/json`.

Query para listar documentos (paginada, mais novos primeiro):

```graphql
query($limit: Int, $page: Int) {
  documents(limit: $limit, page: $page, showSandbox: false) {
    total
    data {
      id
      name
      created_at
      signatures { name email signed { created_at } viewed { created_at } }
    }
  }
}
```

Use `limit: 60` e vá incrementando `page` a partir de 1.

> Para varreduras grandes (a conta tem milhares de documentos), o caminho eficiente é escrever um pequeno script descartável (Node ou Python) que faça a paginação e a agregação seguindo as regras abaixo, em vez de chamar página por página manualmente. Gere o script na hora, rode, e use o resultado.

## Regra de completude (essencial — não truncar)

Os documentos vêm em ordem decrescente de `created_at`. Para cada mês analisado, **só pare de paginar quando tiver PROVA de que cobriu a janela inteira**:
- alcançou um documento anterior ao início do mês, **ou**
- chegou ao fim dos documentos (página com menos de 60 itens).

Erros transitórios (`no_alive_nodes`, HTTP 429) devem ser **retentados** com espera, nunca ignorados. Se não conseguir cobrir a janela, **reporte o erro** — não entregue número parcial como se fosse final.

## Regras de análise

**Janela de tempo:** por padrão últimos 3 meses. Início do mês = 00:00 em Brasília (= 03:00 UTC). "Hoje" sempre em fuso BR.

**Identidade do cliente / dedup:** um cliente pode ter vários documentos. Agrupe por: e-mail extraído do nome do documento (regex de e-mail) ou, na falta, o e-mail/nome do signatário que **não** seja `financeiro@ptadigital.com.br`.

**Assinou?** o cliente assinou se algum signatário externo (≠ `financeiro@ptadigital.com.br`) tem `signed.created_at`. "Abriu o link" = tem `viewed` ou múltiplos documentos.

**Turma (a partir do nome do documento):**
- bate `/Pós-\w+/i` → a turma é esse código em maiúsculo (ex.: PÓS-SM7, PÓS-BM9, PÓS-BB5, PÓS-FE3, PÓS-MPA…)
- contém TERMO → `TERMO`; ADITIVO → `ADITIVO`; "CONTRATO DE PREST" → `CONSULTORIA`; "CONTRATO PROFESSOR" → `PROFESSOR`
- senão → `OUTROS`

> OUTROS / ADITIVO / TERMO **não são pós-graduação** — são documentos avulsos. Trate à parte na cobrança real.

## O que calcular

1. **Macro mês a mês:** por mês — total de clientes, assinados, pendentes, taxa %.
2. **Mês atual, semana a semana:** divida o mês corrente em faixas por dia (01–07, 08–14, 15–21, 22–28, 29–fim) — total/assinados/pendentes/taxa por faixa.
3. **Turma a turma (mês atual):** por turma — total, assinados, pendentes, taxa % e a **lista de alunos** (nome, e-mail, assinou?, dias pendente, abriu link?).
4. **KPIs (período todo):** total, assinados, taxa geral, pendentes, vencidos +30 dias, quantos nunca abriram o link.

## Como gerar o painel

O layout está em `${CLAUDE_PLUGIN_ROOT}/skills/analista-contratos/painel.html`. Ele tem o texto `__PAINEL_DATA__` no lugar dos dados. Monte um objeto JSON com este formato e **substitua** `__PAINEL_DATA__` por ele, salvando como `painel-contratos.html` na pasta atual:

```js
{
  geradoEm: "10/06/2026 11:48",
  janela: "abr/2026 – mai/2026 – jun/2026",
  mesAtual: "jun/2026",
  kpi: { total, assinados, pendentes, taxaPct, vencidos, semLink, docsVarridos },
  meses:   [ { label:"abr/2026", total, assinados, pendentes, taxaPct }, ... ],
  semanas: [ { label:"S1", range:"01–07", total, assinados, pendentes, taxaPct }, ... ],
  turmas:  [ { turma:"PÓS-SM7", total, assinados, pendentes, taxaPct,
               alunos:[ { n:"Nome", e:"email", a:true, d:null, l:true }, ... ] }, ... ],
  cobertura: [ { label:"abr/2026", docs, paginas, motivoParada }, ... ]
}
```

Onde nos alunos: `n`=nome, `e`=e-mail, `a`=assinou (bool), `d`=dias pendente (ou null), `l`=abriu link (bool).

Depois de gerar, **abra o `painel-contratos.html`** e resuma ao usuário: taxa por mês, total de pendentes e a cobertura da varredura (prova de que varreu tudo).
