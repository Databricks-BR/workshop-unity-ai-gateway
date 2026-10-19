# 🎬 Netbricks — Lab de Streaming no Databricks

Lab fim-a-fim de uma plataforma de streaming fictícia (**Netbricks**) sobre a Lakehouse.
Cobre **geração de dados em escala → engenharia de atributos → dois modelos de ML (churn e
propensão de upgrade) → Vector Search → um agente híbrido (Mosaic AI Agent Framework)**, com
a camada de GenAI governada por **Unity AI Gateway**.

> Nenhum dado é real: usuários, títulos, e-mails (`@netbricks.com`) e artigos de ajuda são
> todos sintéticos e gerados de forma determinística. O lab roda em **qualquer workspace
> Databricks** com os pré-requisitos abaixo — basta ajustar `catalog`/`schema` no `_config`.

---

## Apresentadores

<table>
  <tr>
    <td align="center" width="50%">
      <img src="https://raw.githubusercontent.com/Databricks-BR/workshop-energy/main/images/juliandro_circle.png" width="150"/><br>
      <strong>Juliandro Figueiró</strong><br>
      <em>Sr. Solutions Architect</em><br>
      <em>Databricks</em>
    </td>
    <td align="center" width="50%">
      <img src="https://raw.githubusercontent.com/Databricks-BR/workshop-energy/main/images/jean_circle.png" width="150"/><br>
      <strong>Jean Ertzogue</strong><br>
      <em>Account Executive</em><br>
      <em>Databricks</em>
    </td>
  </tr>
</table>

---

## Ambiente

| Item | Valor |
|------|-------|
| Compute | Serverless notebooks / jobs |
| Catálogo | `CATALOG` no `_config` (padrão `netbricks`) — **deve existir** |
| Schema | `SCHEMA` no `_config` (padrão `suas_iniciais_aqui`) — **criado na preparação** (`preparacao/00_gerar_dados`) |
| Vector Search endpoint | `netbricks_vs_<schema>` (criado em `preparacao/04_vector_search`; nome inclui o schema para evitar colisão entre labs no mesmo workspace) |

**Configuração centralizada:** catálogo e schema ficam em **um único lugar** — o notebook
**`_config`**. Todos os notebooks (0–7) fazem `%run ./_config` no início e reaproveitam
`CATALOG`, `SCHEMA` e `VS_ENDPOINT`. Você edita o `_config` **uma vez** e não precisa repetir em
cada notebook. Em um hands-on com várias pessoas no mesmo workspace, o catálogo `netbricks`
é compartilhado e **cada participante troca o `SCHEMA` pelas suas próprias iniciais** — assim
tabelas, modelos e o endpoint de Vector Search (`netbricks_vs_<schema>`) ficam isolados por
pessoa. O schema é criado automaticamente; o catálogo **não** é criado (deve existir e você
precisa ter permissão de uso).

> ⚠️ O `_config` precisa estar **na mesma pasta** dos notebooks (o `%run ./_config` é relativo).

### Modelos usados

| Papel | Modelo | Onde entra |
|-------|--------|------------|
| LLM do agente | `system.ai.claude-opus-4-8` (model service do Unity Gateway, nome UC de 3 partes) — **parametrizável** | raciocínio e *tool calling* do agente híbrido |
| Embeddings | `databricks-gte-large-en` (Foundation Model API) | geração de embeddings dos índices de Vector Search |
| ML clássico | scikit-learn `HistGradientBoostingClassifier` | modelos de churn e de propensão de upgrade |

> O LLM é consumido como **model service do Unity Gateway** (chamado pela API unificada
> `/ai-gateway/mlflow/v1` pelo nome UC de 3 partes), então toda chamada já passa pela governança do
> gateway; o endpoint de embeddings é um endpoint de serving. Ver a seção do Unity AI Gateway abaixo.

