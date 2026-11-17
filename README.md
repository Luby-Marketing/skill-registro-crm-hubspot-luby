# registro-crm-hubspot-luby

Skill do Claude que varre os e-mails e as conversas do vendedor e propõe o registro no HubSpot nos funis de venda **BR** e **US** da Luby. A IA preenche, confere e alerta; a pessoa aprova. Nada é gravado sem o OK do vendedor.

O conteúdo da skill está em [`registro-crm-hubspot-luby/SKILL.md`](registro-crm-hubspot-luby/SKILL.md).

## Como instalar

- **Claude (app ou claude.ai):** compacte a pasta `registro-crm-hubspot-luby` em um `.zip` e envie na área de Skills das configurações do Claude.
- **Claude Code:** copie a pasta `registro-crm-hubspot-luby` para `~/.claude/skills/`.

## O que precisa estar conectado

- **HubSpot**, com permissão de escrita em Negócios, Empresas, Contatos, Notas e Tarefas.
- **Gmail** e a busca nas conversas do Claude, para a coleta (disponíveis no claude.ai).
- Opcionais: Google Calendar, Plaud ou Fireflies.

## Versões

| Versão | Data | O que mudou |
|---|---|---|
| v2 | 17/11/2026 | Ajustada à GMUD HubSpot (Ondas 1 e 2) e à migração de 10/11/2026: só os funis BR e US; 6 etapas + Perdido (10/20/40/75/90/100%); obrigatórios por etapa, que o HubSpot também barra pelo conector; valor só a partir da proposta e previsão em data provável; campos de governança da Onda 2; lista de campos legados. |
| v1 | início de nov/2026 | Primeira versão, a partir do Funil do Cliente (Matheus Pedralli, out/2026), apresentada ao comercial antes da migração. |

Mudanças de funil, etapa ou campo no HubSpot são feitas pela admin do HubSpot. Quando acontecerem, atualize o `SKILL.md`, registre a nova versão nesta tabela e crie a tag correspondente.
