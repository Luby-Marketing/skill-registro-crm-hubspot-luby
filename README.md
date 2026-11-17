# skill-registro-crm-hubspot-luby

Marketplace da Luby com o plugin **luby-hubspot-toolkit**, que traz a skill **registro-crm-hubspot-luby**: o Claude lê os e-mails e as conversas do vendedor e propõe o registro no HubSpot, com leads nos funis **Leads BR** e **Leads US** e negócios nos funis de venda **BR** e **US**. A IA preenche, confere e alerta; o vendedor aprova. Nada é gravado sem o OK dele.

As regras seguem a GMUD do HubSpot (nov/2026), o Funil de Leads (11/11/2026) e o Guia do Account (13/11/2026).

## Requisitos

- **Claude** (app desktop ou claude.ai). É onde a skill tem todas as fontes. Ela também roda no Claude Code, mas lá não existe a busca nas conversas do Claude.
- **Conector do HubSpot**, com permissão de escrita em Negócios, Empresas, Contatos, Notas e Tarefas.
- **Conector do Gmail.** Opcionais: Google Calendar, Plaud ou Fireflies.

## Instalar

### Time comercial (Claude app e claude.ai)

A skill chega pelo catálogo de plugins da organização, sincronizado com este repositório. Quem usa não precisa de acesso ao GitHub e recebe cada versão nova sozinho.

1. Em **Customize > Plugins**, procure **luby-hubspot-toolkit** e instale. Se o admin deixou o plugin instalado por padrão, ele já aparece ativo.
2. Peça em linguagem natural (veja "Como chamar").

**Configuração do admin, uma vez:** em **Organization settings > Plugins & skills > Add > Sync from GitHub**, escolha o repositório `Luby-Marketing/skill-registro-crm-hubspot-luby`, deixe **Sync automatically** ligado e defina a disponibilidade (disponível, instalado por padrão ou obrigatório). Se o repositório não aparecer na lista, instale o app do Claude no GitHub para ele. Com a sincronização automática, cada push na `main` atualiza o plugin para toda a organização.

### Claude Code

```
/plugin marketplace add Luby-Marketing/skill-registro-crm-hubspot-luby
```

```
/plugin install luby-hubspot-toolkit@luby-hubspot
```

O repositório é privado: quem instala precisa ter acesso a ele no GitHub e credencial do git na máquina (`gh auth login` e depois `gh auth setup-git`).

Para receber as versões novas sozinho, ligue a atualização automática uma vez: em `/plugin`, aba **Marketplaces**, selecione `luby-hubspot` e escolha **Enable auto-update**. A atualização é baixada alguns minutos depois do início da sessão e passa a valer na sessão seguinte (ou depois de `/reload-plugins`). Para atualizar na hora:

```
/plugin marketplace update luby-hubspot
```

### Zip, só se não houver outra forma

Para quem está fora da organização, dá para enviar um `.zip` com a pasta `registro-crm-hubspot-luby` (e o `SKILL.md` dentro) em **Customize > Skills**. Esse caminho **não se atualiza**: cada versão nova exige um novo envio.

## Como chamar

Peça em linguagem natural; a skill é acionada sozinha:

| Você pede | O que sai |
|---|---|
| "atualiza meu CRM", "FUP do CRM da semana" | Varredura dos últimos 7 dias e a proposta por empresa, em Bloco A (lote) e Bloco B (item a item) |
| "registra a reunião de hoje com a Acme no HubSpot" | Registro daquela conversa e, se o account quiser seguir, a conversão do lead em negócio |
| "o que eu preciso lançar no CRM?" | Pendências e alertas da semana; só grava o que for aprovado |

No Claude Code também dá para chamar direto com `/luby-hubspot-toolkit:registro-crm-hubspot-luby`.

## Publicar uma versão nova

Toda mudança que precisa chegar ao time segue estes passos:

1. Edite `plugins/luby-hubspot-toolkit/skills/registro-crm-hubspot-luby/SKILL.md`.
2. Suba o `version` em `plugins/luby-hubspot-toolkit/.claude-plugin/plugin.json`. **Sem subir a versão, quem usa o Claude Code continua com a cópia antiga.**
   - Ajuste de texto ou correção: último número (3.0.0 → 3.0.1).
   - Regra nova ou mudança no que a skill grava: número do meio (3.0.0 → 3.1.0).
   - Mudança de funil ou de estrutura do plugin: primeiro número (3.0.0 → 4.0.0).
3. Registre a mudança no [`CHANGELOG.md`](CHANGELOG.md).
4. Faça o commit na `main`, crie a tag `vX.Y.Z` e envie: `git push` e `git push --tags`.

O push na `main` dispara a sincronização do catálogo da organização. A documentação não diz se essa sincronização também depende do número de versão, por isso a regra é subir a versão sempre.

Antes de enviar, dá para conferir a estrutura com `claude plugin validate .`.

Mudanças de funil, etapa ou campo no HubSpot são feitas pelo Growth (admin do HubSpot). Quando acontecerem, a skill precisa de uma versão nova.

## Estrutura

```
.claude-plugin/marketplace.json            marketplace luby-hubspot
CHANGELOG.md                               histórico de versões
plugins/luby-hubspot-toolkit/
  .claude-plugin/plugin.json               nome e versão do plugin
  skills/registro-crm-hubspot-luby/
    SKILL.md                               instruções da skill
```

As tags `v1` e `v2` são das versões anteriores ao plugin, quando a skill ficava em `registro-crm-hubspot-luby/SKILL.md`. A partir da 3.0.0, as tags seguem a versão do plugin (`v3.0.0`, `v3.1.0`…).
