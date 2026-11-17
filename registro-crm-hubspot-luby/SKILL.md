---
name: "registro-crm-hubspot-luby"
description: "Varre e-mails e conversas no Claude do vendedor e propõe o registro no HubSpot nos funis de venda BR e US da Luby (regras da GMUD de nov/2026); só grava após aprovação humana. Use quando o vendedor pedir para atualizar o CRM, registrar reuniões ou o FUP no HubSpot."
---

# Registro no CRM (HubSpot) — funis de venda BR e US da Luby, com humano no loop

Use quando o vendedor pedir "atualiza meu CRM", "registra minhas reuniões no HubSpot", "o que eu preciso lançar no CRM?", "varre meus e-mails e preenche o HubSpot", "FUP do CRM da semana", ou mencionar uma conversa com cliente que precisa ir para o HubSpot.

Objetivo: o vendedor não digita. O Claude lê e-mails e conversas, monta o registro no padrão dos **funis de venda BR e US** do HubSpot e mostra tudo para aprovação. **Nada é gravado sem o OK explícito da pessoa.** A IA preenche, confere e alerta; a pessoa decide etapa, valor, ganho e perda.

Base das regras: Funil do Cliente (Matheus Pedralli, Growth e Receita Brasil, out/2026), ajustado e implantado pela GMUD HubSpot (Ondas 1 e 2, nov/2026). Em 10/11/2026 os negócios de 2026 foram migrados do funil antigo "Luby" para os funis BR e US.

---

## 0. Preparação (toda execução, em silêncio)

1. **Quem é o vendedor:** `get_user_details` no HubSpot → guarde o `ownerId`. Todo registro novo sai com esse dono (`hubspot_owner_id`). Não use `account_executive`, que é legado.
2. **Permissão:** `discover_hubspot_schema` (GET_OBJECT_TYPES para DEAL, CONTACT, COMPANY, NOTE, TASK). Se não houver writeAccess, avise e pare na proposta (entregue a lista para ele lançar à mão).
3. **Conferir os dois funis:** `discover_hubspot_schema` (GET_PIPELINES, DEAL). Confira se BR e US continuam com os IDs e as etapas da tabela abaixo. Se algo mudou (etapa nova, renomeada ou com outro ID), **não improvise**: avise o vendedor e a admin do HubSpot (Karime) e pare na proposta.
4. **Opções dos campos de lista:** `get_properties` dos campos de seleção que você vai gravar (seção 4). Grave sempre o **valor interno** (`value`) da opção, nunca o rótulo que aparece na tela. Em vários campos eles são diferentes. Exemplos em `canal_de_origem`: o value `Linkedin Orgânico` aparece como "Linkedin Ads", `linkedin_organico` aparece como "Linkedin Orgânico", `Sites de Referência` aparece como "Landing Page" e `Stand Evento` aparece como "Stand/Patrocínio Evento". Em `bu`, `Alocação` aparece como "Outsourcing US". Listas como evento e parceiro crescem com o tempo: opção que não existir não é inventada.
5. **Período:** desde a última execução que o vendedor informar; padrão = últimos 7 dias.

### Só os funis de venda BR e US

| Funil | ID | Quando usar | Moeda (`deal_currency_code`) |
|---|---|---|---|
| **BR** | `960560941` | Cliente no Brasil, contrato em reais | `BRL` |
| **US** | `953928114` | Cliente nos EUA, contrato em dólar | `USD` |

O HubSpot não define moeda por funil: informe a moeda **sempre** ao criar o negócio. Na dúvida sobre o funil, pergunte ao vendedor.

Os outros funis do portal ficam fora desta skill:
- **"Luby"** (`855703866`): só histórico de antes de 2026. Nunca crie nem mova negócio para ele. Negócio aberto encontrado lá é alerta.
- **"Leads BR", "Leads US" e "Prospecção":** não crie, não mova, não atualize. Se a empresa tiver negócio neles, cite no cabeçalho da revisão (para não duplicar) e siga.

## 1. Coleta das fontes

Use só os conectores que o vendedor tiver. Se uma ferramenta citada aqui não existir no ambiente, siga com as fontes disponíveis e diga qual ficou de fora. Fontes principais:

- **Gmail:** `search_threads` com `newer_than:7d -category:promotions -category:social -category:updates` (inclui enviados). Fique só com threads em que há pelo menos um participante de domínio **externo** à Luby (luby.com.br, luby.co) e que não sejam newsletter, notificação ou convite automático. Leia com `get_thread` (PLAIN_TEXT). Convites de reunião aceitos contam como sinal de reunião marcada.
- **Conversas no Claude:** `recent_chats` no período e `conversation_search` com o nome de cada empresa encontrada no Gmail ou citada pelo vendedor. Aproveite só fatos comerciais que **o vendedor** escreveu ("falei com o CFO da X, querem proposta até dia 15"). Sugestão do Claude em conversa antiga não é fato do cliente.
- **Opcionais, se conectados:** Google Calendar (confirma se a primeira reunião aconteceu e quem estava), Plaud ou Fireflies (transcrição da reunião).
- **O que o vendedor disser agora** no chat também é fonte (ele pode ditar um resumo).

Agrupe tudo **por empresa** (domínio do e-mail). Ignore assuntos pessoais, de RH e internos sem cliente.

## 2. Cruzamento com o HubSpot

Para cada empresa:
1. Empresa por domínio (`search_crm_objects` COMPANY, `domain`). Se não existir, proponha criar.
2. Contatos por e-mail. Os que não existirem, proponha criar, associados à empresa, com cargo.
3. Negócios abertos da empresa **nos funis BR e US** (excluindo Ganho e Perdido). Se houver mais de um, pergunte a qual o assunto se refere, ou use a oferta para desempatar.
4. Leia os campos atuais do negócio para mostrar "atual → proposto", inclusive os obrigatórios da etapa atual que estiverem vazios (muitos negócios migrados em 10/11/2026 vieram com pendências).
5. Anote negócios da mesma empresa em outros funis (Leads, Prospecção, Luby) só para citar.

Classifique cada empresa em um destes casos:

| Situação encontrada | O que propor |
|---|---|
| Sem reunião feita ainda (só troca de e-mail, resposta, indicação, evento) | **Não criar negócio.** Empresa com Tipo (`type`) = Prospect e contato em Lead no ciclo de vida (`lifecyclestage`); registrar o sinal de interesse e a próxima tarefa. Nunca rebaixe o ciclo de vida de quem já está à frente (ex.: Cliente) |
| Primeira reunião aconteceu e não há negócio | Propor **negócio novo em Qualificação** no funil BR ou US (sempre na primeira etapa, nunca direto em proposta ou ganho), associado a empresa e contatos. Lembrar que o SQL depende do aceite de accounts |
| Negócio aberto existente (BR ou US) | Nota, tarefas, campos e, se os critérios forem cumpridos, sugestão de **uma** etapa à frente |
| Cliente da base (renovação, expansão, reajuste, reposição) | **Mesmo funil BR ou US**, com `natureza_da_venda` = Renovação, Upsell, Cross-sell ou Reposição. Não existe funil Base |

## 3. Etapas, critérios e obrigatórios

Etapa é **decisão do cliente**, não atividade da Luby. Só sugira avançar se o critério de saída tiver **evidência** nas fontes e os obrigatórios da etapa de destino estiverem preenchidos (ou forem preenchidos agora). Crie sempre em Qualificação. Nunca pule etapa. O HubSpot permite voltar etapa, mas só proponha isso com evidência e perguntando.

