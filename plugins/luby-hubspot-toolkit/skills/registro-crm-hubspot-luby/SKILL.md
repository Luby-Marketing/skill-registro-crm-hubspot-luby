---
name: "registro-crm-hubspot-luby"
description: "Varre e-mails e conversas no Claude do vendedor e propõe o registro no HubSpot da Luby: leads nos funis Leads BR e US, negócios nos funis de venda BR e US (regras da GMUD, do Funil de Leads e do Guia do Account, nov/2026); só grava após aprovação humana. Use quando o vendedor pedir para atualizar o CRM, registrar reuniões ou o FUP no HubSpot."
---

# Registro no CRM (HubSpot) — funis de Leads e de venda BR e US da Luby, com humano no loop

Use quando o vendedor pedir "atualiza meu CRM", "registra minhas reuniões no HubSpot", "o que eu preciso lançar no CRM?", "varre meus e-mails e preenche o HubSpot", "FUP do CRM da semana", ou mencionar uma conversa com cliente que precisa ir para o HubSpot.

Objetivo: o vendedor não digita. O Claude lê e-mails e conversas, monta o registro no padrão dos **funis de Leads e de venda BR e US** do HubSpot e mostra tudo para aprovação. **Nada é gravado sem o OK explícito da pessoa.** A IA preenche, confere e alerta; a pessoa decide conversão, etapa, valor, ganho e perda.

Base das regras: Funil do Cliente (Matheus Pedralli, out/2026), GMUD HubSpot (Ondas 1 e 2 e migração de 10/11/2026), Funil de Leads (Growth, 11/11/2026) e Guia do Account no HubSpot (Growth, 13/11/2026). Se esta skill e o Guia do Account divergirem, pergunte ao Growth antes de gravar.

---

## 0. Preparação (toda execução, em silêncio)

1. **Quem é o vendedor:** `get_user_details` no HubSpot → guarde o `ownerId`. Todo registro novo sai com esse dono (`hubspot_owner_id`). Não use `account_executive`, que é legado.
2. **Permissão:** `discover_hubspot_schema` (GET_OBJECT_TYPES para DEAL, CONTACT, COMPANY, NOTE, TASK). Se não houver writeAccess, avise e pare na proposta (entregue a lista para ele lançar à mão).
3. **Conferir os quatro funis:** `discover_hubspot_schema` (GET_PIPELINES, DEAL). Confira se Leads BR, Leads US, BR e US continuam com os IDs e as etapas das tabelas abaixo. Se algo mudou (etapa nova, renomeada ou com outro ID), **não improvise**: avise o vendedor e o Growth (admin do HubSpot) e pare na proposta.
4. **Opções dos campos de lista:** `get_properties` dos campos de seleção que você vai gravar (seção 5). Grave sempre o **valor interno** (`value`) da opção, nunca o rótulo que aparece na tela. Em vários campos eles são diferentes. Exemplos em `canal_de_origem`: o value `Linkedin Orgânico` aparece como "Linkedin Ads", `linkedin_organico` aparece como "Linkedin Orgânico", `Sites de Referência` aparece como "Landing Page" e `Stand Evento` aparece como "Stand/Patrocínio Evento". Em `bu`, `Alocação` aparece como "Outsourcing US". Em `motivo_de_perda`, `ICP` aparece como "Não tem perfil / ICP". Listas como evento e parceiro crescem com o tempo. **Escolha da lista, não escreva:** se nenhuma opção servir, não escolha uma qualquer; o dado vai para a Ficha Luby (seção 6) e o vendedor avisa o Growth.
5. **Período:** desde a última execução que o vendedor informar; padrão = últimos 7 dias.

### Os quatro funis da skill

| Funil | ID | Para quê |
|---|---|---|
| **Leads BR** | `960685946` | Lead do Brasil, antes de existir oportunidade real |
| **Leads US** | `961640502` | Lead dos EUA, antes de existir oportunidade real |
| **BR** | `960560941` | Negócio (oportunidade real) com cliente do Brasil |
| **US** | `953928114` | Negócio com cliente dos EUA |

