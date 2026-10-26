# Tutorial: Agent Development Kit (ADK)

Exemplos de estudo do [Google ADK](https://google.github.io/adk-docs/), o framework do Google para criar agentes com o Gemini. Cada pasta é um projeto independente, com seu próprio README detalhado.

## Projetos

| Projeto | O que faz | Ferramentas | Conexão com o Gemini |
|---|---|---|---|
| [multi_tool_agent](multi_tool_agent/README.md) | responde sobre **clima** e **hora** de uma cidade | funções Python próprias, executadas localmente | HTTP, uma request por passo |
| [streaming-agent](streaming-agent/app/README.md) | responde perguntas usando o **Google Search**, por texto, voz ou vídeo | `google_search` built-in, executado pelo Google | WebSocket persistente (Live API) |

### multi_tool_agent

Mostra o ciclo básico de um agente com ferramentas: o Gemini pede uma *function call*, o ADK executa a função Python na sua máquina e devolve o resultado para o modelo gerar a resposta final. É o melhor ponto de partida para entender como o ADK transforma docstrings e parâmetros em declarações de ferramentas.

### streaming-agent

Mostra o modo **streaming** do ADK: uma sessão bidirecional com a Gemini Live API, em que o usuário pode falar e interromper o modelo. A ferramenta `google_search` é nativa do Gemini, então a busca acontece nos servidores do Google, sem ida e volta pelo seu código.

> O campo `model` em `streaming-agent/app/google_search_agent/agent.py` precisa ser preenchido com um modelo que suporte a Live API antes de rodar.

## Requisitos

- Python 3.10+
- `pip install google-adk`
- Uma `GOOGLE_API_KEY` do [Google AI Studio](https://aistudio.google.com/apikey), colocada em um arquivo `.env` (não versionado) de cada projeto

## Como rodar

```bash
# multi_tool_agent: a partir da raiz do repositório
adk web
adk run multi_tool_agent

# streaming-agent: a partir da pasta app
cd streaming-agent/app
adk web
```

O `adk web` abre uma interface no navegador com a aba **Events**, onde dá para acompanhar cada request, resposta e chamada de ferramenta trocada com o Gemini.