| # | Etapa (prob.) | ID BR | ID US | Entra quando | Sai quando (evidência) | O HubSpot exige para estar na etapa |
|---|---|---|---|---|---|---|
| 1 | Qualificação (10%) | `1484148018` | `1473224705` | Primeira reunião feita e aceite de accounts | Conta no ICP (tier e porte conhecidos), dor e gatilho identificados, janela de compra até 12 meses | Nome, funil, etapa, dono, `origem_negocio`, `canal_de_origem` (+ condicionais de canal), empresa e pelo menos 1 contato associados |
| 2 | Descoberta (20%) | `1484148019` | `1473224706` | Dor confirmada pelo cliente | Impacto quantificado, sponsor identificado, processo de decisão conhecido, 2+ contatos | Nada a mais. `amount` continua **vazio** |
| 3 | Solução e Escopo (40%) | `1484148020` | `1473224707` | Go/no-go = **Go** (decisão da Luby após a Descoberta) | Escopo e premissas validados **por escrito** pelo cliente | `tipo_de_oferta`, `bu`, `modelo_comercial`, `decisao_go_no_go` |
| 4 | Proposta Apresentada (75%) | `1484148021` | `1473224708` | Proposta apresentada **ao decisor econômico** | Aprovação formal do cliente (e-mail ou ata; `aprovacao_do_comite` = Aprovada) e minuta enviada | `deal_currency_code`, `amount`, `data_provavel_de_fechamento`, `proposta_enviada_arquivo`, `tipo_de_oferta`, `meses_de_contrato`, `modelo_comercial`, `margem_contribuicao`, `receita_mensal`, `sinal_de_risco` |
| 5 | Contrato em Assinatura (90%) | `1484148022` | `1473224709` | Minuta enviada | Contrato assinado pelas duas partes | Moeda, `amount`, `data_de_envio_da_minuta`, data provável, meses, receita mensal, margem, `sinal_de_risco` |
| 6 | Ganho (100%) | `1484148023` | `1473224710` | Contrato assinado pelas duas partes e anexado | Passagem ao delivery no kick-off | Moeda, `amount`, `tipo_de_oferta`, `bu`, `proposta_enviada_arquivo`, meses, `origem_negocio`, `natureza_da_venda`, `contrato_anexado` |
| — | Perdido (0%) | `1484148024` | `1473224711` | Cliente desiste, some ou a Luby encerra (No-go) | — | `motivo_de_perda`, `etapa_em_que_perdeu`, `tipo_de_oferta`, `bu` |

Não existe etapa de Comitê: a aprovação interna do cliente acontece dentro de Proposta Apresentada e é registrada em `aprovacao_do_comite`.

**Obrigatórios condicionais:**
- `canal_de_origem` = Indicação Parceiro → `parceiro_responsavel` e `tipo_de_parceria` obrigatórios.
- `canal_de_origem` = Stand/Patrocínio Evento ou Networking Evento → `evento` obrigatório.
- `canal_de_origem` = Indicação Cliente ou Indicação Luber → `quem_indicou` (opcional; preencha se souber).
- `decisao_go_no_go` = Go, No-go ou Reavaliar → `nota_de_prontidao` e `evidencias_da_nota_de_prontidao` obrigatórias.
- `decisao_do_deal_review` preenchida → `data_do_deal_review` obrigatória.
- `aprovacao_do_comite` = Aprovada → `plano_de_acao_mutuo` aparece (opcional).
- `motivo_de_perda` = Optou por concorrente → `concorrente` obrigatório.

**O HubSpot também barra, inclusive pelo conector.** Criação ou mudança de etapa sem os obrigatórios é recusada inteira. Por isso a skill confere antes, mostra as lacunas e manda os obrigatórios na mesma gravação da mudança de etapa. Se ninguém souber um obrigatório (ex.: oferta e BU numa perda precoce), registre a lacuna e avise a admin. **Nunca preencha valor fictício só para passar na trava.**

**Arquivos:** `proposta_enviada_arquivo` (proposta) e `contrato_anexado` (contrato assinado) são campos de **upload de arquivo**. O conector não sobe arquivo: peça ao vendedor que anexe na tela do negócio antes de propor Proposta Apresentada ou Ganho, e confira que o campo está preenchido.

**No-go:** com `decisao_go_no_go` = No-go, não sugira avançar. Proponha Perdido (com motivo) ou pergunte ao vendedor. "Pendente" não serve para entrar em Solução e Escopo.