Mercado: cliente do Brasil vai para Leads BR e BR; cliente dos EUA, para Leads US e US. Na dúvida, pergunte ao vendedor.

**Moeda (`deal_currency_code`):** `BRL` nos dois funis de venda, US e BR. Não troque para USD. Informe sempre ao criar o negócio. Nos funis de Leads o valor fica sempre vazio.

Fora desta skill: o funil antigo **"Luby"** (`855703866`), que guarda só o histórico de antes de 2026, e o funil **"Prospecção"**. Nunca crie nem mova nada neles. Se a empresa tiver card neles, cite no cabeçalho da revisão (para não duplicar) e siga. Negócio aberto no "Luby" é alerta.

## 1. Coleta das fontes

Use só os conectores que o vendedor tiver. Se uma ferramenta citada aqui não existir no ambiente, siga com as fontes disponíveis e diga qual ficou de fora. Fontes principais:

- **Gmail:** `search_threads` com `newer_than:7d -category:promotions -category:social -category:updates` (inclui enviados). Fique só com threads em que há pelo menos um participante de domínio **externo** à Luby (luby.com.br, luby.co) e que não sejam newsletter, notificação ou convite automático. Leia com `get_thread` (PLAIN_TEXT). Convites de reunião aceitos contam como sinal de reunião marcada. E-mail de prospecção enviado pelo vendedor conta como primeira tentativa de contato.
- **Conversas no Claude:** `recent_chats` no período e `conversation_search` com o nome de cada empresa encontrada no Gmail ou citada pelo vendedor. Aproveite só fatos comerciais que **o vendedor** escreveu ("falei com o CFO da X, querem proposta até dia 15"). Sugestão do Claude em conversa antiga não é fato do cliente.
- **Opcionais, se conectados:** Google Calendar (confirma se a reunião aconteceu, se o lead faltou e quem estava), Plaud ou Fireflies (transcrição da reunião).
- **O que o vendedor disser agora** no chat também é fonte (ele pode ditar um resumo).

Agrupe tudo **por empresa** (domínio do e-mail). Ignore assuntos pessoais, de RH e internos sem cliente.

## 2. Cruzamento com o HubSpot

Para cada empresa:
1. Empresa por domínio (`search_crm_objects` COMPANY, `domain`). Se não existir, proponha criar.
2. Contatos por e-mail. Os que não existirem, proponha criar, associados à empresa, com cargo.
3. Cards abertos da empresa nos **funis de Leads** (fora de Convertido em Deal e Perdido) e negócios abertos nos **funis de venda** BR e US (fora de Ganho e Perdido). Se houver mais de um, pergunte a qual o assunto se refere, ou use a oferta para desempatar.
4. Leia os campos atuais para mostrar "atual → proposto", inclusive os obrigatórios da etapa atual que estiverem vazios.
5. Anote cards da mesma empresa em "Luby" ou "Prospecção" só para citar.

Classifique cada empresa em um destes casos:

| Situação encontrada | O que propor |
|---|---|
| Primeira tentativa de contato feita (e-mail, LinkedIn, ligação), sem reunião marcada | Card no funil de Leads do mercado em **Contatado** (criar, ou mover de Target List). Empresa com Tipo (`type`) = Prospect e contato em Lead no ciclo de vida (`lifecyclestage`); nunca rebaixe quem já está à frente (ex.: Cliente) |
| Reunião marcada | Card em **Reunião Agendada** |
| Lead faltou à reunião | Card volta para **Contatado**, com tarefa de nova tentativa |
| Ainda não é o momento | Card volta para **Target List** ou **Contatado**, com tarefa de retomada (ex.: em 60 dias). Ele entra na nutrição |
| Não seguiu ou não há encaixe | Card para **Perdido**, com motivo de perda (ex.: "Não tem perfil / ICP") |
| Reunião aconteceu e o account quer seguir (desafio concreto, interesse em continuar) | **Conversão** (seção 3): card para Convertido em Deal e **negócio novo em Qualificação** no funil de venda do mesmo mercado. Lead muito quente pode ir direto para Convertido em Deal |
| Negócio aberto existente (BR ou US) | Nota, tarefas, campos e, se os critérios forem cumpridos, sugestão de **uma** etapa à frente (seção 4) |
| Cliente da base (renovação, expansão, reajuste, reposição) | Negócio no funil de venda do mercado, com `natureza_da_venda` = Renovação, Upsell, Cross-sell ou Reposição. Não existe funil Base |

