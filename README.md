# Ambiente de desenvolvimento com agentes

Para instruções de inicialização, autenticação e uso de cada CLI, consulte [docs/clis.md](docs/clis.md).
Para registrar o ai-memory nas CLIs, consulte [docs/mcp-ai-memory.md](docs/mcp-ai-memory.md).

Copie `.env.example` para `.env`, informe a chave da OpenRouter e gere um token com `openssl rand -hex 32`. O modelo pode ser alterado em `AI_MEMORY_LLM_MODEL`. A chave da OpenRouter é usada pelo ai-memory; cada CLI requer sua própria autenticação.

O diretório `memory_data` deve pertencer ao UID/GID 1000, utilizado pela imagem do ai-memory. Garanta também que o usuário UID 1000 consiga escrever em `src`.

```sh
cp .env.example .env
# Edite .env e preencha a chave e o token antes de continuar.
docker compose --profile agents up -d --build
docker compose exec codex codex login --device-auth
docker compose exec codex codex
docker compose exec claude claude
docker compose exec antigravity agy
docker compose exec opencode opencode
```

Os agentes usam `src` como `/workspace`; cada um tem um volume próprio para preservar autenticação e configuração. A porta do ai-memory é exposta apenas no loopback do host. Para iniciar apenas um agente, use, por exemplo, `docker compose --profile agents up -d --build codex ai-sandbox`.

Para autenticar o Codex, abra no navegador do host o endereço apresentado por `codex login --device-auth` e informe o código exibido no terminal. Esse fluxo evita o callback em `localhost` do contêiner. Em contas gerenciadas, a organização pode precisar habilitar a autenticação por código de dispositivo.

O endpoint MCP dentro dos contêineres é `http://ai-memory:49374/mcp`, com cabeçalho `Authorization: Bearer <AI_MEMORY_AUTH_TOKEN>`. Configure esse servidor em cada CLI que for usá-lo, seguindo a documentação do cliente e do [ai-memory](https://github.com/akitaonrails/ai-memory/blob/main/docs/install.md). A variável `AI_MEMORY_URL` é uma referência para configuração manual: ela não registra o servidor automaticamente. A captura automática e os hooks também exigem instalação e configuração próprias.

Para executar testes, use `docker compose exec ai-sandbox bash` e execute os comandos em `/workspace`. Tanto o sandbox quanto os agentes têm acesso à internet para baixar dependências e usar APIs. Os projetos do host só são montados a partir de `src`; instalações feitas nos contêineres não instalam pacotes no sistema do host. Arquivos criados em `/workspace` permanecem em `src`. As CLIs rodam sem privilégios de administrador; instalações globais podem exigir configuração de um diretório de usuário ou alteração do Dockerfile. O sandbox roda como root dentro do contêiner e pode instalar pacotes nele, mas essas instalações são perdidas quando ele é recriado.

Não há encaminhamento automático de comandos dos agentes para o sandbox: as CLIs também podem executar comandos no próprio contêiner. Além do projeto montado, cada agente pode acessar os arquivos e configurações dentro de seu próprio contêiner, inclusive seu volume pessoal. A separação descrita em `docs/esquema.md` não é garantida por esta configuração; o acesso à rede também permite alcançar serviços acessíveis pela rede.

As imagens das CLIs instalam as versões atuais durante o build. Para repetir um ambiente específico, fixe as versões das dependências e a tag de `AI_MEMORY_IMAGE` antes de reconstruir.
