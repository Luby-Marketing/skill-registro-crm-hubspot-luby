---
name: "registro-crm-hubspot-luby"
description: "Varre e-mails e conversas no Claude do vendedor e propõe o registro no HubSpot no formato do Funil do Cliente da Luby; só grava após aprovação humana."
---

# Registro no CRM (HubSpot) — Funil do Cliente Luby, com humano no loop

Use quando o vendedor pedir "atualiza meu CRM", "registra minhas reuniões no HubSpot", "o que eu preciso lançar no CRM?", "varre meus e-mails e preenche o HubSpot", "FUP do CRM da semana", ou mencionar uma conversa com cliente que precisa ir para o HubSpot.

Objetivo: o vendedor não digita. O Claude lê e-mails e conversas, monta o registro no padrão do **Funil do Cliente no HubSpot** (Matheus Pedralli, Growth e Receita Brasil, out/2026) e mostra tudo para aprovação. **Nada é gravado sem o OK explícito da pessoa.** A IA preenche, confere e alerta; a pessoa decide etapa, valor, ganho e perda.

---

## 0. Preparação (toda execução, em silêncio)

1. **Quem é o vendedor:** `get_user_details` no HubSpot → guarde o `ownerId`. Todo registro novo sai com esse dono.
2. **Permissão:** `discover_hubspot_schema` (GET_OBJECT_TYPES para DEAL, CONTACT, COMPANY, NOTE, TASK). Se não houver writeAccess, avise e pare na proposta (entregue a lista para ele lançar à mão).
3. **Funis e etapas reais:** `discover_hubspot_schema` (GET_PIPELINES, DEAL). Mapeie pelo nome. Funis esperados: **Brasil**, **US** e **Base** (renovação e expansão). Enquanto o funil Brasil não existir, o funil antigo **"Luby"** (Possible Opportunity, Drafting Proposal, Proposal Sent, Contract, Won, Lost) segue valendo para negócios abertos: não mova negócio entre funis; sugira a etapa nova equivalente só na nota (tabela da seção 3).
4. **Campos disponíveis:** `search_properties` nos negócios, contatos e empresas para os campos da seção 4. Campo que não existe ainda **não é inventado**: o dado vai para o bloco "Ficha Luby" da nota.
5. **Período:** desde a última execução que o vendedor informar; padrão = últimos 7 dias.

## 1. Coleta das fontes

Use só os conectores que o vendedor tiver. Fontes principais:

- **Gmail:** `search_threads` com `newer_than:7d -category:promotions -category:social -category:updates` (inclui enviados). Fique só com threads em que há pelo menos um participante de domínio **externo** à Luby (luby.com.br, luby.co) e que não sejam newsletter, notificação ou convite automático. Leia com `get_thread` (PLAIN_TEXT). Convites de reunião aceitos contam como sinal de reunião marcada.
- **Conversas no Claude:** `recent_chats` no período e `conversation_search` com o nome de cada empresa encontrada no Gmail ou citada pelo vendedor. Aproveite só fatos comerciais que **o vendedor** escreveu ("falei com o CFO da X, querem proposta até dia 15"). Sugestão do Claude em conversa antiga não é fato do cliente.
- **Opcionais, se conectados:** Google Calendar (confirma se a primeira reunião aconteceu e quem estava), Plaud ou Fireflies (transcrição da reunião).
- **O que o vendedor disser agora** no chat também é fonte (ele pode ditar um resumo).

Agrupe tudo **por empresa** (domínio do e-mail). Ignore assuntos pessoais, de RH e internos sem cliente.

## 2. Cruzamento com o HubSpot

Para cada empresa:
1. Empresa por domínio (`search_crm_objects` COMPANY, `domain`). Se não existir, proponha criar.
2. Contatos por e-mail. Os que não existirem, proponha criar, associados à empresa, com cargo.
3. Negócios abertos associados à empresa (excluindo Ganho/Won e Perdido/Lost). Se houver mais de um, pergunte a qual o assunto se refere, ou use a oferta para desempatar.
4. Leia os campos atuais do negócio para mostrar "atual → proposto".

Classifique cada empresa em um destes casos:

| Situação encontrada | O que propor |
|---|---|
| Sem reunião feita ainda (só troca de e-mail, resposta, indicação, evento) | **Não criar negócio.** Empresa em *Prospect* ou contato em *Lead* (estágio do ciclo de vida), registrar o sinal de interesse e a próxima tarefa |
| Primeira reunião aconteceu e não há negócio | Propor **negócio novo em Qualificação** (sempre na primeira etapa, nunca direto em proposta ou ganho), associado a empresa e contatos. Lembrar que o SQL depende do aceite de accounts |
| Negócio aberto existente | Nota, tarefas, campos e, se os critérios forem cumpridos, sugestão de **uma** etapa à frente |
| Cliente da base (renovação, expansão, reajuste) | Funil **Base**, não o de novos negócios |