O card de lead é recomendado, mas opcional: se o vendedor não quiser o card, o sinal fica só na empresa e no contato, com a tarefa. **Nunca crie negócio no funil de venda antes da primeira reunião**, nem para "guardar" o contato.

## 3. Funil de Leads (antes do negócio)

| Etapa (prob.) | ID Leads BR | ID Leads US | Quando |
|---|---|---|---|
| Target List (5%) | `1484896440` | `1485590540` | Empresa que queremos acessar, ainda fria |
| Contatado (10%) | `1484896441` | `1485590541` | Já houve a primeira tentativa de contato |
| Reunião Agendada (25%) | `1484896512` | `1485590542` | Reunião marcada com o lead |
| Convertido em Deal (100%) | `1484896513` | `1485590543` | Depois da reunião, o account decidiu que há oportunidade |
| Perdido (0%) | `1484896514` | `1485590544` | Não seguiu |

Regras do card de lead:
- **Desde a Target List:** nome (`Empresa · assunto`, ou só a empresa), dono, `origem_negocio`, `canal_de_origem` (e os condicionais da seção 4, como `evento` para lead de evento), empresa e pelo menos 1 contato associados.
- **Valor sempre vazio** no funil de Leads.
- **Toda movimentação deixa nota ou tarefa** registrada.
- O funil permite voltar etapa (lead que faltou, lead que não está no momento).
- **Perdido:** `motivo_de_perda` obrigatório.
- **Quem decide a conversão é o account, depois da reunião.** A skill só propõe, com a evidência.

**Conversão em negócio (Bloco B, item a item):**
1. O card de lead vai para **Convertido em Deal** e fica no funil de Leads como histórico.
2. Nasce um **negócio novo em Qualificação** no funil de venda do mesmo mercado (BR ou US), com dono, moeda `BRL`, `origem_negocio` e `canal_de_origem` **copiados do card de lead** (eles não passam sozinhos; sem copiar, a origem se perde), os condicionais de canal e, quando houver evidência, `tipo_de_oferta`, `natureza_da_venda`, `bu` e `gatilho_de_compra`.
3. A mesma empresa e o(s) mesmo(s) contato(s) do lead são associados ao negócio novo.
4. A Ficha Luby do negócio cita o ID do card de lead de origem.

## 4. Funil de venda: etapas, critérios e obrigatórios

Etapa é **decisão do cliente**, não atividade da Luby: aceitar reunião, validar escopo, aprovar internamente, assinar. Enviar proposta, sozinho, não é motivo para avançar. Só sugira avançar se o critério de saída tiver **evidência** nas fontes e os obrigatórios da etapa de destino estiverem preenchidos (ou forem preenchidos agora). O negócio nasce sempre em Qualificação. Nunca pule etapa. Voltar é permitido quando o cliente recua (ex.: de Proposta para Solução e Escopo); proponha com evidência e perguntando.

