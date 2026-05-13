---
name: feedback-aprendizado
description: |
  Coleta feedback dos vendedores sobre as mensagens enviadas e
  incorpora o aprendizado no próximo ciclo de nutrição. Roda em
  dois momentos: (1) follow-up automático 7 dias após o envio
  para coletar feedback, (2) no início de cada ciclo mensal para
  consolidar o aprendizado antes de gerar novas mensagens.

  Use esta skill SEMPRE que:
  - Precisar enviar DMs de follow-up para coletar feedback
  - O vendedor clicar em "dar feedback" no painel
  - O orquestrador solicitar o resumo de aprendizado do ciclo anterior
  - Você quiser entender o que está funcionando nas mensagens

  Recebe: lista de deals com data_ultima_mensagem preenchida
  Entrega: DMs de follow-up + planilha atualizada + resumo de aprendizado
---

# Feedback e Aprendizado Contínuo

Você é responsável por fechar o loop do sistema. Sem feedback,
o módulo repete os mesmos padrões independente dos resultados.
Com feedback, cada ciclo fica mais preciso que o anterior.

---

## Configuração fixa

**Planilha de nutrição:** `1KvqrV9YY2kjZbt0_EB9tOkqs9tsSqQslUyVYaURXDuU`
**Janela de follow-up:** 7 dias após `data_ultima_mensagem`

### Vendedores e Slack IDs

| Vendedor | Slack User ID |
|----------|--------------|
| Aline Furlaneto | `U03GFTJ649K` |
| Marcelo Silva | `U05EW1UF1C7` |
| Breno Haus | `U04MHL72USV` |
| Luciano Nascimento Moreira | `U035X1LGBAP` |

---

## Estrutura de feedback na planilha

Cada deal tem 6 campos de feedback:

| Campo | Valores possíveis |
|-------|-----------------|
| `feedback_resultado` | `gerou_resposta` \| `nao_respondeu` \| `aguardando` \| `nao_enviou` |
| `feedback_tom` | `adequado` \| `muito_formal` \| `muito_generico` \| `muito_agressivo` \| `outro` |
| `feedback_gancho` | `adequado` \| `gancho_errado` \| `persona_mudou` \| `contexto_desatualizado` \| `outro` |
| `feedback_texto_livre` | texto livre do vendedor |
| `feedback_data` | DD/MM/AAAA |
| `aprendizado_aplicado` | `sim` \| `nao` |

---

## Procedimento de execução

### MODO 1 — Feedback imediato (logo após o envio)

O feedback deve ser solicitado imediatamente após o
vendedor copiar ou enviar a mensagem — não dias depois.
Quando o contexto está fresco, o feedback é mais preciso.

**1.1 — Trigger: vendedor clica em "Copiar" ou "Enviei" no painel**

No artefato HTML, cada card tem dois botões após a mensagem:

```
[Copiar mensagem]   ← ao clicar, marca como "copiada"
[Já enviei ✓]       ← ao clicar, abre mini-formulário inline
```

Quando o vendedor clica em "Já enviei", o painel exibe
imediatamente um mini-formulário colapsado abaixo da mensagem:

```
Como você avaliaria essa mensagem?

Tom:     [👍 Adequado]  [✏️ Ajustei bastante]  [❌ Precisou reescrever]
Gancho:  [🎯 Certeiro]  [🤔 Genérico]  [❌ Errado para esse contexto]

Comentário (opcional): ________________

[Salvar feedback]
```

O preenchimento é opcional — se o vendedor só clicar em
"Salvar" sem preencher, registra apenas `feedback_resultado
= enviou` e segue.

**1.2 — Follow-up leve após 3 dias (só se não houve resposta)**

Se após 3 dias o deal não tem `feedback_resultado =
gerou_resposta`, envie UMA DM curta e direta:

```
Oi [nome]! Você enviou mensagem para *[Empresa]* há 3 dias.
Teve retorno? [Sim ✅]  [Não ainda ❌]
```

Um clique. Sem fricção. Não enviar se já tiver feedback.

> ⚠️ Máximo 1 follow-up por deal. Se não responder
> em 7 dias, registrar `feedback_resultado = sem_retorno`
> e não enviar mais DMs sobre esse ciclo.

**1.3 — Processar resposta do vendedor**

Quando o vendedor responder (via Slack ou artefato), extraia:
```
deal_identificado  : [empresa mencionada]
feedback_resultado : [gerou_resposta | nao_respondeu | aguardando | nao_enviou]
feedback_tom       : [se mencionado]
feedback_gancho    : [se mencionado]
feedback_livre     : [texto adicional]
```

Atualize a planilha imediatamente com os valores coletados
e registre `feedback_data = [hoje]`.

---

### MODO 2 — Consolidação de aprendizado (início do ciclo mensal)

Roda antes da Skill 2 no Fluxo A do orquestrador.

**2.1 — Ler todos os feedbacks do ciclo anterior**

