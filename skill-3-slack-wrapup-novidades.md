---
name: slack-wrapup-novidades
description: |
  Lê os canais de vendas e eventos do Slack do último mês,
  extrai highlights para ganchos de reativação, coleta listas
  de presença de eventos e cruza com a planilha de nutrição
  para identificar quais deals perdidos têm conexão com eventos
  recentes da Pipo.

  Use esta skill SEMPRE que precisar de:
  - Resumo mensal das novidades de Pipo para ganchos de mensagens
  - Identificar quais prospects da lista de nutrição estiveram
    (ou foram convidados) em eventos recentes da Pipo
  - Input enriquecido para o avaliador e gerador de mensagem

  Retorna lista de ganchos + mapa de conexão evento↔deal.
---

# Slack Wrap-up — Novidades e Cruzamento com Eventos

Você é responsável por:
1. Extrair ganchos de conteúdo dos canais de vendas e eventos
2. Coletar listas de presença de eventos linkadas nos canais
3. Cruzar essas listas com a planilha de nutrição de deals perdidos
4. Retornar o contexto enriquecido para o avaliador

---

## Configuração fixa

**Planilha de nutrição (fonte da verdade de deals):**
`1JCii0etNdJKLt-XB5UANt8ATUJK8aBTb88-pgHuEPrg`

### Canais monitorados

| Canal | ID | O que contém |
|-------|-----|-------------|
| #vendas-conceito | `C02ABPGUAAY` | Novidades de produto; materiais ricos; pesquisas; cadências; argumentos de venda |
| #eventos | `C046CB912AX` | Eventos Pipo (Humanship; lançamentos; roadshows); listas de presença |
| #abm_vendas | `C0A9Q8Z5NQ4` | Status de eventos; listas ABM; ações de marketing para vendas |

---

## Procedimento de execução

### Passo 1 — Calcular janela de tempo

A janela de leitura é **30 dias atrás até hoje**.

Calcule o timestamp Unix em segundos para 30 dias atrás:
- Fórmula: `(data_hoje - 30 dias) às 00:00:00 UTC → em segundos`
- Exemplo: se hoje é 01/06/2026, o início é 02/05/2026 → `1746144000`

---

### Passo 2 — Ler os 3 canais do Slack

Para cada canal, use `slack_read_channel`:

```
channel_id: [ID do canal]
oldest: [timestamp_30_dias_atras em segundos]
limit: 100
response_format: concise
```

Pagine com `next_cursor` até buscar todas as mensagens do período.

Durante a leitura, separe as mensagens em dois grupos:
- **Grupo A:** mensagens com links para Google Sheets
  (contêm `docs.google.com/spreadsheets`) com contexto de evento
- **Grupo B:** mensagens de conteúdo relevante para ganchos
  (sem links de planilha ou com links de outros tipos)

---

### Passo 3 — Processar listas de presença (Grupo A)

Para cada mensagem do Grupo A:

**3.1 — Identificar se é lista de evento**
A mensagem deve conter palavras como: "lista", "convidados",
"confirmados", "presença", "planilha de controle", "evento".
Se não contiver, ignore o link.

**3.2 — Extrair fileId do link**
Do URL `https://docs.google.com/spreadsheets/d/[fileId]/edit...`
extraia o `fileId`.

**3.3 — Ler a planilha**
Use `read_file_content` com o fileId extraído.

**3.4 — Extrair lista de participantes**
Para cada linha da planilha, extraia:
```
empresa: [nome da empresa]
contato: [nome da pessoa]
email: [email se disponível]
cargo: [cargo]
tipo: [Prospect | Cliente]
status_presenca: [Sim | Não | Não foi | Não pode ir | Convidado]
```

**3.5 — Registrar metadados do evento**
```
nome_evento: [extraído do contexto da mensagem ou aba da planilha]
data_evento: [DD/MM/AAAA]
canal_origem: [#nome-do-canal]
```

---

### Passo 4 — Cruzar lista de eventos com planilha de nutrição

Leia a planilha de nutrição via `download_file_content` com
`exportMimeType: text/csv` no fileId acima.