| # | Etapa (prob.) | ID BR | ID US | Entra quando | Sai quando (evidência) | O HubSpot exige para estar na etapa |
|---|---|---|---|---|---|---|
| 1 | Qualificação (10%) | `1484148018` | `1473224705` | Primeira reunião feita, oportunidade real e conversão do lead pelo account | Conta no ICP (tier e porte conhecidos), dor e gatilho identificados, janela de compra até 12 meses | Nome, funil, etapa, dono, `origem_negocio`, `canal_de_origem` (+ condicionais de canal), empresa e pelo menos 1 contato associados |
| 2 | Descoberta (20%) | `1484148019` | `1473224706` | Dor confirmada pelo cliente | Impacto quantificado, sponsor identificado, processo de decisão conhecido, 2+ contatos | Nada a mais. `amount` continua **vazio** |
| 3 | Solução e Escopo (40%) | `1484148020` | `1473224707` | Go/no-go = **Go** (decisão da Luby após a Descoberta) | Escopo validado **por escrito** pelo cliente e proposta pronta | `tipo_de_oferta`, `bu`, `modelo_comercial`, `decisao_go_no_go` |
| 4 | Proposta Apresentada (75%) | `1484148021` | `1473224708` | Proposta **mostrada a quem decide** (não só enviada) | Cliente aprova (e-mail ou ata; `aprovacao_do_comite` = Aprovada) e pede o contrato | `deal_currency_code`, `amount`, `data_provavel_de_fechamento`, `proposta_enviada_arquivo`, `tipo_de_oferta`, `meses_de_contrato`, `modelo_comercial`, `margem_contribuicao`, `receita_mensal`, `sinal_de_risco` |
| 5 | Contrato em Assinatura (90%) | `1484148022` | `1473224709` | Minuta enviada ao cliente | Contrato assinado pelas duas partes | Moeda, `amount`, `data_de_envio_da_minuta`, data provável, meses, receita mensal, margem, `sinal_de_risco` |
| 6 | Ganho (100%) | `1484148023` | `1473224710` | Contrato assinado pelas duas partes e anexado | Passagem ao delivery no kick-off | Moeda, `amount`, `tipo_de_oferta`, `bu`, `proposta_enviada_arquivo`, meses, `origem_negocio`, `natureza_da_venda`, `contrato_anexado` |
| — | Perdido (0%) | `1484148024` | `1473224711` | Cliente desiste, some ou a Luby encerra (No-go) | — | `motivo_de_perda`, `etapa_em_que_perdeu`, `tipo_de_oferta`, `bu` |

Não existe etapa de Comitê: a aprovação interna do cliente acontece dentro de Proposta Apresentada e é registrada em `aprovacao_do_comite`.

**Obrigatórios condicionais** (valem nos funis de venda e no card de lead):
- `canal_de_origem` = Indicação Parceiro → `parceiro_responsavel` e `tipo_de_parceria` obrigatórios.
- `canal_de_origem` = Stand/Patrocínio Evento ou Networking Evento → `evento` obrigatório (padrão mercado e ano, ex.: "US24 - Money2020").
- `canal_de_origem` = Indicação Cliente, Indicação Luber ou Networking Executivo → `quem_indicou` obrigatório (pessoa ou empresa que indicou; parceiro não entra aqui).
- `decisao_go_no_go` = Go, No-go ou Reavaliar → `nota_de_prontidao` e `evidencias_da_nota_de_prontidao` obrigatórias.
- `decisao_do_deal_review` preenchida → `data_do_deal_review` obrigatória.
- `aprovacao_do_comite` = Aprovada → `plano_de_acao_mutuo` aparece (opcional; só o link, não texto longo).
- `motivo_de_perda` = Optou por concorrente → `concorrente` obrigatório.

**O HubSpot também barra, inclusive pelo conector.** Criação ou mudança de etapa sem os obrigatórios é recusada inteira. Por isso a skill confere antes, mostra as lacunas e manda os obrigatórios na mesma gravação da mudança de etapa. Se ninguém souber um obrigatório (ex.: oferta e BU numa perda precoce), registre na Ficha Luby e avise o Growth. **Nunca preencha valor fictício só para passar na trava.** Exceção: `margem_contribuicao` já nasce com `0.0001` (valor padrão do campo), então o HubSpot não barra por ela; a skill confere e trata `0.0001` como vazio.

**Arquivos:** `proposta_enviada_arquivo` (proposta) e `contrato_anexado` (PDF assinado pelas duas partes) são campos de **upload de arquivo**. O conector não sobe arquivo: peça ao vendedor que anexe na tela do negócio antes de propor Proposta Apresentada ou Ganho, e confira que o campo está preenchido.

**No-go:** o HubSpot deixa avançar mesmo com No-go, mas a regra é: com No-go, o negócio não segue para proposta. Proponha voltar para Descoberta ou ir para Perdido (com motivo). "Pendente" não serve para entrar em Solução e Escopo.

