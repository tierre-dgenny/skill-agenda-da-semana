---
name: agenda-da-semana
description: Lista todas as reuniões com clientes programadas para a semana atual cujo título contenha "IA para compras" ou "plano de ROI", mostrando empresa, contato, telefone, data e horário. TRIGGER quando usuário pedir "agenda da semana", "reuniões da semana", "minha agenda", "que reuniões tenho essa semana", "compromissos da semana", "lista de reuniões", "reuniões programadas" ou qualquer variação de consulta de agenda semanal com clientes.
---

# Skill: Agenda da Semana dgenny®

## O que esta skill faz

Consulta o Google Calendar via Composio e retorna todas as reuniões da **semana atual** (segunda a domingo) cujo título contenha:

- `IA para compras` (com qualquer capitalização)
- `plano de ROI` (com qualquer capitalização)

Para cada reunião, extrai e exibe:

| Campo | Fonte |
|---|---|
| **Empresa** | Dados do evento (descrição, localização) e/ou domínio do e-mail dos convidados |
| **Contato** | Nome dos participantes/convidados do evento |
| **Telefone** | Descrição do evento (se presente) |
| **Data** | Dia em número + dia da semana em português |
| **Horário** | Início e fim do evento |

---

## Fluxo de execução

### Passo 1 — Calcular o intervalo da semana atual

Determinar o intervalo da semana atual em formato ISO 8601 (UTC-3, horário de Brasília):

- **Início:** segunda-feira da semana atual às 00:00:00
- **Fim:** domingo da semana atual às 23:59:59

Usar a data de hoje disponível no contexto (`currentDate`) para calcular.

**Exemplo:** se hoje é 2026-06-22 (segunda), o intervalo é `2026-06-22T00:00:00-03:00` a `2026-06-28T23:59:59-03:00`.

---

### Passo 2 — Buscar a ferramenta correta no Composio

Usar `mcp__composio__COMPOSIO_SEARCH_TOOLS` para localizar ferramentas de listagem de eventos do Google Calendar:

```
query: "google calendar list events"
```

Identificar a ferramenta correta (tipicamente `GOOGLECALENDAR_LIST_EVENTS` ou similar).

Se a ferramenta não aparecer na primeira busca, tentar também:
```
query: "google calendar find event"
```

---

### Passo 3 — Executar a consulta ao Google Calendar

Usar `mcp__composio__COMPOSIO_MULTI_EXECUTE_TOOL` com a ferramenta identificada no Passo 2, passando:

- `calendarId`: `primary`
- `timeMin`: início da semana (formato RFC3339)
- `timeMax`: fim da semana (formato RFC3339)
- `singleEvents`: `true`
- `orderBy`: `startTime`
- `maxResults`: `100`

---

### Passo 4 — Filtrar os eventos relevantes

Da lista retornada, manter **apenas** os eventos cujo título (`summary`) contenha (case-insensitive):
- `IA para compras`
- `plano de ROI`

Se nenhum evento for encontrado com esses critérios, informar:
```
Nenhuma reunião com "IA par compras" ou "plano de ROI" encontrada para esta semana.
```

---

### Passo 5 — Extrair os dados de cada evento

Para cada evento filtrado, extrair:

#### Empresa
Buscar em ordem de prioridade:
1. Campo `description` do evento — procurar menção explícita à empresa (ex: "Cliente: Construtora X", "Empresa: Y")
2. Campo `location` do evento
3. **Domínio do e-mail** dos convidados (`attendees[].email`) — excluir domínios genéricos (gmail.com, hotmail.com, outlook.com, yahoo.com, icloud.com, me.com) e domínios internos dgenny — o domínio restante geralmente indica a empresa. Formatar como nome próprio (ex: `construtoranordeste.com.br` → `Construtora Nordeste`)
4. Se nenhuma fonte identificar a empresa, registrar `—`

#### Contato
1. Lista de `attendees` — pegar os **nomes** (`attendees[].displayName`) que não sejam da dgenny (excluir e-mails com `@dgenny.com.br` ou o próprio calendário)
2. Se `displayName` não estiver disponível, usar a parte do e-mail antes do `@` formatada como nome (ex: `joao.silva@` → `João Silva`)
3. Se houver múltiplos convidados externos, listar todos separados por vírgula

#### Telefone
1. Procurar no campo `description` padrões de telefone brasileiro:
   - `(XX) XXXXX-XXXX` ou `(XX) XXXX-XXXX`
   - `+55 XX XXXXX-XXXX`
   - Variações sem formatação: 10 ou 11 dígitos seguidos
2. Se não encontrado: `—`

#### Data
- Extrair `start.dateTime` ou `start.date`
- Formatar como: `DD · [Dia da semana]`
- Dias da semana em português: Segunda, Terça, Quarta, Quinta, Sexta, Sábado, Domingo
- Exemplo: `22 · Segunda`

#### Horário
- Formatar como: `HH:MM – HH:MM`
- Exemplo: `10:00 – 11:00`
- Usar horário de Brasília (UTC-3)

---

### Passo 6 — Montar e exibir a lista

Apresentar em formato de tabela markdown limpa:

```
📅 AGENDA DA SEMANA — [DD/MM] a [DD/MM/AAAA]
Reuniões com "IA par compras" e "plano de ROI"

| # | Empresa | Contato | Telefone | Data | Horário |
|---|---------|---------|----------|------|---------|
| 1 | [Empresa] | [Contato] | [Fone] | [Data] | [Horário] |
| 2 | ... | ... | ... | ... | ... |

Total: [N] reunião(ões) encontrada(s)
```

---

## Regras importantes

- **Nunca inventar dados** — se não encontrou empresa, contato ou telefone, colocar `—`
- **Filtro é obrigatório** — exibir SOMENTE eventos com "IA par compras" ou "plano de ROI" no título
- **Semana atual sempre** — não consultar semanas passadas ou futuras, salvo pedido explícito
- **Horário de Brasília** — todos os horários devem ser convertidos para UTC-3
- **Excluir participantes internos** — convidados com e-mail dgenny não são "contatos do cliente"
- **Domínio como empresa** — quando não há nome explícito da empresa no evento, o domínio do e-mail do cliente é a melhor pista; formatar de forma legível
- **Se o Composio não tiver conexão com Google Calendar**, informar à usuária e orientar a conectar em composio.io antes de usar a skill novamente
- **Nunca usar "Jane"** — o agente se chama dgenny®
