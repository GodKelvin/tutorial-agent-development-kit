# Tutorial: Agent Development Kit (ADK)

Um agente simples, feito com o [Google ADK](https://google.github.io/adk-docs/), que responde perguntas sobre **clima** e **hora** de uma cidade. Ele usa o Gemini como modelo e duas funções Python como ferramentas.

## Estrutura

```
multi_tool_agent/
├── __init__.py   # importa o módulo agent, para o ADK encontrar o root_agent
├── agent.py      # ferramentas + definição do agente
└── .env          # GOOGLE_API_KEY (não versionado)
```

## Como rodar

```bash
pip install google-adk

# multi_tool_agent/.env
GOOGLE_API_KEY=sua-chave-aqui

# a partir da raiz do repositório
adk web    # interface no navegador (com aba de Events/Trace)
adk run multi_tool_agent   # no terminal
```

## O que o `agent.py` define

O arquivo **não chama o Gemini diretamente**. Ele apenas declara:

- **Ferramentas**: `get_weather(city)` e `get_current_time(city)`, funções Python comuns que devolvem um `dict` com `status` e `report` (ou `error_message`). Hoje só funcionam para `"new york"`.
- **O agente** (`root_agent`): qual modelo usar (`gemini-flash-latest`), a instrução de sistema e a lista de ferramentas.

O ADK lê o **nome, os parâmetros tipados e a docstring** de cada função e os converte em uma *function declaration* enviada ao Gemini. É pela docstring que o modelo decide quando e como chamar cada ferramenta, então ela precisa ser clara.

## Ciclo de execução

```
 Usuário: "Que horas são em New York?"
    │
    ▼
 ┌──────────────────────────────────────────────┐
 │ 1. Runner (ADK)                              │
 │    monta o request:                          │
 │    instruction + histórico + declarações     │
 │    das tools                                 │
 └──────────────────────────────────────────────┘
    │  HTTP
    ▼
 ┌──────────────────────────────────────────────┐
 │ 2. Gemini API                                │
 │    responde com uma function call:           │
 │    get_current_time(city="New York")         │
 └──────────────────────────────────────────────┘
    │
    ▼
 ┌──────────────────────────────────────────────┐
 │ 3. ADK executa a função Python LOCALMENTE    │
 │    → {"status": "success", "report": "..."}  │
 └──────────────────────────────────────────────┘
    │  resultado volta como function response
    ▼
 ┌──────────────────────────────────────────────┐
 │ 4. Gemini API (nova chamada)                 │
 │    lê o resultado e gera o texto final       │
 └──────────────────────────────────────────────┘
    │
    ▼
 Resposta: "Agora são 11:32 em New York."
```

Pontos importantes:

- **As ferramentas rodam na sua máquina.** O Gemini nunca executa código. Ele só devolve *qual* função chamar e *com quais argumentos*.
- **Uma pergunta pode gerar várias chamadas ao modelo.** Os passos 2 e 3 se repetem enquanto o Gemini pedir ferramentas. Por exemplo, "clima e hora em New York" pode disparar as duas tools antes da resposta final.
- **Erros também são resultados.** Se a função devolve `status: "error"`, o Gemini recebe a mensagem e a explica ao usuário em vez de quebrar.

## Onde a chamada ao Gemini acontece

Dentro da biblioteca ADK (caminhos relativos a `site-packages/google/adk/`, versão 2.11.0):

| Arquivo | O que faz |
|---|---|
| `runners.py` | `Runner`: recebe a mensagem do usuário, gerencia a sessão e dispara o agente |
| `flows/llm_flows/base_llm_flow.py` → `_call_llm_async` | o loop do agente: monta o request, chama o modelo, executa tools e repete |
| `models/google_llm.py` → `Gemini.generate_content_async` | **a saída para o Gemini**: chama `api_client.aio.models.generate_content(...)` (ou `generate_content_stream` em modo streaming) |

O `api_client` é um `google.genai.Client`, o SDK oficial do Gemini, que faz o HTTP para `generativelanguage.googleapis.com` usando a `GOOGLE_API_KEY` do `.env`.

## Como observar o ciclo

- **`adk web`** → aba **Events / Trace**: mostra cada request e response trocado com o Gemini, incluindo as function calls e os resultados das tools.
- **Breakpoint** em `Gemini.generate_content_async` (`models/google_llm.py`) para inspecionar o request exato que vai para a API.