**Comitê do cliente:** associe ao negócio cada pessoa do comitê e proponha o **Buying Role** (`hs_buying_role`) de cada contato com base no que ele faz na conversa (um contato pode ter mais de um papel): decisor → `DECISION_MAKER`, dono do orçamento → `BUDGET_HOLDER`, sponsor → `EXECUTIVE_SPONSOR`, champion → `CHAMPION`, técnico/influenciador → `INFLUENCER`, usuário → `END_USER`, jurídico → `LEGAL_AND_COMPLIANCE`, contra → `BLOCKER`, compras e outros → `OTHER`. Referência de comitê mínimo para passar da proposta (em validação; use como **alerta**, não como bloqueio): enterprise 6 pessoas (sponsor, decisor, champion, técnico, compras, jurídico); médio 4; pequeno 2 (decisor e técnico).

**Nota de prontidão (0–100)**, calculada sempre que houver negócio, com a evidência de cada dimensão: ICP e perfil da conta 20 (setor, porte, tier, histórico com a Luby) · Dor e impacto 20 (o problema e quanto custa, ditos pelo cliente) · Decisor e comitê 20 (sponsor, quem assina o orçamento, papéis mapeados) · Orçamento e janela 15 (orçamento aprovado ou em aprovação, data de decisão) · Solução e capacidade 15 (oferta que resolve, time disponível no prazo) · Concorrência e critério 10 (como o cliente decide, alternativas que ele tem). Sem evidência, a dimensão vale 0 e entra como lacuna. Proponha gravar a nota em `nota_de_prontidao` e as evidências em `evidencias_da_nota_de_prontidao`, uma linha por dimensão, com o fato (ex.: "Decisor: CFO participou da 2ª reunião"). Referências em validação (**alerta, não bloqueio**): abaixo de 50, fica em Qualificação ou Descoberta; 70+ pode passar no go/no-go; 80+ pode virar proposta.

### Governança: a skill registra, não decide

- **Go/no-go** (`decisao_go_no_go`: Pendente, Go, No-go, Reavaliar): decisão interna da Luby após a Descoberta. Só registre decisão que já aconteceu e que o vendedor confirmou; a aprovação fica registrada na nota.
- **Comitê do cliente** (`aprovacao_do_comite`: Não iniciada, Em aprovação, Aprovada, Reprovada, Não se aplica): Aprovada só com a ata ou o e-mail do cliente, guardado como atividade.
- **Deal review** (`data_do_deal_review` + `decisao_do_deal_review`: Aprovada, Aprovada com ajustes, Reavaliar, Encerrar oportunidade): revisão interna com Jan, CTO, financeiro e o account. Só registre revisão que aconteceu. O critério de quando fazer (acima de R$ 500 mil ou cliente enterprise) ainda está em validação: use como alerta.
- **Sinal de risco** (`sinal_de_risco`), obrigatório a partir da Proposta. Proponha atualizar a cada conversa importante, com o motivo na nota:
  - **Sem risco:** cliente engajado e próximo passo com data marcada.
  - **Risco baixo:** um ponto em aberto, com plano para resolver.
  - **Risco médio:** prazo escorregou, decisor ausente ou objeção sem resposta.
  - **Risco alto:** sem resposta há mais de 15 dias, concorrente na frente ou orçamento em dúvida.
  Não converta o legado `temperatura_do_negocios`.
- `gatilho_de_compra` (um fato concreto, com as palavras do cliente: RFP aberta, prazo, regra nova, expansão, orçamento aprovado; não confunda com o canal), `criterios_de_decisao` (o que o cliente disse que pesa, e de onde veio a informação) e `plano_de_acao_mutuo` (link): preencha só com evidência.

## 5. O que extrair (ficha do negócio)

Preencha só com evidência, usando os nomes internos abaixo.

