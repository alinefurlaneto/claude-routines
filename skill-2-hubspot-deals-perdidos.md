---
name: hubspot-deals-perdidos
description: |
  Busca deals perdidos no HubSpot que chegaram até Proposta ou
  Negociação, cruza com a planilha de controle de nutrição no
  Google Drive, e retorna apenas os deals elegíveis para receber
  mensagem neste ciclo mensal.

  Use esta skill SEMPRE que precisar de:
  - Lista de deals elegíveis para nutrição no mês corrente
  - Verificar quais contas perdidas ainda não foram nutridas
  - Input para o avaliador e gerador de mensagem

  Retorna lista estruturada pronta para o avaliador trabalhar.
---

# HubSpot Deals Perdidos — com controle via Google Sheets

Você é responsável por buscar deals perdidos no HubSpot, cruzar
com a planilha de controle, e retornar apenas os deals que
precisam de ação neste mês. Nenhuma mensagem é gerada aqui —
seu output alimenta o avaliador.

---

## Configuração fixa

**Portal HubSpot:** 8518826
**Pipeline alvo:** Oportunidades (`default`)
**Planilha de controle:** `Nutrição Deals Perdidos — Pipo Saúde`
**File ID Google Drive:** `1T4c3XlbgbLV2S9FS14iOuA8BtFkq83csWMxi9P2gtPk`

### Executivos e seus IDs

| Nome | HubSpot Owner ID | Slack User ID |
|------|-----------------|---------------|
| Aline Furlaneto | `192702358` | U03GFTJ649K |
| Marcelo Silva | `465298555` | U05EW1UF1C7 |
| Luciano Nascimento Moreira | `175077773` | U035X1LGBAP |
| Breno Haus | `331668616` | U04MHL72USV |
| Luciano Moreira | (buscar via search_owners) | U035X1LGBAP |

### Status válidos na planilha

| Status | Significado |
|--------|------------|
| `aguardando` | Perdido há menos de 3 meses — na carência |
| `em_nutricao` | Nutrição ativa — recebe mensagem mensalmente |
| `reativado` | Voltou ao funil — parar nutrição |
| `descartado` | Vendedor decidiu não nutrir — ignorar |

---

## Procedimento de execução

### Passo 1 — Ler a planilha de controle

Use `download_file_content` com `exportMimeType: text/csv`
no file ID: `1mtK-HoF5pBtfYHqZjpa8mIXqeq0PyvWA8YTxWW4WoV4`

Parse o CSV e carregue todas as linhas em memória como lista de
objetos indexados por `deal_id`.

---

### Passo 2 — Buscar deals perdidos elegíveis no HubSpot

Calcule o timestamp de **12 meses atrás** em ms (janela ampla
para capturar contas em qualquer fase da nutrição).

> Por que 12 meses e não 3? A planilha controla a carência.
> O HubSpot precisa trazer todos os deals ativos na nutrição,
> incluindo os que foram perdidos há mais tempo.

Execute `search_crm_objects`:

```
objectType: deals
filterGroups:
  - filters:
    - propertyName: hs_is_closed_lost
      operator: EQ
      value: "true"
    - propertyName: hs_v2_date_entered_closedlost
      operator: GTE
      value: "[timestamp_12_meses_atras_em_ms]"
    - propertyName: em_que_etapa_o_negocio_estava_antes_de_ser_perdido_
      operator: IN
      values:
        - "11 Negociação (Oportunidades)"
        - "12 Proposta aceita (Oportunidades)"
        - "09 Elaboração de proposta (Oportunidades)"

properties:
  - dealname
  - hubspot_owner_id
  - hs_v2_date_entered_closedlost
  - em_que_etapa_o_negocio_estava_antes_de_ser_perdido_
  - motivo_macro_de_perda
  - motivo_micro_de_perda
  - observacoes_de_perda_de_tof
  - pre_vendas_mes_proxima_renovacao_da_apolice
  - amount
  - notes_last_contacted

limit: 200
sorts:
  - propertyName: hs_v2_date_entered_closedlost
    direction: DESCENDING
```

Filtre mantendo apenas deals cujo `hubspot_owner_id` seja de
um dos executivos da tabela acima.

---

### Passo extra — Excluir empresas com deal ativo no funil

Após buscar os deals perdidos, faça uma segunda busca:

```
objectType: deals
filters:
  - hubspot_owner_id: [mesmo owner]
  - hs_is_closed: false         ← deals ainda abertos
properties: [dealname, nome_da_empresa]
```

Para cada deal perdido elegível, verifique se a empresa
(campo `nome_da_empresa` ou nome do deal) aparece entre
os deals ativos. Se sim: **excluir da lista de nutrição**.

> Lógica: se já existe um deal aberto para a empresa,
> o vendedor já está em contato ativo. Enviar mensagem
> de nutrição seria duplicar esforço e pode confundir
> o prospect.
>
> Use correspondência parcial no nome da empresa.
> Ex: "Sympla - Retomada" ativo → exclui "Sympla - Levantada de Mão" perdido.

Registre as exclusões no resumo de execução:
```
Excluídos por deal ativo no funil: N empresas
  - [empresa] → deal ativo [nome do deal]
```

---

### Exceção de carência — evento iminente

Deals com `status = aguardando` (dentro dos 90 dias de
carência) normalmente não entram na lista de nutrição.

**Exceção:** se a Skill 3 retornar evento com `urgente: true`
(≤ 14 dias) E o contato do deal tiver perfil compatível
com o público do evento, incluir o deal com flag:

```
excecao_carencia : true
motivo           : evento iminente — [nome do evento] em [data]
instrucao        : não usar mensagem padrão de nutrição;
                   usar convite para o evento como único gancho;
                   não mencionar o deal perdido recente
```

> Lógica: um evento com janela de 14 dias fecha antes da
> carência de 90 dias terminar. Perder a janela é pior
> do que quebrar a carência. O convite é relacional —
> não é pitch, não reabre a negociação.

---

### Passo 3 — Sincronizar HubSpot com a planilha

Para cada deal retornado pelo HubSpot:

**Se o deal NÃO existe na planilha** → adicione nova linha:
```
deal_id          → ID do HubSpot
empresa          → dealname
vendedor         → nome do owner (resolva via tabela de IDs)
owner_id_hubspot → hubspot_owner_id
data_perda       → hs_v2_date_entered_closedlost (DD/MM/AAAA)
etapa_antes_perda→ em_que_etapa_o_negocio_estava_antes_de_ser_perdido_
motivo_perda     → motivo_macro_de_perda
data_inicio_nutricao → vazio
data_ultima_mensagem → vazio
contador         → 0
status           → "aguardando"
ultimo_gancho    → vazio
observacoes      → vazio
```

**Se o deal JÁ existe na planilha** → não altere nada.
A planilha é a fonte da verdade para status.

---

### Passo 4 — Verificar elegibilidade para este ciclo

Para cada linha da planilha, aplique as regras:

**Regra 0 — Descarte automático (verificar PRIMEIRO)**
```
se motivo_perda contém "Alinhamento com o Perfil Ideal de Cliente (ICP)":
  → status = "descartado" → PULAR

se dias_desde_perda > 400:
  → status = "descartado" → PULAR
```

**Regra 1 — Carência de 3 meses**
```
dias_desde_perda = hoje - data_perda (em dias)

se dias_desde_perda < 90:
  → status permanece "aguardando" → PULAR

se dias_desde_perda >= 90 E status = "aguardando":
  → mudar status para "em_nutricao"
  → registrar data_inicio_nutricao = hoje
  → ELEGÍVEL
```

**Regra 2 — Frequência mensal**
```
se status = "em_nutricao" E data_ultima_mensagem não vazia:
  dias_desde_ultima = hoje - data_ultima_mensagem

  se dias_desde_ultima < 28 → JÁ NUTRIDO ESTE MÊS → PULAR
  se dias_desde_ultima >= 28 → ELEGÍVEL

se status = "em_nutricao" E data_ultima_mensagem vazia:
  → primeira mensagem → ELEGÍVEL
```

**Regra 3 — Status de parada**
```
se status = "reativado"  → PULAR
se status = "descartado" → PULAR
```

---

### Passo 5 — Buscar notas recentes dos deals elegíveis

Para cada deal elegível, busque as 3 últimas notas no HubSpot:

```
objectType: engagements
filterGroups:
  - associatedWith:
    - objectType: deals
      operator: EQUAL
      objectIdValues: [deal_id]

properties:
  - hs_note_body
  - hs_timestamp

sorts:
  - propertyName: hs_timestamp
    direction: DESCENDING

limit: 3
```

---

### Passo 6 — Atualizar planilha

Após determinar elegíveis, atualize a planilha via
`create_file` (sobrescrevendo o CSV com o mesmo file ID):

- Deals que mudaram de `aguardando` → `em_nutricao`:
  → Atualize `status` e `data_inicio_nutricao` com hoje

> Não atualize `data_ultima_mensagem` aqui — isso é feito
> pela skill de entrega, após o Slack ser enviado com sucesso.

---

### Passo 7 — Retornar lista estruturada

Para cada deal elegível, retorne o objeto:

```
deal_id                    : [id HubSpot]
deal_name                  : [nome do deal]
empresa                    : [nome da empresa]
owner_id                   : [hubspot_owner_id]
owner_slack_id             : [slack user id]
link_hubspot               : https://app.hubspot.com/contacts/8518826/record/0-3/[id]
valor                      : [amount em R$]
etapa_antes_perda          : [etapa]
data_perda                 : [DD/MM/AAAA]
motivo_macro               : [motivo_macro_de_perda]
motivo_micro               : [motivo_micro_de_perda]
observacoes_perda          : [observacoes_de_perda_de_tof]
mes_renovacao              : [pre_vendas_mes_proxima_renovacao_da_apolice]
dias_desde_perda           : [número]
numero_mensagens_enviadas  : [contador da planilha]
ultimo_gancho_usado        : [ultimo_gancho da planilha — null se primeira]
notas_recentes             : [lista das 3 notas mais recentes]
```

Ao final, imprima o resumo de execução:
```
✅ Sincronização concluída
────────────────────────────
Total buscado no HubSpot : X deals
Novos adicionados        : Y
Em carência (aguardando) : Z
Elegíveis este mês       : W
────────────────────────────
```

---

## Regras importantes

- **A planilha é a fonte da verdade para status e datas.**
  O HubSpot é a fonte da verdade para dados do deal.
- **Nunca sobrescreva `reativado` ou `descartado`.**
  Esses status são definidos manualmente pelo vendedor.
- **Calcule todas as datas dinamicamente** com base na data
  real de execução — nunca hardcode valores de data.
- **Se a planilha estiver vazia** (primeira execução),
  popule-a inteiramente com os deals do HubSpot aplicando
  a regra de carência para definir o status inicial.
- **Se um deal estiver como `reativado` na planilha mas
  aparecer como perdido no HubSpot**, mantenha o status
  da planilha — pode ser um deal novo com nome similar.