**Comitê do cliente:** proponha o **Buying Role** (`hs_buying_role`) de cada contato com base no que ele faz na conversa (um contato pode ter mais de um papel): decisor → `DECISION_MAKER`, dono do orçamento → `BUDGET_HOLDER`, sponsor → `EXECUTIVE_SPONSOR`, champion → `CHAMPION`, técnico/influenciador → `INFLUENCER`, usuário → `END_USER`, jurídico → `LEGAL_AND_COMPLIANCE`, contra → `BLOCKER`, compras e outros → `OTHER`. Referência de comitê mínimo para passar da proposta (ainda em homologação, use como **alerta**, não como bloqueio): enterprise 6 pessoas (sponsor, decisor, champion, técnico, compras, jurídico); médio 4; pequeno 2 (decisor e técnico).

**Nota de prontidão (0–100)**, calculada sempre que houver negócio, com a evidência de cada dimensão: ICP e perfil da conta 20 · Dor e impacto 20 · Decisor e comitê 20 · Orçamento e janela 15 · Solução e capacidade 15 · Concorrência e critério 10. Sem evidência, a dimensão vale 0 e entra como lacuna. Proponha gravar a nota em `nota_de_prontidao` e o resumo das evidências em `evidencias_da_nota_de_prontidao`. Referências em homologação (**alerta, não bloqueio**): abaixo de 50, fica em Qualificação/Descoberta com plano; 70+ para o go/no-go; 80+ para virar proposta; forecast só com 70+.

### Governança (Onda 2): a skill registra, não decide

- **Go/no-go** (`decisao_go_no_go`: Pendente, Go, No-go, Reavaliar): decisão interna da Luby após a Descoberta. Só registre decisão que já aconteceu e que o vendedor confirmou.
- **Comitê do cliente** (`aprovacao_do_comite`: Não iniciada, Em aprovação, Aprovada, Reprovada, Não se aplica): Aprovada só com evidência escrita do cliente (e-mail ou ata).
- **Deal review** (`data_do_deal_review` + `decisao_do_deal_review`: Aprovada, Aprovada com ajustes, Reavaliar, Encerrar oportunidade): revisão interna da Luby (Jan, CTO e financeiro). Só registre revisão que aconteceu. O critério de quando fazer (acima de R$ 500 mil ou enterprise) ainda está em homologação: use como alerta.
- **Sinal de risco** (`sinal_de_risco`: Sem risco, Risco baixo, Risco médio, Risco alto): obrigatório a partir da Proposta. Proponha atualizar a cada execução, com o motivo. Não converta o legado `temperatura_do_negocios`.
- `gatilho_de_compra`, `criterios_de_decisao` e `plano_de_acao_mutuo` (link do plano combinado com o cliente): texto, preencha só com evidência.

## 4. O que extrair (ficha do negócio)

Preencha só com evidência, usando os nomes internos abaixo.

- **Empresa associada** (sempre): domínio, setor (`industry`), porte (`numberofemployees`), `tier` (Tier 1, Tier 2, Tier 3; as faixas por funcionários ainda estão em definição) e `nota_icp` (0–100; a rubrica ainda está em homologação: proponha com a evidência e diga isso). Não confunda com `hs_ideal_customer_profile`. O `canal_de_origem` da **empresa** é o canal do primeiro contato com a conta e não substitui o canal do negócio.
- **Contatos** com cargo (`jobtitle`), e-mail e papel na compra (`hs_buying_role`).
- **Funil e moeda:** BR com `BRL` ou US com `USD` (`deal_currency_code`).
- **Valor** (`amount`, valor total do contrato): **vazio até a proposta**; obrigatório em Proposta Apresentada. Estimativa feita antes da proposta vai só na nota. **Nunca R$ 0 ou R$ 1 como marcador.**
- **Datas:** previsão de assinatura em `data_provavel_de_fechamento` (só aceita **dia útil**; se a data cair no fim de semana, proponha a segunda-feira seguinte e diga). `closedate` é só o fechamento real: em Ganho, a data da assinatura; em Perdido, a data da perda. **Nunca grave `closedate` em negócio aberto.**
- **Origem do negócio** (`origem_negocio`): Inbound ou Outbound (quem deu o primeiro passo).
- **Natureza da venda** (`natureza_da_venda`): Nova aquisição, Upsell, Cross-sell, Renovação, Reposição.
- **Canal de origem** (`canal_de_origem`): lista fechada, **nunca "Outro"** (está oculto e só existe para o histórico). Se não souber, pergunte. Atenção aos condicionais da seção 3.
- **Parceiro** (`parceiro_responsavel` + `tipo_de_parceria` no negócio) ou **quem indicou** (`quem_indicou`).
- **Oferta** (`tipo_de_oferta`, evite "Outro"), **BU** (`bu`) e **modelo comercial** (`modelo_comercial`: Alocação, Escopo fechado, Managed squad, Formato misto).
- **Contrato:** `meses_de_contrato` (texto que aceita só número inteiro, ex.: `12`), `receita_mensal` (número) e `margem_contribuicao` (número em **fração**: 42% → `0.42`, como os negócios atuais estão gravados). **Nunca `0.0001` como marcador.**
- **Documentos:** `proposta_enviada_arquivo` e `contrato_anexado` (upload pelo vendedor; veja a seção 3) e `data_de_envio_da_minuta`.
- **MEDDICC resumido:** dor e impacto em reais, gatilho de compra, decisor econômico, sponsor, champion, critérios e processo de decisão, orçamento, janela, concorrentes, objeções.
- **Próximos passos** com dono, data e o que o cliente se comprometeu a fazer.
- **Perda:** `motivo_de_perda` da lista (evite "Outro"), `etapa_em_que_perdeu`, `concorrente` (obrigatório se o motivo for "Optou por concorrente").

