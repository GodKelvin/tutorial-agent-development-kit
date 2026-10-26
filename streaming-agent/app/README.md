# Tutorial: ADK Streaming com Google Search

Um agente feito com o [Google ADK](https://google.github.io/adk-docs/) que responde perguntas usando o **Google Search** como ferramenta e conversa em **streaming** (texto, voz e vídeo) pela Gemini Live API.

Diferente do `multi_tool_agent`, aqui não há funções Python escritas por nós: a ferramenta é *built-in* e quem executa a busca é o próprio Gemini.

## Estrutura

```
streaming-agent/
├── .gitignore
└── app/
    ├── .env                    # GOOGLE_API_KEY (não versionado)
    └── google_search_agent/
        ├── __init__.py         # importa o módulo agent, para o ADK encontrar o root_agent
        └── agent.py            # definição do agente
```

## Como rodar

```bash
pip install google-adk

# app/.env
GOOGLE_API_KEY=sua-chave-aqui

cd app
adk web    # interface no navegador, com botões de microfone e câmera
```

No `adk web`, selecione `google_search_agent` no canto superior esquerdo. Dá para digitar normalmente ou clicar no **microfone** para conversar por voz.

> **Antes de rodar:** o campo `model` em `agent.py` está como `"..."`. Preencha com um modelo que suporte a Live API, conforme a [lista de modelos suportados](https://adk.dev/live/get-started/streaming-python/#supported-models). Sem um modelo *live*, o chat por texto pode funcionar, mas voz e vídeo não.

## O que o `agent.py` define

- **O agente** (`root_agent`): nome `basic_search_agent`, o modelo, uma descrição e a instrução de sistema ("You are an expert researcher. You always stick to the facts.").
- **A ferramenta**: `google_search`, importada de `google.adk.tools`.

O `google_search` não é uma função Python que roda na sua máquina. Ao montar o request, o ADK apenas adiciona `types.Tool(google_search=types.GoogleSearch())` à lista de ferramentas enviada ao Gemini. A busca e o *grounding* (embasar a resposta nos resultados) acontecem nos servidores do Google.

Por ser built-in, ele tem uma limitação: por padrão não pode ser combinado com outras ferramentas no mesmo agente. Para isso existe `GoogleSearchTool(bypass_multi_tools_limit=True)`, ou então usar um sub-agente só para a busca.

## Ciclo de execução (streaming)

```
 Usuário fala: "Quem ganhou a última Copa do Mundo?"
    │  áudio via WebSocket (adk web)
    ▼
 ┌──────────────────────────────────────────────┐
 │ 1. Runner.run_live (ADK)                     │
 │    abre uma sessão Live com o Gemini:        │
 │    instruction + tools (google_search)       │
 │    e repassa o áudio/texto em tempo real     │
 └──────────────────────────────────────────────┘
    │  WebSocket bidirecional
    ▼
 ┌──────────────────────────────────────────────┐
 │ 2. Gemini Live API                           │
 │    entende a pergunta, decide buscar,        │
 │    executa o Google Search DO LADO DO GOOGLE │
 │    e gera a resposta embasada                │
 └──────────────────────────────────────────────┘
    │  pedaços da resposta (áudio/texto) chegam
    │  conforme são gerados
    ▼
 Resposta falada/escrita, com metadados de grounding
```

Comparando com o `multi_tool_agent`:

| | `multi_tool_agent` | `google_search_agent` |
|---|---|---|
| Ferramenta | funções Python próprias | `google_search` built-in |
| Onde a ferramenta roda | na sua máquina | nos servidores do Google |
| Ida e volta por ferramenta | function call → ADK executa → nova chamada | nenhuma: tudo dentro de uma resposta |
| Conexão com o Gemini | HTTP, uma request por passo | WebSocket persistente (Live API) |
| Entrada/saída | texto | texto, áudio e vídeo em tempo real |

Pontos importantes:

- **Streaming é bidirecional.** A conexão fica aberta: o usuário pode interromper o modelo no meio da fala e o modelo responde enquanto ainda está gerando.
- **Não há function call visível para a busca.** Nos eventos você vê a resposta final e os `grounding_metadata` (consultas feitas e fontes usadas), não uma chamada/resultado de ferramenta como no outro projeto.

## Onde a chamada ao Gemini acontece

Dentro da biblioteca ADK (caminhos relativos a `site-packages/google/adk/`, versão 2.11.0):

| Arquivo | O que faz |
|---|---|
| `runners.py` → `Runner.run_live` | ponto de entrada do modo streaming: gerencia a sessão e a fila de mensagens ao vivo |
| `flows/llm_flows/base_llm_flow.py` → `run_live` | o loop do agente no modo live: envia entradas ao modelo e recebe eventos |
| `models/google_llm.py` → `Gemini.connect` | **a saída para o Gemini**: abre a sessão com `client.aio.live.connect(...)` |
| `models/gemini_llm_connection.py` | envia e recebe as mensagens dessa sessão Live |
| `tools/google_search_tool.py` | `GoogleSearchTool.process_llm_request`: só adiciona o `GoogleSearch` à lista de tools do request |

## Como observar o ciclo

- **`adk web`** → aba **Events**: mostra os eventos recebidos em streaming, incluindo os `grounding_metadata` com as buscas feitas.
- **Breakpoint** em `Gemini.connect` (`models/google_llm.py`) para inspecionar a configuração enviada à Live API (instrução, tools, modalidades de resposta).