Na planilha, filtre deals onde:
- `feedback_data` está preenchida
- `aprendizado_aplicado = nao`

**2.2 — Identificar padrões**

Analise os feedbacks e produza um resumo estruturado:

```
PADRÕES POSITIVOS (o que funcionou):
  - Ganchos que geraram resposta
  - Tons que foram bem recebidos
  - Contextos que ressoaram

PADRÕES NEGATIVOS (o que não funcionou):
  - Ganchos que não geraram resposta (> 60% de não resposta)
  - Tons mencionados como inadequados
  - Personas que mudaram e invalidaram o contexto

AJUSTES SUGERIDOS PARA O CICLO ATUAL:
  - [lista de ajustes concretos]

DADOS DO CICLO ANTERIOR:
  - Total de mensagens enviadas : N
  - Taxa de resposta            : X%
  - Gancho com melhor resultado : [tipo]
  - Gancho com pior resultado   : [tipo]
  - Vendedor com maior taxa     : [nome] (X%)
```

**2.3 — Alimentar a Skill 4 e Skill 5**

Retorne o resumo de aprendizado para o orquestrador, que
o passará como contexto adicional para:

**Skill 4 (avaliador):** ajustar pesos de score baseado
em histórico de cada deal:
```
Se deal já teve 2+ tentativas com feedback "nao_respondeu":
  → score -= 1 (penalidade adicional por histórico ruim)
  → registrar como "deal resistente"

Se deal teve feedback "gerou_resposta" em ciclo anterior:
  → score += 1 (bônus por receptividade histórica)
  → registrar como "deal receptivo"
```

**Skill 5 (gerador):** instruções específicas baseadas
no feedback anterior do deal:
```
Se feedback_tom = "muito_generico" no ciclo anterior:
  → instrução: "mensagem mais específica, mencionar dado
    concreto do histórico deste deal"

Se feedback_gancho = "gancho_errado":
  → instrução: "não usar [tipo de gancho anterior],
    tentar [próximo gancho disponível]"

Se feedback_tom = "muito_formal":
  → instrução: "tom mais próximo, como WhatsApp"

Se feedback_texto_livre contém observação:
  → instrução: "considerar: [texto livre do vendedor]"
```

**2.4 — Marcar aprendizado como aplicado**

Após processar, atualize `aprendizado_aplicado = sim`
para todos os deals processados.

---

### MODO 3 — Feedback via botões no painel HTML

Quando o vendedor clica em um botão de feedback no artefato
HTML (via `sendPrompt()`), processe a mensagem recebida:

Exemplos de mensagens esperadas:
- "Marcar Sympla como: respondeu"
- "Feedback Keeggo: não respondeu, gancho muito genérico"
- "Sympla respondeu — mensagem foi ótima"

Para cada mensagem:
1. Identifique o deal pelo nome da empresa
2. Extraia os campos de feedback
3. Atualize a planilha via Google Drive
4. Confirme para o usuário: *"✅ Feedback da Sympla registrado!"*

---

## Padrões de aprendizado acumulado

A cada ciclo, o sistema acumula um perfil por tipo de gancho.
Mantenha uma seção de memória de aprendizado:

```
MEMÓRIA DE GANCHOS (atualizar a cada ciclo):

pesquisa_conteudo:
  taxa_resposta_historica : X%
  melhor_para             : [motivos de perda que mais respondem]
  pior_para               : [motivos que raramente respondem]
  observacoes             : [padrões identificados]

evento:
  taxa_resposta_historica : X%
  melhor_para             : [perfis de persona que mais respondem]
  observacoes             : [padrões identificados]

persona_mudanca:
  taxa_resposta_historica : X%
  observacoes             : [padrões identificados]

levantou_a_mao:
  taxa_resposta_historica : X%
  observacoes             : [padrões identificados]

abertura_apresentacao (Luciano):
  taxa_resposta_historica : X%
  observacoes             : [o que funciona melhor como apresentação]
```

---

## Regras importantes

- **Nunca enviar follow-up antes de 7 dias.** O vendedor
  precisa de tempo para enviar a mensagem e aguardar retorno.
- **Uma DM por vendedor, nunca por deal.** Agrupe tudo
  em uma mensagem para não criar flood.
- **Feedback parcial é melhor que nenhum.** Se o vendedor
  responder só o resultado (✅/❌) sem detalhar tom e gancho,
  registrar o que veio e deixar os demais campos vazios.
- **Não pressionar.** Se após 14 dias não houver feedback,
  registrar `feedback_resultado = sem_retorno` e seguir.
- **Aprendizado é incremental.** Com 1 ciclo de dados,
  os ajustes são sugestivos. Com 3+ ciclos, os padrões
  ficam confiáveis o suficiente para ajuste automático
  de pesos de score.
- **Transparência para a gestora.** O resumo de aprendizado
  do Modo 2 sempre deve ser enviado para Aline
  (`U03GFTJ649K`) antes de ser aplicado no ciclo atual.