## 3. Etapas e critérios de passagem (regra da Luby)

Etapa é **decisão do cliente**, não atividade da Luby. Só sugira avançar se o critério de saída tiver **evidência** nas fontes e os campos obrigatórios estiverem preenchidos (ou forem preenchidos agora). Nunca pule etapa, nunca retroceda sem perguntar.

| # | Etapa (prob.) | Entra quando | Sai quando | Campos obrigatórios para avançar |
|---|---|---|---|---|
| 1 | Qualificação (10%) | Primeira reunião feita e aceite de accounts | Conta no ICP, dor e gatilho identificados, janela de compra até 12 meses | Origem e canal, oferta provável, nota de ICP, porte |
| 2 | Descoberta (20%) | Dor confirmada pelo cliente | Impacto quantificado, sponsor identificado, processo de decisão conhecido | Valor estimado, sponsor, data prevista de decisão, 2+ contatos |
| 3 | Solução e escopo (40%) | Go/no-go aprovado (nota ≥ 70); delivery só em exceção | Escopo e premissas validados **por escrito** pelo cliente | Oferta, tipo de contrato, escopo anexado, critérios de decisão |
| 4 | Proposta apresentada (55%) | Proposta apresentada **ao decisor econômico** (deal review se > R$ 500 mil ou enterprise; nota ≥ 80) | Cliente confirma que a proposta segue para aprovação | Valor final, data prevista de assinatura, comitê mapeado com papéis |
| 5 | Comitê do cliente (75%) | Proposta em aprovação interna do cliente | Aprovação formal do comitê (e-mail ou ata) e plano de assinatura | Aprovação registrada, compras e jurídico identificados, plano de ação mútuo |
| 6 | Contrato em assinatura (90%) | Minuta enviada | Contrato assinado pelas duas partes | Data de envio da minuta, jurídico do cliente |
| 7 | Ganho (100%) | Contrato assinado | Passagem ao delivery no kick-off | Data de assinatura, contrato anexado, valor contratado, início previsto |
| — | Perdido | Cliente desiste ou some | — | Motivo (lista fechada), etapa em que perdeu, concorrente se houver |

O funil que existir no portal manda: use as etapas e probabilidades configuradas (o funil US hoje não tem "Comitê do cliente"). Equivalência com o funil antigo "Luby", só para escrever na nota: Possible Opportunity → Lead na empresa, Qualificação ou Descoberta; Drafting Proposal → Solução e escopo; Proposal Sent → Proposta apresentada ou Comitê; Contract → Contrato em assinatura; Won → Ganho (somente com contrato assinado).

**Comitê mínimo para passar da proposta:** enterprise 6 pessoas (sponsor, decisor, champion, técnico, compras, jurídico); médio 4; pequeno 2 (decisor e técnico). Proponha o **Buying Role** (`hs_buying_role`) de cada contato com base no que ele faz na conversa.

**Nota de prontidão (0–100)**, calculada sempre que houver negócio, com a evidência de cada dimensão: ICP e perfil da conta 20 · Dor e impacto 20 · Decisor e comitê 20 · Orçamento e janela 15 · Solução e capacidade 15 · Concorrência e critério 10. Abaixo de 50: fica em Qualificação/Descoberta, com plano. 70+: pode passar o go/no-go. 80+: pode virar proposta. Forecast só com 70+. Sem evidência, a dimensão vale 0 e entra como lacuna.

## 4. O que extrair (ficha do negócio)

Preencha só com evidência. Use os nomes internos quando o campo existir; senão, vá para a "Ficha Luby" na nota.

- **Empresa associada** (sempre), domínio, setor, porte, perfil A a E, nota de ICP.
- **Contatos** com cargo, e-mail e papel na compra.
- **Valor e data de fechamento** (`amount`, `closedate`). Use valor real ou estimado a partir da Descoberta. **Nunca R$ 0 ou R$ 1 como marcador**: sem valor, deixe vazio e aponte como lacuna. Não use "Projected Close Date" (`data_provavel_de_fechamento`), que é legado.
- **Origem do negócio** (`origem_do_negocio`): Inbound, Outbound, Upsell, Cross-sell, Renovação, Reposição.
- **Canal de origem** (`canal_de_origem`): lista fechada, **nunca "Outro"**. Se não souber, pergunte.
- **Parceiro** (`parceiro_responsavel`) ou **quem indicou** (`quem_indicou`).
- **Oferta** (`tipo_de_oferta`) e **tipo de contrato** (`new_client`: Novo ou Vigente), **BU** (`bu`).
- **Modelo comercial** (alocação, escopo fechado, squad por assinatura, resultado), **valor total do contrato, duração e receita mensal** (Ficha Luby enquanto não houver campo).
- **MEDDICC resumido:** dor e impacto em reais, decisor econômico, sponsor, champion, critérios e processo de decisão, orçamento, janela, concorrentes, objeções.
- **Próximos passos** com dono, data e o que o cliente se comprometeu a fazer.
- **Perda:** `motivo_de_perda` da lista (evite "Outro"), etapa em que perdeu, concorrente.