> **O LLM é um parâmetro (widget `llm`) no notebook 02.** O default é
> `system.ai.claude-opus-4-8` — referenciado pelo **nome de 3 partes no Unity Catalog** (model
> service do Unity Gateway) e chamado pela **API unificada** do gateway (`/ai-gateway/mlflow/v1`),
> com *tool calling*.
> Como o LLM é chamado **via OBO (on-behalf-of-user)**, quem precisa de `EXECUTE` no model service
> é o **usuário** que invoca o agente — não o service principal do endpoint. Você pode trocar por
> qualquer model service de chat com *tool calling* a que o usuário tenha acesso (ex.:
> `system.ai.claude-sonnet-4-5` ou um `catalog.schema.nome` customizado). O agente usa o valor via a
> variável de ambiente `LLM_MODEL`.

---

## Pré-requisitos

- Workspace Databricks com **Serverless compute** habilitado.
- **Unity Catalog** com um catálogo onde você possa criar schema/tabelas/modelos.
- **Vector Search** habilitado no workspace.
- Acesso às **Foundation Model APIs**: o **usuário** do lab precisa de `EXECUTE` no model service de
  LLM escolhido (widget `llm`; default `system.ai.claude-opus-4-8`), pois ele é chamado via OBO; o
  endpoint de embeddings (`databricks-gte-large-en`) é usado com credenciais de sistema.
- **Preview "Agent Framework: On-Behalf-Of-User Authorization" habilitado** pelo admin do workspace
  (em *(seu usuário) → Previews*) — necessário para o agente chamar LLM e Genie como o usuário.