Para cada deal na planilha com `status = em_nutricao` ou
`status = aguardando`, verifique se a empresa aparece na
lista de algum evento.

**Critério de match:** compare o nome da empresa do deal
(coluna `empresa`) com o nome da empresa na lista de evento.
Use correspondência parcial — "Keeggo" bate com "Keeggo S.A."

Para cada match encontrado, registre:
```
deal_id: [id do deal na planilha de nutrição]
empresa: [nome da empresa]
vendedor: [dono do deal]
evento_nome: [nome do evento]
evento_data: [data do evento]
contato_no_evento: [nome do contato que foi/foi convidado]
status_presenca: [Sim | Não foi | Convidado mas não confirmou]
gancho_evento: [frase sugerida baseada no status]
```

**Regras para `gancho_evento` por status:**

- `Sim` (foi ao evento):
  *"Vi que a [Pessoa] esteve no nosso evento [Nome]. Gostei bastante
  de ter vocês lá — algum dos temas tocou em algo do dia a dia de
  vocês?"*

- `Não foi` / `Não pode ir` (foi convidado mas não foi):
  *"Tinha te convidado para o nosso [Nome do Evento] mas não deu
  certo. Separei o material que apresentamos lá — posso te mandar?"*

- `Convidado mas não confirmou` (estava na lista sem resposta):
  *"Vi que seu nome estava na nossa lista de convidados para o
  [Nome do Evento]. Caso queira, posso te mandar o conteúdo
  que apresentamos."*

---

### Passo 5 — Extrair ganchos de conteúdo (Grupo B)

Para cada mensagem do Grupo B, classifique e extraia:

**Categorias:**

| Categoria | Exemplos |
|-----------|----------|
| `lançamento_produto` | Nova funcionalidade; novo relatório; nova integração |
| `pesquisa_conteudo` | Pesquisa de benefícios; relatório de mercado; estudo |
| `argumento_venda` | Nova cadência; case; argumento novo; história de sucesso |
| `novidade_mercado` | Tendência; dado de mercado; insight de RH |

Para cada gancho, retorne:
```
categoria: [categoria]
titulo: [nome curto — máx 8 palavras]
descricao: [1-2 frases explicando o que é]
data: [DD/MM/AAAA]
canal: [#nome-do-canal]
exemplo_de_abertura: [frase pronta para o vendedor adaptar]
link: [URL original se houver]
```

Retorne no máximo **8 ganchos de conteúdo**, priorizando
lançamentos e materiais com link.

---

### Passo 6 — Retornar output estruturado

Retorne em dois blocos:

**Bloco 1 — Mapa evento↔deal (mais importante)**
Lista de deals com conexão a eventos recentes, ordenados por
força do gancho (foi ao evento > foi convidado e não foi >
foi convidado sem resposta):

```
conexoes_evento_deal:
  - deal_id: ...
    empresa: ...
    vendedor: ...
    evento: ...
    status_presenca: ...
    gancho_evento: ...
```

**Bloco 2 — Ganchos gerais de conteúdo**
Lista de ganchos extraídos dos canais (máx 8), ordenados
por prioridade.

**Bloco 3 — Resumo do mês**
Texto corrido de 3-5 frases com os principais temas do mês
para contexto geral do avaliador.

---

## Regras importantes

- **Bloco 1 tem prioridade sobre tudo.** Uma empresa que foi a
  um evento tem um gancho muito mais forte do que qualquer
  conteúdo genérico.
- **Não force matches.** Se o nome não bater com clareza,
  não registre a conexão — falso positivo é pior que não ter.
- **Preserve os links originais** dos materiais e eventos.
- **Tom humano, nunca corporativo.** Os exemplos de abertura
  devem soar como mensagem de WhatsApp de alguém que se conhece,
  não como email de marketing.
- **Não invente ganchos.** Se o mês foi fraco em novidades,
  retorne poucos itens com qualidade.
- **Eventos passados são ganchos válidos** — "você foi ao
  evento X" ainda funciona por semanas depois.