**Não preencha:**
- Campos de pós-venda: faturamento (`valor_faturado_ano_atual`, `valor_faturado_ano_post`, início e fim do faturamento), `health_score_conta` (é do Delivery/CS) e churn (`cliente_churnado_` e seus dependentes). São do delivery e do CS.
- Campos legados: `origem_do_negocio`, `new_client`, `proposta_enviada`, `margem_de_contribuicao`, `temperatura_do_negocios`, `account_executive`, `closed_lost_reason`.

## 5. Revisão humana (obrigatória)

Antes de gravar qualquer coisa, mostre por empresa:

**Cabeçalho:** Empresa · negócio (funil BR/US e etapa atual) · nota de prontidão com 1 linha de justificativa · obrigatórios faltando na etapa atual · negócios da empresa em outros funis, se houver · fontes usadas (assunto e data do e-mail, nome da conversa).

**Bloco A — registro de atividade** (baixo risco, aprovado em lote):
- Nota da reunião ou da troca de e-mails (modelo abaixo).
- Tarefas de follow-up com dono e data.
- Contatos e empresa novos, associações, cargo e buying role.

**Bloco B — decisões** (uma linha por item, aprovação item a item):

| # | Objeto | Campo | Atual | Proposto | Evidência | Confiança |
|---|---|---|---|---|---|---|

Sempre no Bloco B: criar negócio, funil, moeda, mudar etapa, valor, data provável, data de fechamento, oferta, BU, modelo comercial, origem, natureza, canal, nota de prontidão, go/no-go, comitê, deal review, sinal de risco, ganho e perda.

**Lacunas:** obrigatórios da etapa atual e da próxima que continuam vazios, com a pergunta que o vendedor deve fazer ao cliente. Inclua os arquivos que o vendedor precisa anexar na tela.

Feche com: "Aprova o Bloco A? E quais itens do Bloco B (ex.: 1, 3, 4)? Pode corrigir qualquer valor." Aceite edições ("o valor é 380 mil", "a data é 30/11") e mostre a versão final antes de gravar se algo mudou.

No primeiro uso, deixe a pessoa dispensar só a reconfirmação do Bloco A na conversa. O Bloco B sempre pede OK.

## 6. Gravação no HubSpot

Só depois do OK, com `manage_crm_objects` (confirmationStatus CONFIRMED), no máximo 10 objetos por chamada:
1. Empresa e contatos (criar ou atualizar) e associações.
2. Negócio: criar em Qualificação já com nome, funil, dono, moeda, `origem_negocio`, `canal_de_origem` (e os condicionais) e associado à empresa e ao(s) contato(s) na mesma chamada; ou atualizar os campos aprovados.
3. Nota (objeto NOTE, `hs_note_body` e `hs_timestamp`) associada ao negócio, à empresa e aos contatos.
4. Tarefas (TASK: `hs_task_subject`, `hs_task_body`, `hs_timestamp` = vencimento, `hubspot_owner_id`) associadas ao negócio.
5. Etapa por último, se aprovada, **na mesma gravação** dos obrigatórios da etapa de destino que ainda faltarem (em Ganho e Perdido, junto com o `closedate` aprovado).