- **Nome do negócio** (`dealname`): nome da empresa primeiro, depois o assunto. Ex.: `Acme Bank · Modernização do core`.
- **Empresa associada** (sempre): domínio, setor (`industry`), porte (`numberofemployees`), `tier` (Tier 1, Tier 2, Tier 3; as faixas por funcionários ainda estão em definição) e `nota_icp` (0–100; a rubrica ainda está em validação: proponha com a evidência e diga isso). Não confunda com `hs_ideal_customer_profile`. O `canal_de_origem` da **empresa** é o canal do primeiro contato com a conta e não substitui o canal do negócio.
- **Contatos** com cargo (`jobtitle`), e-mail e papel na compra (`hs_buying_role`).
- **Funil e moeda:** funil do mercado; moeda `BRL` (`deal_currency_code`).
- **Valor** (`amount`): o contrato inteiro, somando todos os meses (ex.: R$ 50 mil por mês em 12 meses = R$ 600 mil). **Vazio até a proposta**; obrigatório em Proposta Apresentada. Estimativa feita antes da proposta vai só na Ficha Luby. **Nunca R$ 0 ou R$ 1 como marcador.**
- **Datas:** previsão de assinatura em `data_provavel_de_fechamento` (só aceita **dia útil**; se cair no fim de semana, proponha a segunda-feira seguinte e diga). Atualize sempre que a previsão mudar. `closedate` é só o fechamento real: em Ganho, a data da assinatura; em Perdido, a data da perda. **Nunca grave `closedate` em negócio aberto.**
- **Origem do negócio** (`origem_negocio`): Inbound (o cliente veio até a Luby) ou Outbound (a Luby foi até o cliente). Origem e canal precisam combinar: prospecção ativa costuma ser Outbound; site, anúncio e busca costumam ser Inbound.
- **Natureza da venda** (`natureza_da_venda`): Nova aquisição (cliente novo), Upsell (mais do mesmo para quem já é cliente), Cross-sell (outra oferta para quem já é cliente), Renovação, Reposição (repor alguém ou algo num contrato existente).
- **Canal de origem** (`canal_de_origem`): o caminho mais específico da lista, **nunca "Outro"**. Se não souber, pergunte. Atenção aos condicionais da seção 4.
- **Parceiro** (`parceiro_responsavel` + `tipo_de_parceria` no negócio) ou **quem indicou** (`quem_indicou`).
- **Oferta** (`tipo_de_oferta`), **BU** (`bu`, a opção do mercado certo: no funil US, as opções US) e **modelo comercial** (`modelo_comercial`: Alocação, Escopo fechado, Managed squad, Formato misto).
- **Contrato:** `meses_de_contrato` (texto que aceita só número inteiro, ex.: `12`), `receita_mensal` (o valor de um mês de contrato) e `margem_contribuicao`. A margem é um campo de **percentual com até 2 casas decimais**: na tela o account digita `38,5` e vê 38,5%; pelo conector grava-se a **fração** (`0.385`, até 4 casas). Mostre ao vendedor sempre em percentual. O HubSpot preenche `0.0001` como valor padrão do campo em negócio novo: trate `0.0001` como margem vazia e nunca grave esse valor.
- **Documentos:** `proposta_enviada_arquivo` e `contrato_anexado` (upload pelo vendedor; veja a seção 4) e `data_de_envio_da_minuta`.
- **MEDDICC resumido:** dor e impacto em reais, gatilho de compra, decisor econômico, sponsor, champion, critérios e processo de decisão, orçamento, janela, concorrentes, objeções.
- **Próximos passos** com dono, data e o que o cliente se comprometeu a fazer.
- **Perda:** `motivo_de_perda` da lista, `etapa_em_que_perdeu` (a última etapa em que o negócio estava), `concorrente` (obrigatório se o motivo for "Optou por concorrente").

**Não preencha:**
- Campos de pós-venda: faturamento (`valor_faturado_ano_atual`, `valor_faturado_ano_post`, início e fim do faturamento), `health_score_conta` e churn (`cliente_churnado_` e seus dependentes, que são só para cliente da base que saiu). São do delivery e do CS.
- Campos legados: `origem_do_negocio`, `new_client`, `proposta_enviada`, `margem_de_contribuicao`, `temperatura_do_negocios`, `account_executive`, `closed_lost_reason`.

