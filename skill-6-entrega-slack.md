---
name: entrega-slack
description: |
  Gera o artefato HTML personalizado de nutrição para cada vendedor
  e envia a notificação no Slack. Roda no dia 1 de cada mês como
  parte do ciclo mensal de nutrição. Também detecta casos urgentes
  (levantada de mão ou renovação em 30–60 dias) e envia DM
  individual imediata, fora do ciclo mensal.

  Use esta skill SEMPRE que precisar de:
  - Entregar o painel mensal de nutrição para os vendedores
  - Enviar DM urgente para deal com levantada de mão ou
    renovação iminente
  - Gerar o artefato HTML com cards colapsados por vendedor

  Recebe: output da Skill 5 (mensagens geradas por deal)
  Entrega: artefato HTML + mensagem Slack por vendedor
---

# Entrega Slack — Painel de Nutrição

Você é responsável pela entrega final do sistema. Recebe as
mensagens prontas da Skill 5 e transforma em dois tipos de
entrega no Slack:

1. **Painel mensal** — artefato HTML com todos os deals do
   vendedor, enviado no dia 1 com uma DM de aviso
2. **DM urgente** — mensagem direta imediata para deals com
   levantada de mão ou renovação em 30–60 dias

---

## Configuração fixa

**Canal do time:** `C0881B22ZQF` (#time_vendas_smb)
**Canal de materiais baixados:** `C01C97VCTK5` (#leads-materiais-e-cases)
**Planilha de nutrição:** `1KvqrV9YY2kjZbt0_EB9tOkqs9tsSqQslUyVYaURXDuU`

### Vendedores e Slack IDs

| Vendedor | Slack User ID |
|----------|--------------|
| Aline Furlaneto | `U03GFTJ649K` |
| Marcelo Silva | `U05EW1UF1C7` |
| Breno Haus | `U04MHL72USV` |
| Luciano Nascimento Moreira | `U035X1LGBAP` |

---

## Critérios de entrega

### DM urgente (imediata, qualquer dia do mês)

Disparar DM individual quando:
- `decisao = reativar` E `levantou_a_mao = true`, OU
- `decisao = reativar` E renovação entre 30–60 dias, OU
- Empresa da planilha de nutrição baixou material rico
  (detectado via monitoramento do canal #leads-materiais-e-cases)

Formato da DM urgente:
```
🚨 *[Empresa] — ação necessária*

[Nome do contato] · [Cargo] · [Empresa]
*Por que agora:* [motivo em 1 linha]

*Mensagem sugerida:*
> [mensagem gerada pela Skill 5]

[botão link HubSpot]
```

### Painel mensal (dia 1 do mês)

Um artefato HTML por vendedor + uma DM com link.

DM de aviso (texto simples, não flood):
```
Bom dia, [primeiro nome]! 🗓️
Seu painel de nutrição de [mês] está pronto —
[N] deals para agir agora, [N] para manter presença.

👉 [link do artefato]
```

---

## Procedimento de execução

### Passo 0 — Monitorar canal de materiais baixados

> Este passo roda em modo contínuo (ou no início de cada
> execução) e é independente do ciclo mensal. Qualquer
> match detectado dispara DM urgente imediatamente.

**0.1 — Ler mensagens novas do canal**
Leia o canal `C01C97VCTK5` (#leads-materiais-e-cases)
buscando mensagens do tipo bot (HubSpot) desde a última
execução verificada.

```
channel_id: C01C97VCTK5
oldest: [timestamp da última verificação]
limit: 100
```

**0.2 — Extrair dados do lead**
Cada mensagem do HubSpot contém uma notificação de download.
Para cada mensagem, busque no HubSpot o contato vinculado
usando a propriedade `recent_conversion_event_name` para
identificar qual material foi baixado.

Busque o contato pelo email ou empresa via HubSpot e
extraia:
```
email_contato    : [email]
empresa_contato  : [nome da empresa]
material_baixado : [nome do material — recent_conversion_event_name]
data_download    : [data/hora]
```

**0.3 — Cruzar com planilha de nutrição**
Para cada lead identificado, verifique se a empresa
aparece na planilha de nutrição com `status = em_nutricao`
ou `status = aguardando`.

Critério de match: compare `empresa_contato` com coluna
`empresa` da planilha. Use correspondência parcial —
"Sympla Internet" bate com "Sympla".

**0.4 — Se encontrar match: disparar DM urgente**

Para cada match encontrado, monte e envie DM para o
`owner_slack_id` do vendedor responsável:

```
📥 *[Empresa] baixou material agora*

*[Nome do contato]* · [Cargo] · [Empresa]
*Material:* [nome do material baixado]
*Quando:* [data/hora do download]

*Por que é relevante:* esse prospect está na sua lista
de nutrição desde [data_inicio_nutricao]. O download é
um sinal de interesse ativo — momento ideal para retomar.

*Mensagem sugerida:*
"[Nome], vi que você baixou [material] — ótimo conteúdo!
Isso me fez lembrar da conversa que tivemos sobre
benefícios. Ainda faz sentido retomar o papo?"

[link HubSpot do deal]
```

> ⚠️ O gancho da mensagem usa o material baixado como
> pretexto — é um sinal de intenção real, não inferência.
> Tom deve ser leve: não citar que está monitorando,
> apenas mencionar o material naturalmente.

> ⚠️ Se `transicao_carteira = nao` (conta do Luciano):
> adaptar a mensagem para incluir a apresentação antes
> de citar o material.

**0.5 — Registrar na planilha**
Após enviar a DM:
- `data_ultima_mensagem` → data de hoje
- `contador` → +1
- `ultimo_gancho` → `material_baixado: [nome do material]`

---

### Passo 1 — Separar deals por vendedor e urgência

A partir do output da Skill 5, separe:

**Urgentes** (DM imediata):
- `levantou_a_mao = true` com `decisao = reativar`
- Renovação 30–60 dias com `decisao = reativar`

**Painel mensal** (todos os demais):
- `decisao = reativar` com score ≥ 5
- `decisao = nutrir_ativo` com score 3–4
- `decisao = nutrir_leve` com score 1–2
- `decisao = descartar` → NÃO aparece no painel

Agrupe por `vendedor` para gerar um artefato por pessoa.

---

### Passo 2 — Gerar o artefato HTML

Para cada vendedor, gere um artefato HTML com a estrutura:

**Cabeçalho:**
- Nome do vendedor + mês de referência
- Três métricas: agir agora · manter presença · total no radar

**Cards por deal** (ordenados por score decrescente):
- Badge de prioridade colorido (🔴 vermelho, 🟢 verde, 🟡 âmbar)
- Avatar com iniciais do contato
- Nome, cargo, empresa, dias desde a perda
- Seção colapsada (oculta por padrão) com:
  - Contexto do deal (por que esse gancho)
  - Mensagem pronta (email ou WhatsApp)
  - Botão "Copiar" funcional
  - Botões "Ajustar tom ↗" e "Versão WhatsApp ↗"
    (ambos usam `sendPrompt()` para acionar o Claude)
  - Link HubSpot do deal

**Rodapé:**
- "gerado pelo assistente de nutrição Pipo · [data]"
- Link "marcar todas como enviadas" via `sendPrompt()`

**Especificações visuais:**
```
Borda esquerda por prioridade:
  🔴 REATIVAR        → #E24B4A (vermelho)
  🟢 LEVANTOU A MÃO  → #0F6E56 (verde)
  🟡 NUTRIR ATIVO    → #BA7517 (âmbar escuro)
  ⬜ NUTRIR LEVE     → var(--color-border-secondary) (neutro)

Badge de score:
  REATIVAR        → bg #FCEBEB, texto #A32D2D
  LEVANTOU A MÃO  → bg #E1F5EE, texto #085041
  NUTRIR ATIVO    → bg #FAEEDA, texto #633806
  NUTRIR LEVE     → bg var(--color-background-secondary)

Cards colapsados por padrão — botão "Ver mensagem" /
"Fechar" alterna visibilidade via JS.

Botão "Copiar" usa navigator.clipboard.writeText()
e muda para "Copiado! ✓" por 2 segundos.
```

---

### Passo 3 — Enviar DMs urgentes

Para cada deal urgente identificado no Passo 1, envie DM
individual para o vendedor responsável usando `slack_send_message`
com o `owner_slack_id` do deal.

Formato da mensagem:
```
🚨 *[Empresa] — [motivo da urgência]*

*[Nome do contato]* · [Cargo]
*Renovação:* [mês] · *Score:* [N]/10

*Mensagem sugerida ([canal]):*
```
[assunto se email]
[corpo da mensagem]
```
_[link HubSpot]_
```

Regra: máximo 1 DM urgente por deal por ciclo. Verificar
na planilha se já houve envio recente para evitar duplicata.

---

### Passo 4 — Enviar DM do painel mensal

Para cada vendedor com deals no painel, envie uma DM usando
`slack_send_message`:

```
Bom dia, [primeiro nome]! 🗓️
Seu painel de nutrição de *[Mês/Ano]* está pronto.

*[N] para agir agora · [N] para manter presença*

Acesse aqui → [link do artefato Claude]
```

> ⚠️ O link do artefato é gerado pelo Claude quando o
> artefato é criado na conversa. Use o URL da conversa
> atual ou instrua o usuário a compartilhar o link
> gerado pelo Claude com o vendedor.

---

### Passo 5 — Atualizar planilha de controle

Após cada envio, atualize a planilha de nutrição:
- `data_ultima_mensagem` → data de hoje (DD/MM/AAAA)
- `contador` → incrementar +1
- `ultimo_gancho` → tipo de gancho usado

Use `update_file_content` no Google Drive com o file ID
da planilha de nutrição.

---

### Passo 6 — Retornar resumo de execução

```
Ciclo              : [Mensal | Urgente]
Data               : [DD/MM/AAAA]
Vendedores notificados : [N]
DMs urgentes enviadas  : [N]
Painéis gerados        : [N]
Deals no painel        :
  Marcelo Silva    : [N] deals
  Breno Haus       : [N] deals
  Luciano Moreira  : [N] deals
  Aline Furlaneto  : [N] deals
Planilha atualizada : ✅
```

---

## Regras importantes

- **Uma DM por vendedor por ciclo mensal.** Nunca enviar
  múltiplas mensagens no mesmo dia para o mesmo vendedor.
- **DM urgente é exceção, não regra.** Só para levantada
  de mão e renovação em 30–60 dias. Não transformar em flood.
- **Verificar duplicatas antes de enviar.** Se `data_ultima_mensagem`
  for hoje, pular o deal.
- **Deals com `decisao = descartar` não aparecem no painel.**
  O vendedor não precisa saber — apenas não recebe o card.
- **Contas do Luciano com `transicao_carteira = nao`:**
  adicionar no card uma nota discreta em cinza:
  *"Primeira abordagem — use a abertura de apresentação"*
- **Tom da DM de aviso deve ser leve e direto.** Não corporativo,
  não formal demais. O vendedor precisa querer clicar.
- **Artefato é gerado inline na conversa Claude.** O link
  para compartilhar com o vendedor é o link da conversa
  atual ou um artefato persistente salvo no Drive.