Se o HubSpot recusar, diga qual campo ele pediu, o que foi gravado e o que não foi, e pergunte ao vendedor. Nunca repita uma gravação que pode já ter acontecido sem conferir antes.

O conector só **cria** associações. Associação errada precisa ser removida pelo vendedor na tela do negócio: avise.

### Modelo da nota

```
[Registro assistido por IA — revisado por <vendedor> em <data>]
Fonte: <e-mail "assunto" de dd/mm | reunião dd/mm | conversa no Claude>

Resumo: <3 a 5 linhas do que o cliente disse e decidiu>
Dor e impacto: <...>
Gatilho de compra: <...>
Decisor / sponsor / champion: <nomes e cargos>
Orçamento e janela: <...> (estimativa de valor antes da proposta fica aqui, não no campo Valor)
Concorrentes e critérios: <...>
Objeções: <...>
Próximos passos: <ação — dono — data>
Nota de prontidão: <nn>/100 (ICP x, dor x, comitê x, orçamento x, solução x, concorrência x)
Sinal de risco: <nível> — <motivo>
Lacunas para a próxima conversa: <...>
```

## 7. Resumo final para o vendedor

Curto: o que foi gravado (por empresa), o que ficou pendente de decisão, as lacunas para a próxima conversa com o cliente, os arquivos a anexar e os alertas da reunião de segunda que se aplicam a ele:
- Negócio sem atividade há mais de 21 dias.
- Data provável de fechamento vencida.
- Obrigatórios da etapa atual vazios.
- Negócio em Proposta Apresentada ou Contrato em Assinatura sem valor; valor 0 ou 1 ou margem 0,0001 em qualquer etapa (marcador falso).
- Negócio US em BRL, BR em USD ou sem moeda.
- Negócio sem empresa ou sem contato associado.
- Dono desativado em negócio aberto.
- Negócio aberto no funil antigo "Luby".
- Go/no-go = No-go com o negócio ainda avançando.
- Referências em homologação (alerta, não bloqueio): proposta com nota de prontidão abaixo de 80; comitê abaixo do mínimo por porte; negócio acima de R$ 500 mil (ou enterprise) sem deal review.

## Regras que nunca mudam

- **Nada vai para o HubSpot sem aprovação humana.** Ganho, perda, etapa e valor sempre item a item.
- **Só os funis BR e US.** Nunca crie nem mova negócio para Luby, Leads BR, Leads US ou Prospecção.
- **Ganho só com contrato assinado** pelas duas partes e anexado (`contrato_anexado`). Início de faturamento não é ganho.
- **Negócio só nasce depois da primeira reunião**, sempre em Qualificação, com empresa e contato associados, dono, moeda, origem e canal.
- **Uma etapa por vez**, com critério cumprido, evidência citada e obrigatórios preenchidos.
- **Valor só a partir da proposta; previsão em data provável; `closedate` só no fechamento.**
- **Sem palpite:** dado sem evidência fica vazio e vira pergunta. Nada de R$ 0 ou R$ 1, margem 0,0001, canal, oferta ou motivo "Outro", data inventada ou campo preenchido só para passar na trava do HubSpot.
- **Fato do cliente ≠ sugestão do Claude.** Só entra o que o cliente ou o vendedor disseram.
- **Decisões internas são de pessoas:** go/no-go, deal review e aprovação do comitê são registrados, nunca decididos pela skill.
- **LGPD e discrição:** registre só informação comercial e resuma (não cole e-mails inteiros). Nada de dado pessoal sensível de contatos. Reunião gravada exige aviso ao cliente.
- Respeite o que o portal tem: se um funil, etapa, campo ou opção não existir, não crie configuração. Isso é com a admin do HubSpot (Karime); registre na nota e avise o vendedor.
