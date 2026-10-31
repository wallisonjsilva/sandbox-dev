A instalação das ferramentas depende do papel que cada uma desempenha na arquitetura. A regra de ouro dessa separação é: **quem pensa não executa, e quem executa não tem acesso ao sistema.**

Aqui está onde e como instalar cada tipo de ferramenta:

### 1. No `ai-sandbox` (O Ambiente de Execução)

**O que instalar aqui:** Compiladores, runtimes e ferramentas de build que os agentes vão usar para testar o código gerado (ex: Python, Node.js, Go, Rust, compiladores C#, utilitários como `git`, `curl` ou `jq`).
**Como instalar:** Em vez de usar uma imagem pronta no `docker-compose.yml`, você cria um `Dockerfile` customizado para o sandbox.

Crie um arquivo `Dockerfile.sandbox`:

```dockerfile
FROM python:3.11-slim

# Instala ferramentas básicas do sistema e compiladores
RUN apt-get update && apt-get install -y \
    curl \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Exemplo: Instalando Node e TypeScript para o agente poder rodar scripts JS/TS
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g typescript ts-node

# Exemplo: Instalando Rust (se seus agentes forem compilar ferramentas em Rust)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

WORKDIR /workspace
CMD ["tail", "-f", "/dev/null"]

```

No seu `docker-compose.yml`, você troca a linha `image: python:3.11-slim` por `build: context: . dockerfile: Dockerfile.sandbox`.

### 2. No `ai-orchestrator` (O Cérebro)

**O que instalar aqui:** As ferramentas de IA de linha de comando (Claude Code, OpenCode CLI, clientes MCP) e as bibliotecas do seu orquestrador.
**Como instalar:** Criando um `Dockerfile` focado nas ferramentas de IA.

Crie um arquivo `Dockerfile.orchestrator`:

```dockerfile
FROM node:20-alpine

# Instala ferramentas CLI baseadas em Node que você usa para orquestrar
RUN npm install -g @anthropic-ai/claude-code
# Se usar ferramentas em Python no orquestrador (ex: LangChain, Ollama client)
RUN apk add --no-cache python3 py3-pip

WORKDIR /app
CMD ["tail", "-f", "/dev/null"]

```

No `docker-compose.yml`, sob `ai-orchestrator`, use o `build: context: . dockerfile: Dockerfile.orchestrator`. Para usar o Claude Code, por exemplo, você entra neste contêiner (`docker exec -it ai_orchestrator bash`) e roda os comandos lá dentro, já com acesso direto ao volume `/workspace` e apontando para o ai-memory no host `http://ai-memory:49374`.

### 3. No seu Host (Direto no CachyOS Linux)

**O que instalar aqui:** Suas IDEs visuais e terminais locais.

* **Cursor / VSCode:** Ficam instalados direto no seu sistema operacional. Eles acessam o código que os agentes geram porque você mapeou o diretório usando volumes (`- ./workspace_data:/workspace`).
* **Ollama:** Se você for rodar modelos locais, é melhor instalar o Ollama nativamente no host (para ter acesso direto à aceleração da sua Radeon RX 7600) e expor a porta para a rede `agent_net` do Docker, em vez de tentar rodar a GPU dentro do contêiner do orquestrador.

### Resumo do Fluxo de Trabalho

1. Você abre o **Cursor** no seu host, editando os arquivos locais.
2. Você acessa o terminal do **`ai-orchestrator`** e roda o Claude Code ou OpenCode, pedindo para criar uma funcionalidade em Go ou Python.
3. O orquestrador salva o contexto no **`ai-memory`** e escreve o código no `/workspace`.
4. O orquestrador envia um comando via Docker/SSH para o **`ai-sandbox`** compilar e testar o código com segurança.