- Um **Genie Agent** sobre as tabelas do lab (ver [passo abaixo](#genie-agent-para-a-ferramenta-de-dados)) — o agente o consulta via API REST do Genie (SDK), como o usuário (OBO).

---

## 🚀 Passo a passo do lab (ordem de execução)

Do zero ao agente rodando dentro de um App — **6 passos**:

| # | Passo | Como fazer |
|:-:|-------|------------|
| **1** | ⚙️ **Preencher o `_config`** | Defina `CATALOG` e `SCHEMA` **uma única vez** no notebook `_config`. Todos os demais notebooks o reaproveitam via `%run`. |
| **2** | 🏗️ **Criar todo o ambiente** | Rode o `00_preparar_ambiente.py` — o **orquestrador** executa, em ordem, os 5 notebooks de `preparacao/` (dados → features → ML → Vector Search). |
| **3** | 🧞 **Criar o Genie Agent** | Rode o `01_criar_genie_agent.py` (prepara PK/FK e consolida as especificações) e, em seguida, peça ao **Genie code**:<br>👉 *"Crie um genie agent baseado nas especificações deste notebook"* |
| **4** | 🤖 **Model service do LLM + deploy do agente** | **(a)** Crie um **model service do Unity AI Gateway** sobre `system.ai.claude-opus-4-8` (ou mais recente), com nome UC de 3 partes `catalog.schema.modelservice_name`, e **aplique nele uma policy de mascaramento de e-mail (PII)**. **(b)** No `02_deploy_agent.py` preencha **os dois widgets**: `genie_space_id` (ID do passo 3) **e** `llm` com o nome do model service (`catalog.schema.modelservice_name`). Rode o notebook → o agente é logado, registrado no UC e publicado em Model Serving (`netbricks_agent_<schema>`). Ver [Detalhe do passo 4](#detalhe-do-passo-4--model-service-do-llm-e-policy-de-masking-de-e-mail). |
| **5** | 💬 **Criar o App** | Crie um **Databricks App** apontando para o **agente já publicado** (o endpoint de serving do passo 4). |
| **6** | 🔑 **Conceder os escopos ao App** | Rode o `03_conceder_escopos_app.py` (widget `APP_NAME`, padrão `chat-netbricks`) — ele faz um `PATCH /api/2.0/apps/{APP_NAME}` aplicando os `user_api_scopes` necessários (`model-serving`, `genie`, `sql`, `ai-gateway`, `vector-search` e `catalog.*:read`). |

### Detalhe do passo 2 — fase de preparação (`preparacao/`)

Rodada pelo `00_preparar_ambiente.py` (ou individualmente, **nesta ordem**):

| # | Notebook | O que faz | Saídas principais |
|---|----------|-----------|-------------------|
| **0** | `preparacao/00_gerar_dados.py` | Gera toda a base de forma determinística e com **sinal comportamental** (engajamento latente governa plano, churn e volume de eventos). Cria o schema. | `catalogo` (10k títulos, PK + CDF), `usuarios` (120k), `eventos_visualizacao` (1M, particionado), `central_ajuda_kb` (12 artigos, PK + CDF) |
| **1** | `preparacao/01_feature_engineering.py` | Agrega eventos por usuário e junta com o perfil. | `features_usuarios` |
| **2** | `preparacao/02_modelo_churn.py` | Treina classificador de **churn** (`status = 'Cancelado'`), registra no UC e escora toda a base. | modelo `modelo_churn`, tabela `scores_churn` (prob + faixa de risco) |
| **3** | `preparacao/03_modelo_upgrade.py` | Treina **propensão de upgrade** (`is_pago`) e aplica aos usuários **Free** — os de maior probabilidade são os melhores alvos de conversão. | modelo `modelo_upgrade`, tabela `scores_upgrade` |
| **4** | `preparacao/04_vector_search.py` | Cria o endpoint VS e dois índices Delta Sync. | `catalogo_index` (sinopse), `ajuda_index` (conteúdo) |

> **Parâmetros:** ajuste `CATALOG`/`SCHEMA` **uma vez** no `_config` (os notebooks de `preparacao/`
> usam `%run ../_config`). A ordem **0 → 1 → 2 → 3** da preparação é obrigatória — cada etapa consome
> a saída da anterior.

### Detalhe do passo 4 — model service do LLM e policy de masking de e-mail

Antes de dar deploy no agente, publique o LLM como um **model service do Unity AI Gateway** com um
nome UC de 3 partes e ligue o **mascaramento de PII (e-mail)**. Assim toda inferência do agente passa
pela governança do gateway — e os e-mails sintéticos (`@netbricks.com`) são mascarados em prompts e
respostas.

**(a) Criar o model service sobre Opus 4.8 (ou mais recente)**

Pela UI: **Serving → Create serving endpoint** → modelo base `system.ai.claude-opus-4-8` (ou versão
mais recente) → dê ao endpoint um nome UC de 3 partes `catalog.schema.modelservice_name`.

Ou via SDK (representativo; os nomes de classe podem variar conforme a versão do `databricks-sdk` —
a UI é a fonte de verdade):

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import (
    EndpointCoreConfigInput, ServedEntityInput,
    AiGatewayConfig, AiGatewayGuardrails, AiGatewayGuardrailParameters,
    AiGatewayGuardrailPiiBehavior, AiGatewayGuardrailPiiBehaviorBehavior,
)

w = WorkspaceClient()
NOME = "catalog.schema.modelservice_name"  # model service (nome UC de 3 partes)

# policy de mascaramento de e-mail (PII) aplicada na entrada e na saída
_mask = AiGatewayGuardrailParameters(
    pii=AiGatewayGuardrailPiiBehavior(behavior=AiGatewayGuardrailPiiBehaviorBehavior.MASK)
)

w.serving_endpoints.create(
    name=NOME,
    config=EndpointCoreConfigInput(served_entities=[
        ServedEntityInput(entity_name="system.ai.claude-opus-4-8", scale_to_zero_enabled=True),
    ]),
    ai_gateway=AiGatewayConfig(
        guardrails=AiGatewayGuardrails(input=_mask, output=_mask),
    ),
)
```

**(b) Policy de mascaramento de e-mail (PII)**

O guardrail de **PII com behavior `MASK`** mascara automaticamente e-mails (e demais dados sensíveis)
em prompts e respostas. Pela UI: aba **AI Gateway** do endpoint → **Guardrails → PII → Mask**. Dá para
aplicar/ajustar depois com `w.serving_endpoints.put_ai_gateway(...)`. Como a governança fica **no model
service do LLM** (não no endpoint do agente), ela vale para toda chamada do agente ao modelo.

**(c) Apontar o agente para o model service**

No `02_deploy_agent.py`, além do `genie_space_id`, informe o nome do model service no widget `llm`:

```
genie_space_id = <ID do Genie Agent do passo 3>
llm            = catalog.schema.modelservice_name
```

O agente passa a chamar o LLM por esse model service (via `/ai-gateway/mlflow/v1`, OBO) — então o
mascaramento de e-mail vale para toda a conversa.

---

## O agente híbrido (`agent.py`)

`ResponsesAgent` do Mosaic AI Agent Framework (interface OpenAI Responses API) — compatível ao
mesmo tempo com o **AI Playground** e com o template de **App "Chat UI for existing Agent"**, sem
ajuste no cliente. O próprio LLM (por padrão `system.ai.claude-opus-4-8`) escolhe entre 3
ferramentas via *tool calling* (loop de até 5 rodadas):

| Ferramenta | Tipo | Para quê |
|------------|------|----------|
| `buscar_titulos` | Vector Search (`catalogo_index`) | Descoberta/recomendação por tema ou clima |
| `suporte` | RAG (`ajuda_index`) | Dúvidas de conta/plano/cobrança/cancelamento |
| `consultar_dados` | **Genie Space (API REST, via OBO)** | Perguntas em linguagem natural sobre os números da plataforma (o Genie gera e executa o SQL com governança) |

É um arquivo importável (`from agent import AGENT`), com defaults que já batem com os recursos do
lab — é publicado em Model Serving pelo notebook 02. A ferramenta `consultar_dados` usa a
**API REST do Genie via SDK** (`w.genie.start_conversation_and_wait`): o agente envia a pergunta em
linguagem natural e o Genie monta/executa a consulta — o agente **não** gera SQL. O Genie Space
usado é definido pela variável de ambiente `GENIE_SPACE_ID` (widget no notebook 02).

### Autenticação híbrida (sistema + OBO)

O agente combina dois modos de credencial:

- **Credenciais de sistema** (service principal do endpoint, via *resources* no deploy): usadas
  pelo **Vector Search** (`buscar_titulos`, `suporte`) e pelo endpoint de **embeddings**.
- **On-behalf-of-user (OBO)**: usadas pelo **LLM** e pelo **Genie** (`consultar_dados`), que são
  chamados como o usuário que invoca o agente. No deploy isso aparece como
  `UserAuthPolicy(api_scopes=["model-serving", "ai-gateway", "genie"])`.

> ⚠️ **Pré-requisito (admin):** habilite o preview **"Agent Framework: On-Behalf-Of-User
> Authorization"** antes do deploy, para o Model Serving encaminhar o token do usuário.

### Genie Agent para a ferramenta de dados

1. Rode o notebook **`01_criar_genie_agent.py`** — ele adiciona as chaves PK/FK (dicas de *join*)
   e consolida as **especificações** do Genie (tabelas, instruções, perguntas de exemplo).
2. Peça ao **Genie code**: *"Crie um genie agent baseado nas especificações deste notebook"*.
3. Copie o **ID do Genie Agent** gerado.
4. No notebook **02** (deploy), preencha **os dois widgets**: o `genie_space_id` com esse ID **e** o
   `llm` com o nome do **model service** criado no passo 4 (`catalog.schema.modelservice_name`, com a
   policy de masking de e-mail — ver [Detalhe do passo 4](#detalhe-do-passo-4--model-service-do-llm-e-policy-de-masking-de-e-mail)).

> Como os dados são idênticos entre schemas (geração determinística), um **único Genie Agent
> compartilhado** atende todos os participantes de um hands-on — basta todos usarem o mesmo ID.

---

## 🛡️ Onde entra o Unity AI Gateway

O **Unity AI Gateway** é a camada de governança que fica **na frente dos modelos** que o lab usa —
principalmente o LLM do agente, consumido como **model service** (`system.ai.claude-opus-4-8`) pela
API unificada do gateway (`/ai-gateway/mlflow/v1`). Ele adiciona, de forma centralizada e sem alterar
o código do agente:

- **Guardrails de segurança/PII** — detecção e mascaramento (ou bloqueio) de dados sensíveis em
  prompts e respostas;
- **Rate limiting** — limites de requisição por usuário/endpoint;
- **Usage tracking & payload logging** — registro de todo o tráfego (inferência) em tabelas do
  Unity Catalog para auditoria e análise de custo;
- **Fallbacks** — roteamento para modelos alternativos.

**Como encaixar neste lab:** como o agente chama o LLM pela API unificada do gateway, usando o nome UC
de 3 partes (`system.ai.claude-opus-4-8`), **cada chamada já passa pelo Unity Gateway** — os
guardrails, rate limits e o logging aplicados ao model service valem automaticamente, sem alterar o
código do agente. Você aplica/ajusta essas políticas no próprio model service (ou cria um model
service customizado `catalog.schema.nome` com rate limits, fallbacks e inference tables próprios).

> **Observação importante:** as políticas do Unity AI Gateway (guardrails/*service policies*, rate
> limits, inference tables, fallbacks) são configuradas no **model service do LLM** — **não** no
> endpoint do agente (`netbricks_agent_<schema>`). Por isso a governança é aplicada ao **LLM subjacente** que
> o agente chama.

---

## Sobre o deploy do agente (notebook 02)

O `agents.deploy` publica o agente em Model Serving e, para isso, **cria uma service principal**
para o endpoint. Requisitos: cota de service principals disponível na conta, os *resources*
declarados (índices de Vector Search e o endpoint de embeddings) e os escopos de OBO para o LLM e o
Genie (ver [Autenticação híbrida](#autenticação-híbrida-sistema--obo)).

Se o deploy falhar por limite de recursos da conta (ex.: cota de service principals esgotada), o
modelo já fica **registrado no Unity Catalog** — então o deploy pode ser refeito quando houver
cota, sem precisar logar o modelo de novo.

---

## Objetos criados em `<catalog>.<schema>`

**Tabelas:** `catalogo`, `usuarios`, `eventos_visualizacao`, `central_ajuda_kb`,
`features_usuarios`, `scores_churn`, `scores_upgrade`
**Modelos (UC):** `modelo_churn_<schema>`, `modelo_upgrade_<schema>`, `netbricks_agent_<schema>`
**Vector Search:** endpoint `netbricks_vs_<schema>` + índices `catalogo_index`, `ajuda_index`
**Serving:** endpoint do agente `netbricks_agent_<schema>`
**Model service (LLM):** `catalog.schema.modelservice_name` (Unity AI Gateway sobre `system.ai.claude-opus-4-8`+, com policy de masking de e-mail/PII)
**Workspace:** App `chat-netbricks` + Genie Agent `Netbricks — Genie de Dados`

---

## 🧹 Limpeza (cleanup)

O notebook **`99_limpar_ambiente.py`** remove **tudo** o que o lab cria, na ordem correta de
dependência: o **App `chat-netbricks`**, o endpoint do agente, os índices e o endpoint de Vector
Search, o **Genie Agent `Netbricks — Genie de Dados`** e, por fim, os dados/modelos no Unity
Catalog.

- É **idempotente** (ignora o que já não existe) e **não toca no catálogo** — só no schema do lab.
- É uma **ação destrutiva protegida por confirmação**: nada é apagado enquanto o widget
  **`confirmar`** não estiver em **`sim`**.
- Widgets: `confirmar` (`nao`/`sim`), `drop_schema` (`sim` apaga o schema inteiro com `CASCADE`;
  `nao` remove apenas os objetos conhecidos e preserva o schema), além de `app_name`,
  `genie_space_name` e `genie_space_id` (opcional) — já com os nomes padrão deste lab.

> Se a API não permitir remover o Genie Agent, o notebook indica como apagá-lo pela UI.

---

*Lab de demonstração — plataforma e dados fictícios.*
