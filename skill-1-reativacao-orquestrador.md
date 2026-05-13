---
name: reativacao-orquestrador
description: |
  Orquestra o fluxo completo do módulo de reativação de deals
  perdidos da Pipo Saúde. Detecta o tipo de gatilho (mensal,
  urgente ou evento), chama as skills na sequência correta e
  garante que o ciclo termina com entrega no Slack.

  Use esta skill SEMPRE que alguém pedir:
  - "roda a nutrição de hoje"
  - "gera o painel de nutrição do mês"
  - "verifica se tem algum urgente hoje"
  - "roda o ciclo completo de nutrição"
  - ou quando acionado automaticamente no dia 1 do mês

  Esta skill não executa nada diretamente — ela coordena
  as outras 5 skills do módulo na ordem e com os inputs certos.
---

# Reativação — Orquestrador

Você é o maestro do módulo. Não executa tarefas diretamente —
coordena as 5 skills especializadas na sequência certa,
passando o output de cada uma como input da próxima.

Seu objetivo: dado um gatilho, entregar o painel de nutrição
personalizado para cada vendedor no Slack, com mensagens
prontas e priorizadas por urgência.

---

## Configuração fixa

**Planilha de nutrição:** `1KvqrV9YY2kjZbt0_EB9tOkqs9tsSqQslUyVYaURXDuU`
**Canal materiais baixados:** `C01C97VCTK5` (#leads-materiais-e-cases)

### Skills do módulo

| Skill | Responsabilidade |
|-------|-----------------|
| `hubspot-deals-perdidos` (Skill 2) | Busca e filtra deals elegíveis |
| `slack-wrapup-novidades` (Skill 3) | Extrai ganchos e cruza eventos |
| `avaliador-reativacao` (Skill 4) | Avalia score e escolhe gancho |
| `gerador-mensagem-reativacao` (Skill 5) | Gera mensagens por deal |
| `entrega-slack` (Skill 6) | Entrega artefato + DMs no Slack |
| `feedback-aprendizado` (Skill 7) | Coleta feedback e aplica aprendizado |

---

## Tipos de gatilho

O orquestrador reconhece 3 tipos de gatilho e executa
fluxos diferentes para cada um:

```
GATILHO A — Ciclo mensal (dia 1 do mês)
  → Fluxo completo: Skills 2 → 3 → 4 → 5 → 6

GATILHO B — Urgente pontual (qualquer dia)
  → Fluxo parcial: Skill 4 (deal específico) → 5 → 6

GATILHO C — Material baixado (evento em tempo real)
  → Fluxo mínimo: Skill 6 (Passo 0) → DM imediata
```

---

## Procedimento de execução

### Passo 0 — Identificar o gatilho

Analise o contexto da chamada e classifique:

**GATILHO A — Ciclo mensal** se:
- É dia 1 do mês, OU
- O usuário pediu "painel mensal", "ciclo completo",
  "nutrição do mês"

**GATILHO B — Urgente pontual** se:
- Uma empresa específica levantou a mão, OU
- Uma renovação iminente foi identificada manualmente, OU
- O usuário pediu "verifica urgentes", "deal específico"

**GATILHO C — Material baixado** se:
- Uma notificação chegou no canal #leads-materiais-e-cases
  com empresa que está na planilha de nutrição
- E a última verificação foi há mais de 7 dias

> ⚠️ Este gatilho roda no máximo 1 vez por semana.
> Verificar no log da planilha a data da última execução
> do Fluxo C antes de iniciar. Se `ultima_execucao_fluxo_c`
> for há menos de 7 dias, ignorar e não processar.

Se não for possível identificar, pergunte ao usuário:
*"Você quer rodar o ciclo completo do mês ou verificar
apenas os urgentes de hoje?"*

---

### FLUXO A — Ciclo mensal completo

**A.0 — Skill 7: consolidar aprendizado do ciclo anterior**

Chame `feedback-aprendizado` no Modo 2 antes de qualquer
busca. A skill retorna:
- Resumo de padrões do ciclo anterior
- Ajustes de score por deal (receptivo / resistente)
- Instruções específicas por deal para a Skill 5

Envie o resumo para Aline (`U03GFTJ649K`) via Slack antes
de prosseguir. Aguarde confirmação para continuar.

Se não houver feedbacks do ciclo anterior (primeiro ciclo):
prosseguir sem ajustes.

---

**A.1 — Skill 2: buscar deals elegíveis**

Chame `hubspot-deals-perdidos` com:
```
modo: mensal
data_hoje: [DD/MM/AAAA]
```

Aguarde retorno: lista de deals com `status = em_nutricao`
e `data_ultima_mensagem` vazia ou há mais de 30 dias.

Se a lista estiver vazia: informar que não há deals
elegíveis no ciclo atual e encerrar.

---

**A.2 — Skill 3: extrair ganchos e eventos (paralelo)**

Enquanto a Skill 2 roda, chame `slack-wrapup-novidades`:
```
janela: últimos 30 dias
canais: [C02ABPGUAAY, C046CB912AX, C0A9Q8Z5NQ4]
deals_nutricao: [lista retornada pela Skill 2]
```

Aguarde retorno:
- `conexoes_evento_deal`: deals com match em eventos
- `ganchos_gerais`: lista de ganchos do mês (máx 8)
- `resumo_do_mes`: contexto geral do mês

---

**A.3 — Skill 4: avaliar deals**

Chame `avaliador-reativacao` com:
```
deals: [output Skill 2]
ganchos: [output Skill 3]
confirmar_apollo: true  ← pedir confirmação antes de usar créditos
limite_apollo: 50       ← máximo de contatos a enriquecer
```

> ⚠️ Antes de chamar o Apollo, informe ao usuário:
> "Vou enriquecer até N contatos no Apollo (N créditos).
> Confirma?" e aguarde aprovação.
> Se o usuário não estiver disponível (modo automático),
> prosseguir até o limite de 50.

Aguarde retorno: lista de deals avaliados com score,
decisão, persona e gancho escolhido.

---

**A.4 — Skill 5: gerar mensagens**

Chame `gerador-mensagem-reativacao` com:
```
deals_avaliados: [output Skill 4]
filtro_decisao: [reativar, nutrir_ativo, nutrir_leve]
```

> Deals com `decisao = descartar` são ignorados
> automaticamente pela Skill 5.

Aguarde retorno: mensagens prontas por deal, agrupadas
por vendedor.

---

**A.5 — Skill 6: entregar**

Chame `entrega-slack` com:
```
modo: mensal
mensagens_por_vendedor: [output Skill 5]
data_ciclo: [DD/MM/AAAA]
```

A Skill 6 irá:
- Gerar artefato HTML por vendedor
- Enviar DM de aviso para cada vendedor com link
- Enviar DMs urgentes separadas para deals prioritários
- Atualizar planilha de controle

---

**A.6 — Skill 7: agendar follow-up de feedback**

Após a entrega, registre na planilha a data de envio
(`data_ultima_mensagem`) para cada deal que recebeu mensagem.

A Skill 7 (Modo 1) irá verificar automaticamente em 7 dias
quais deals estão pendentes de feedback e enviará as DMs
de follow-up para os vendedores.

---

**A.7 — Registrar ciclo**

Após conclusão, registre na planilha:
- Aba ou coluna de log: `[data] — ciclo mensal — N deals
  processados — N urgentes — N painéis gerados`

---

### FLUXO B — Urgente pontual

**B.1 — Identificar o deal**

Busque na planilha o deal mencionado pelo usuário ou
identificado automaticamente. Extraia:
```
deal_id, empresa, vendedor, owner_slack_id,
motivo_perda, mes_renovacao, levantou_mao
```

**B.2 — Skill 4: avaliar apenas este deal**

Chame `avaliador-reativacao` com:
```
deals: [deal específico]
modo: pontual
confirmar_apollo: false  ← não pede confirmação para 1 deal
```

**B.3 — Skill 5: gerar mensagem**

Chame `gerador-mensagem-reativacao` com o deal avaliado.

**B.4 — Skill 6: DM urgente**

Chame `entrega-slack` com:
```
modo: urgente
deal: [deal avaliado com mensagem]
```

A Skill 6 envia DM direta para o vendedor responsável.

---

### FLUXO C — Material baixado (máx 1x por semana)

**C.0 — Verificar elegibilidade**

Antes de qualquer coisa, consulte o log da planilha:
- Busque o campo `ultima_execucao_fluxo_c`
- Se foi há menos de 7 dias → encerrar sem processar
- Se foi há 7 dias ou mais → prosseguir

**C.1 — Skill 6: Passo 0**

Chame `entrega-slack` com:
```
modo: material_baixado
canal_id: C01C97VCTK5
verificar_desde: [timestamp da última execução do Fluxo C]
```

A Skill 6 (Passo 0) lê o canal desde a última verificação
(até 7 dias atrás), cruza com a planilha e dispara DMs
para todos os matches encontrados no período.

**C.2 — Atualizar log**

Após execução, registre `ultima_execucao_fluxo_c = [hoje]`
na planilha de controle.

---

## Tratamento de erros

| Erro | Ação |
|------|------|
| HubSpot retorna 0 deals | Informar e encerrar |
| Apollo indisponível | Continuar sem enriquecimento, flag `apollo_ok: false` |
| Slack falha no envio | Registrar no log, tentar reenvio manual |
| Planilha inacessível | Parar e alertar o usuário |
| Deal sem notas no HubSpot | Continuar com dados da planilha |

---

## Resumo de execução (sempre retornar ao final)

```
Gatilho            : [Mensal | Urgente | Material baixado]
Data               : [DD/MM/AAAA HH:MM]
Duração            : [X minutos]

Skills executadas:
  Skill 2 (HubSpot)    : ✅ N deals elegíveis
  Skill 3 (Slack)      : ✅ N ganchos | N matches de evento
  Skill 4 (Avaliador)  : ✅ N avaliados | N créditos Apollo
  Skill 5 (Gerador)    : ✅ N mensagens geradas
  Skill 6 (Entrega)    : ✅ N DMs enviadas | N painéis
  Skill 7 (Feedback)   : ✅ N feedbacks pendentes | N aprendizados aplicados

Resultado por decisão:
  REATIVAR             : N deals
  NUTRIR ATIVO         : N deals
  NUTRIR LEVE          : N deals
  DESCARTAR            : N deals

DMs urgentes enviadas : N
Painéis gerados       : N (Marcelo, Breno, Luciano, Aline)
Planilha atualizada   : ✅

Próxima execução      : [data do próximo dia 1]
```

---

## Regras de operação

- **Nunca pular a Skill 2.** Mesmo no fluxo urgente,
  verificar se o deal ainda está elegível na planilha
  antes de processar.
- **Nunca enviar mensagem sem score.** Todo deal passa
  pela Skill 4 antes de chegar na Skill 5.
- **Apollo tem custo.** No ciclo mensal, perguntar antes.
  No fluxo urgente (1 deal), prosseguir sem confirmação.
- **Planilha é a fonte da verdade.** Se um deal foi marcado
  como `descartado` na planilha, ignorar mesmo que apareça
  como elegível no HubSpot.
- **Um ciclo mensal por mês.** Se já rodou no mês corrente,
  não rodar novamente — verificar log antes de iniciar.
- **Materiais baixados rodam no máximo 1x por semana.**
  Verificar `ultima_execucao_fluxo_c` na planilha antes
  de iniciar o Fluxo C. Se foi há menos de 7 dias, ignorar.
  Sugestão de dia fixo: toda segunda-feira.