Não preencha campos de pós-venda (início e fim do faturamento, faturado no ano): são do delivery no kick-off.

## 5. Revisão humana (obrigatória)

Antes de gravar qualquer coisa, mostre por empresa:

**Cabeçalho:** Empresa · negócio (funil/etapa atual) · nota de prontidão com 1 linha de justificativa · fontes usadas (assunto e data do e-mail, nome da conversa).

**Bloco A — registro de atividade** (baixo risco, aprovado em lote):
- Nota da reunião ou da troca de e-mails (modelo abaixo).
- Tarefas de follow-up com dono e data.
- Contatos e empresa novos, associações.

**Bloco B — decisões** (uma linha por item, aprovação item a item):

| # | Objeto | Campo | Atual | Proposto | Evidência | Confiança |
|---|---|---|---|---|---|---|

Sempre no Bloco B: criar negócio, mudar etapa, valor, data de fechamento, oferta, canal, ganho e perda.

**Lacunas:** campos obrigatórios da etapa que continuam vazios, com a pergunta que o vendedor deve fazer ao cliente.

Feche com: "Aprova o Bloco A? E quais itens do Bloco B (ex.: 1, 3, 4)? Pode corrigir qualquer valor." Aceite edições ("o valor é 380 mil", "a data é 30/11") e mostre a versão final antes de gravar se algo mudou.

No primeiro uso, deixe a pessoa dispensar só a reconfirmação do Bloco A na conversa. O Bloco B sempre pede OK.

## 6. Gravação no HubSpot

Só depois do OK, com `manage_crm_objects` (confirmationStatus CONFIRMED), no máximo 10 objetos por chamada:
1. Empresa e contatos (criar ou atualizar) e associações.
2. Negócio (criar em Qualificação, ou atualizar campos aprovados).
3. Nota (objeto NOTE, `hs_note_body` e `hs_timestamp`) associada ao negócio, à empresa e aos contatos.
4. Tarefas (TASK: `hs_task_subject`, `hs_task_body`, `hs_timestamp` = vencimento, `hubspot_owner_id`) associadas ao negócio.
5. Etapa por último, se aprovada.

Se algo falhar, diga o que gravou e o que não gravou. Nunca repita uma gravação que pode já ter acontecido sem conferir antes.

### Modelo da nota

```
[Registro assistido por IA — revisado por <vendedor> em <data>]
Fonte: <e-mail "assunto" de dd/mm | reunião dd/mm | conversa no Claude>

Resumo: <3 a 5 linhas do que o cliente disse e decidiu>
Dor e impacto: <...>
Decisor / sponsor / champion: <nomes e cargos>
Orçamento e janela: <...>
Concorrentes e critérios: <...>
Objeções: <...>
Próximos passos: <ação — dono — data>
Nota de prontidão: <nn>/100 (ICP x, dor x, comitê x, orçamento x, solução x, concorrência x)

Ficha Luby (campos ainda sem propriedade no HubSpot):
Modelo comercial: <...> · Valor total: <...> · Duração: <...> · Receita mensal: <...>
Perfil A–E: <...> · Nota de ICP: <...> · Etapa equivalente no funil novo: <...>
```

## 7. Resumo final para o vendedor

Curto: o que foi gravado (por empresa), o que ficou pendente de decisão, as lacunas para a próxima conversa com o cliente e os alertas da reunião de segunda que se aplicam a ele:
- Negócio sem atividade há mais de 21 dias.
- Data de fechamento vencida.
- Proposta ou comitê abaixo do mínimo de contatos.
- Negócio aberto sem valor ou sem oferta.
- Proposta com nota abaixo de 80.

## Regras que nunca mudam

- **Nada vai para o HubSpot sem aprovação humana.** Ganho, perda, etapa e valor sempre item a item.
- **Ganho só com contrato assinado** pelas duas partes (data de assinatura e contrato anexado). Início de faturamento não é ganho.
- **Negócio só nasce depois da primeira reunião**, sempre em Qualificação e sempre com empresa associada e dono.
- **Uma etapa por vez**, com critério cumprido e evidência citada.
- **Sem palpite:** dado sem evidência fica vazio e vira pergunta. Nada de R$ 1, canal "Outro" ou data inventada.
- **Fato do cliente ≠ sugestão do Claude.** Só entra o que o cliente ou o vendedor disseram.
- **LGPD e discrição:** registre só informação comercial e resuma (não cole e-mails inteiros). Nada de dado pessoal sensível de contatos. Reunião gravada exige aviso ao cliente.
- Respeite o que o portal tem: se um funil, etapa ou campo não existir, não crie configuração. Isso é com a admin do HubSpot (Karime); registre na nota e avise o vendedor.