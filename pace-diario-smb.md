---
name: pace-diario-smb
description: |
  Gera e envia o relatório de pace diário do time de vendas SMB para o Slack.

  Use esta skill SEMPRE que alguém pedir para:
  - Gerar o pace diário do time SMB ("pace do dia", "pace diário", "relatório de pace")
  - Checar o progresso diário de Breno, Larissa ou Barbara contra as metas mensais
  - Enviar o update do funil SMB no Slack
  - Saber como o time SMB está em relação ao pace esperado

  A skill consulta o HubSpot para contar quantos negócios entraram em cada etapa
  do funil SMB durante o mês corrente, calcula o pace esperado com base no dia
  útil de hoje, aplica sinais de farol (🟢🟡🔴) para quem tem meta e posta
  no canal #time_vendas_smb.
---

# Pace Diário SMB

## Configuração fixa (não muda)

**Portal HubSpot:** 8518826  
**Canal Slack:** #time_vendas_smb → ID `C0881B22ZQF`

### Consultores e seus IDs HubSpot (campo `cn`)
| Consultor | ID | Tem meta/farol? |
|---|---|---|
| Breno Haus | `331668616` | Sim |
| Larissa Martins | `1134248428` | Sim |
| Barbara Costa Borges | `91493869` | Não (só número real) |

### Etapas do funil e seus IDs HubSpot
| # | Etapa | deal stage ID |
|---|---|---|
| 01 | Mapeamento | `72557853` |
| 02 | Em cadência | `24595559` |
| 03 | Lead Conectado | `24595562` |
| 04 | Qualif. Agendada | `24595560` |
| 05 | Qualif. Realizada | `24595561` |
| 06 | Diagnóstico | `appointmentscheduled` |
| 07 | Desenvolvimento | `9102669` |

Para cada etapa, a propriedade HubSpot que registra quando o negócio entrou nela é:
`hs_v2_date_entered_[deal_stage_id]`  
Exemplo: `hs_v2_date_entered_72557853` para Mapeamento.

---

## Configuração mensal (atualizar todo mês)

> **Nota:** Esta seção muda a cada mês. Atualize antes do primeiro dia útil do mês.

### Maio/2026

**Total de dias úteis:** 20  
**Feriado:** 01/05 (Dia do Trabalho)

**Calendário de Business Days:**
```
BD1=04/05, BD2=05/05, BD3=06/05, BD4=07/05, BD5=08/05,
BD6=11/05, BD7=12/05, BD8=13/05, BD9=14/05, BD10=15/05,
BD11=18/05, BD12=19/05, BD13=20/05, BD14=21/05, BD15=22/05,
BD16=25/05, BD17=26/05, BD18=27/05, BD19=28/05, BD20=29/05
```

**Timestamps Unix para filtros HubSpot (ms):**
- Início: `1777593600000` (01/05/2026 00:00:00 UTC)
- Fim: `1780271999000` (31/05/2026 23:59:59 UTC)

**Metas mensais:**

| Etapa | Breno | Larissa | Barbara |
|---|---|---|---|
| Mapeamento | 70 | 100 | — |
| Em cadência | 70 | 100 | — |
| Lead Conectado | 30 | 35 | — |
| Qualif. Agendada | 15 | 17 | — |
| Qualif. Realizada | 12 | 15 | — |
| Diagnóstico | 6 | 8 | — |
| Desenvolvimento | 5 | 7 | — |

---

## Procedimento de execução

### Passo 1 — Identificar o dia útil de hoje

Consulte o calendário de BDs da seção de configuração mensal e identifique qual BD corresponde à data de hoje (formato DD/MM). Este é o `BD_atual`.

### Passo 2 — Calcular o pace esperado

```
pace = BD_atual / total_BDs_do_mês
```

Exemplo: BD7 de 20 → pace = 7/20 = 35%

### Passo 3 — Buscar dados no HubSpot

Para **cada uma das 7 etapas**, faça uma busca via `search_crm_objects` no HubSpot:

- **objectType:** `deals`
- **filterGroups:** três grupos (OR entre eles):
  - Grupo 1: `cn = "331668616"` AND `hs_v2_date_entered_[stage_id]` BETWEEN início AND fim
  - Grupo 2: `cn = "1134248428"` AND `hs_v2_date_entered_[stage_id]` BETWEEN início AND fim
  - Grupo 3: `cn = "91493869"` AND `hs_v2_date_entered_[stage_id]` BETWEEN início AND fim
- **properties:** `["cn", "dealname", "hs_v2_date_entered_[stage_id]"]`
- **limit:** 200

> **Por que uma busca por etapa?** Cada `hs_v2_date_entered_*` é uma propriedade distinta; não é possível filtrar por múltiplas datas de entrada em uma única query sem falsos positivos.

> **Verifique o `total`** retornado. Se for maior que 200, use paginação (offset) para pegar todos os registros.

Após a busca, **separe os resultados por `cn`** e conte quantos negócios cada consultor tem naquela etapa.

Você pode rodar as 7 buscas em paralelo para economizar tempo.

### Passo 4 — Calcular farol (apenas para quem tem meta)

```
esperado = meta × pace

🟢  se real ≥ esperado × 1.10   (acima de +10%)
🟡  se real ≥ esperado × 0.90   (dentro de ±10%)
🔴  se real < esperado × 0.90   (abaixo de -10%)
```

Barbara não tem meta ainda — exibir apenas o número real, sem farol.

### Passo 5 — Enviar no Slack

Envie no canal `C0881B22ZQF` com este formato exato (use *negrito* com asteriscos do Slack):

```
📊 Pace Diário — DD/MM/AAAA
Dia útil X de 20 → pace esperado: Y%

*BRENO HAUS*
🟢/🟡/🔴 Mapeamento: [real] / esp. [esperado]
🟢/🟡/🔴 Em cadência: [real] / esp. [esperado]
🟢/🟡/🔴 Lead Conectado: [real] / esp. [esperado]
🟢/🟡/🔴 Qualif. Agendada: [real] / esp. [esperado]
🟢/🟡/🔴 Qualif. Realizada: [real] / esp. [esperado]
🟢/🟡/🔴 Diagnóstico: [real] / esp. [esperado]
🟢/🟡/🔴 Desenvolvimento: [real] / esp. [esperado]

*LARISSA MARTINS*
🟢/🟡/🔴 Mapeamento: [real] / esp. [esperado]
🟢/🟡/🔴 Em cadência: [real] / esp. [esperado]
🟢/🟡/🔴 Lead Conectado: [real] / esp. [esperado]
🟢/🟡/🔴 Qualif. Agendada: [real] / esp. [esperado]
🟢/🟡/🔴 Qualif. Realizada: [real] / esp. [esperado]
🟢/🟡/🔴 Diagnóstico: [real] / esp. [esperado]
🟢/🟡/🔴 Desenvolvimento: [real] / esp. [esperado]

*BARBARA COSTA BORGES*
Mapeamento: [real]
Em cadência: [real]
Lead Conectado: [real]
Qualif. Agendada: [real]
Qualif. Realizada: [real]
Diagnóstico: [real]
Desenvolvimento: [real]
```

Use vírgula como separador decimal nos números (ex: `24,5` não `24.5`).  
Arredonde os valores esperados para 1 casa decimal.

---

## Manutenção mensal

No início de cada mês novo, atualize a seção **"Configuração mensal"** com:
1. O novo calendário de business days (excluindo feriados e fins de semana)
2. Os timestamps Unix do início e fim do mês em milissegundos
3. As metas mensais de cada consultor (se houver mudança)
4. Quando Barbara receber metas, mover ela para a coluna de consultores com meta/farol e atualizar o Passo 4 e o formato do Slack

Para calcular os timestamps Unix: use Python `import calendar, datetime` ou pesquise online.