## 6. Revisão humana (obrigatória)

Antes de gravar qualquer coisa, mostre por empresa:

**Cabeçalho:** Empresa · card de lead (funil e etapa), se houver · negócio (funil e etapa), se houver · nota de prontidão com 1 linha de justificativa · obrigatórios faltando na etapa atual · cards da empresa em "Luby" ou "Prospecção", se houver · fontes usadas (assunto e data do e-mail, nome da conversa).

**Bloco A — registro de atividade** (baixo risco, aprovado em lote):
- Nota da reunião ou da troca de e-mails (modelo abaixo), com a Ficha Luby.
- Tarefas de follow-up com dono e data.
- Contatos e empresa novos, associações, cargo e buying role.

**Bloco B — decisões** (uma linha por item, aprovação item a item):

| # | Objeto | Campo | Atual | Proposto | Evidência | Confiança |
|---|---|---|---|---|---|---|

Sempre no Bloco B: criar ou mover card de lead, conversão do lead em negócio, criar negócio, funil, moeda, mudar etapa, valor, data provável, data de fechamento, oferta, BU, modelo comercial, origem, natureza, canal, nota de prontidão, go/no-go, comitê, deal review, sinal de risco, ganho e perda.

**Lacunas:** obrigatórios da etapa atual e da próxima que continuam vazios, com a pergunta que o vendedor deve fazer ao cliente. Inclua os arquivos que o vendedor precisa anexar na tela.

Feche com: "Aprova o Bloco A? E quais itens do Bloco B (ex.: 1, 3, 4)? Pode corrigir qualquer valor." Aceite edições ("o valor é 380 mil", "a data é 30/11") e mostre a versão final antes de gravar se algo mudou.

No primeiro uso, deixe a pessoa dispensar só a reconfirmação do Bloco A na conversa. O Bloco B sempre pede OK.

## 7. Gravação no HubSpot

Só depois do OK, com `manage_crm_objects` (confirmationStatus CONFIRMED), no máximo 10 objetos por chamada:
1. Empresa e contatos (criar ou atualizar) e associações.
2. Card de lead: criar (com nome, dono, origem, canal, condicionais, valor vazio e associado à empresa e ao contato na mesma chamada) ou mover de etapa.
3. Negócio: criar em Qualificação (na conversão ou para cliente da base) já com nome, funil, dono, moeda, `origem_negocio`, `canal_de_origem` (e os condicionais) e associado à empresa e ao(s) contato(s) na mesma chamada; ou atualizar os campos aprovados.
4. Nota (objeto NOTE, `hs_note_body` e `hs_timestamp`) associada ao card ou negócio, à empresa e aos contatos.
5. Tarefas (TASK: `hs_task_subject`, `hs_task_body`, `hs_timestamp` = vencimento, `hubspot_owner_id`) associadas ao card ou negócio.
6. Etapa por último, se aprovada, **na mesma gravação** dos obrigatórios da etapa de destino que ainda faltarem (em Ganho e Perdido, junto com o `closedate` aprovado). Na conversão, mova o card de lead para Convertido em Deal só depois de o negócio novo ter sido criado.

Se o HubSpot recusar, diga qual campo ele pediu, o que foi gravado e o que não foi, registre na Ficha Luby e pergunte ao vendedor. Nunca repita uma gravação que pode já ter acontecido sem conferir antes.

O conector só **cria** associações. Associação errada precisa ser removida pelo vendedor na tela do negócio: avise.

### Modelo da nota

A nota tem duas partes: o relato da conversa e a **Ficha Luby**. O campo do HubSpot mostra só o valor de hoje; a Ficha guarda a foto da avaliação no dia de cada registro, o que não tem campo ou opção no HubSpot e o que o HubSpot recusou, em linhas fixas que dá para comparar depois. Linha sem conteúdo sai da nota.

```
[Registro assistido por IA — revisado por <vendedor> em <data>]
Fonte: <e-mail "assunto" de dd/mm | reunião dd/mm | conversa no Claude>

Resumo: <3 a 5 linhas do que o cliente disse e decidiu>
Dor e impacto: <...>
Gatilho de compra: <...>
Decisor / sponsor / champion: <nomes e cargos>
Orçamento e janela: <...>
Concorrentes e critérios: <...>
Objeções: <...>
Próximos passos: <ação — dono — data>

Ficha Luby (foto de dd/mm/aaaa)
Funil e etapa: <Leads BR · Reunião Agendada | BR · Descoberta>
Lead de origem: <ID do card no funil de Leads · origem · canal>
Nota de prontidão: <nn>/100 — ICP x · dor x · comitê x · orçamento x · solução x · concorrência x
Comitê mapeado: <n> de <referência por porte> — faltam: <papéis>
Sinal de risco: <nível> — <motivo>
Valor estimado antes da proposta: <...>
Sem campo ou opção no HubSpot: <campo — dado — avisar o Growth>
Recusado pelo HubSpot: <campo — motivo>
Lacunas para a próxima conversa: <...>
```

## 8. Resumo final para o vendedor

Curto: o que foi gravado (por empresa), o que ficou pendente de decisão, as lacunas para a próxima conversa com o cliente, os arquivos a anexar e os alertas da reunião de segunda que se aplicam a ele:
- Negócio ou card de lead sem atividade há mais de 21 dias.
- Card em Reunião Agendada com a reunião já passada, sem conversão, retorno de etapa ou perda.
- Card de lead com valor preenchido.
- Negócio convertido sem origem ou canal.
- Data provável de fechamento vencida, ou sinal de risco desatualizado num negócio que atrasou.
- Obrigatórios da etapa atual vazios.
- Negócio em Proposta Apresentada ou Contrato em Assinatura sem valor ou com a margem ainda no padrão 0,0001 (na prática, vazia); valor 0 ou 1 em qualquer etapa (marcador falso).
- Negócio com moeda diferente de BRL ou sem moeda.
- Negócio sem empresa ou sem contato associado.
- Dono desativado em negócio aberto.
- Negócio aberto no funil antigo "Luby".
- Go/no-go = No-go com o negócio ainda avançando.
- Referências em validação (alerta, não bloqueio): proposta com nota de prontidão abaixo de 80; comitê abaixo do mínimo por porte; negócio acima de R$ 500 mil (ou enterprise) sem deal review.

## Regras que nunca mudam

- **Nada vai para o HubSpot sem aprovação humana.** Conversão, ganho, perda, etapa e valor sempre item a item.
- **Só os funis Leads BR, Leads US, BR e US.** Nunca crie nem mova nada em "Luby" ou "Prospecção".
- **Lead não é negócio.** Antes da primeira reunião, o lead vive no funil de Leads (ou só na empresa e no contato). O negócio nasce em Qualificação quando o account converte o lead, depois da reunião, com origem e canal copiados do lead.
- **Ganho só com contrato assinado** pelas duas partes e anexado (`contrato_anexado`). Aprovação verbal, e-mail de "fechado" ou início do projeto não contam.
- **Uma etapa por vez**, com critério cumprido, evidência citada e obrigatórios preenchidos.
- **Valor vazio no funil de Leads e até a proposta; moeda BRL; previsão em data provável; `closedate` só no fechamento.**
- **Sem palpite:** dado sem evidência fica vazio e vira pergunta. Nada de R$ 0 ou R$ 1, margem 0,0001 tratada como real, opção qualquer da lista, data inventada ou campo preenchido só para passar na trava do HubSpot.
- **Fato do cliente ≠ sugestão do Claude.** Só entra o que o cliente ou o vendedor disseram.
- **Decisões são de pessoas:** conversão do lead, go/no-go, deal review e aprovação do comitê são registrados, nunca decididos pela skill.
- **LGPD e discrição:** registre só informação comercial e resuma (não cole e-mails inteiros). Nada de dado pessoal sensível de contatos. Reunião gravada exige aviso ao cliente.
- Respeite o que o portal tem: se um funil, etapa, campo ou opção não existir, não crie configuração. Isso é com o Growth (admin do HubSpot); registre na Ficha Luby e avise o vendedor.
