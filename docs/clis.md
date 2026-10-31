# CLIs de desenvolvimento

Execute os comandos deste guia na raiz do projeto, onde está `docker-compose.yaml`. É necessário ter Docker e Docker Compose instalados. A configuração geral e as limitações da arquitetura estão no [README](../README.md).

## Preparação

Se ainda não existir, copie o arquivo de exemplo e edite-o:

```sh
cp .env.example .env
openssl rand -hex 32
```

Coloque o resultado do segundo comando em `AI_MEMORY_AUTH_TOKEN` e preencha `OPENROUTER_API_KEY` no `.env`. Não substitua um `.env` já configurado pelo exemplo. A chave da OpenRouter configura o LLM do ai-memory; ela não autentica automaticamente nenhuma das quatro CLIs.

Coloque os projetos em `src`: eles aparecerão em `/workspace` nos contêineres. O usuário UID 1000 deve conseguir escrever em `src`; `memory_data` deve pertencer ao UID/GID 1000. Cada CLI mantém suas configurações em um volume próprio, preservado ao recriar o contêiner.

Para construir e iniciar tudo:

```sh
docker compose --profile agents up -d --build
```

Para iniciar uma única CLI e o ai-memory, substitua `codex` pelo serviço desejado:

```sh
docker compose --profile agents up -d --build codex
```

Para inspecionar o estado e parar os serviços:

```sh
docker compose --profile agents ps
docker compose --profile agents down
```

`down` preserva os volumes. Evite a opção `-v` se quiser manter as credenciais e configurações das CLIs.

## Codex

Inicie o serviço e autentique por código de dispositivo:

```sh
docker compose --profile agents up -d --build codex
docker compose exec codex codex login --device-auth
```

Abra no navegador do host o endereço apresentado e informe o código exibido no terminal. Em contas gerenciadas, a organização pode precisar habilitar esse método. Confira o estado e abra a CLI:

```sh
docker compose exec codex codex login status
docker compose exec codex codex
```

O login comum pode tentar retornar a `localhost` dentro do contêiner, que é diferente do `localhost` no navegador do host. O fluxo por dispositivo evita esse retorno. Caso apareça `failed to invoke ps`, reconstrua e recrie o serviço para usar a imagem com `procps`:

```sh
docker compose --profile agents up -d --build --no-deps --force-recreate codex
docker compose exec codex codex
```

Como alternativa temporária ao daemon, a mensagem de erro indica `docker compose exec codex codex --no-daemon`. Consulte também a [documentação do Codex](https://developers.openai.com/codex/).

Se o Codex não conseguir editar o workspace, confira se a sessão está em modo de escrita e inicie uma nova sessão com `docker compose exec codex codex -s workspace-write`. O serviço usa `seccomp=unconfined` para permitir que o sandbox interno crie namespaces de usuário; isso desativa o filtro seccomp padrão do Docker para esse contêiner. As capacidades continuam removidas e `no-new-privileges` permanece ativo. Confirme também que `src` e o diretório de destino permitem escrita ao UID 1000; arquivos do host fora de `src` não estão montados.

## Claude Code

```sh
docker compose --profile agents up -d --build claude
docker compose exec claude claude
```

Na primeira execução, selecione o método de login e siga as instruções exibidas. Se uma URL for apresentada, abra-a no navegador do host e siga o fluxo indicado pela CLI. Caso a instalação ofereça um código para copiar, cole-o no terminal. O acesso depende da conta ou do provedor selecionado; veja a [documentação de autenticação](https://code.claude.com/docs/en/authentication). Para usar o projeto, faça uma solicitação no prompt depois de autenticar.

## Antigravity CLI

```sh
docker compose --profile agents up -d --build antigravity
docker compose exec antigravity agy
```

Siga o login apresentado pela CLI. O fluxo documentado para SSH remoto exibe uma URL e pede o código obtido no navegador; um contêiner executado com `docker compose exec` não é necessariamente detectado como SSH e o login com conta pode depender de um keyring/D-Bus disponível. Consulte a [documentação de autenticação](https://antigravity.google/docs/cli/install/) se o fluxo de navegador ou keyring falhar.

Como alternativa sem navegador, a documentação permite usar uma chave Gemini e selecionar o provedor `gemini`. Defina `GEMINI_API_KEY` no ambiente do **terminal do host**, sem colocá-la nos arquivos do projeto. Em seguida, configure o provedor no volume pessoal e passe a variável pelo nome:

```sh
docker compose exec antigravity sh -c 'mkdir -p "$HOME/.gemini/antigravity-cli"; test ! -e "$HOME/.gemini/antigravity-cli/settings.json" && printf "{\"modelProvider\":\"gemini\"}\n" > "$HOME/.gemini/antigravity-cli/settings.json" || echo "settings.json já existe; configure modelProvider manualmente"'
docker compose exec -e GEMINI_API_KEY antigravity agy
```

Se o arquivo já existir, acrescente `"modelProvider": "gemini"` ao JSON existente sem apagar outras configurações. A variável precisa estar disponível no terminal do host em cada execução que usar `-e GEMINI_API_KEY`. A disponibilidade e cobrança dependem da chave e da conta Google.

## OpenCode

```sh
docker compose --profile agents up -d --build opencode
docker compose exec opencode opencode
```

Na interface, use `/connect`, selecione o provedor e siga a autenticação apresentada; depois use `/models` para selecionar um modelo. Por exemplo, é possível selecionar OpenRouter e informar uma chave para o OpenCode separadamente da configuração do ai-memory. Se um método de OAuth exigir callback em `localhost` inacessível, escolha um método de código de dispositivo ou chave de API oferecido pelo provedor. Veja a [documentação de provedores](https://opencode.ai/docs/providers/).

## Rede, workspace e ai-memory

As quatro CLIs e o sandbox têm acesso de saída à internet. Não é necessário publicar uma porta para usar APIs externas. Cada agente vê `src` como `/workspace`, além dos próprios arquivos e configurações do contêiner; não há outros diretórios de projetos do host montados. Serviços acessíveis na rede podem continuar acessíveis aos contêineres. As CLIs rodam como usuário sem privilégios de administrador; instalar pacotes globalmente pode exigir um diretório de usuário ou alteração da imagem.

O endpoint MCP do ai-memory dentro das CLIs é `http://ai-memory:49374/mcp`, com cabeçalho `Authorization: Bearer <AI_MEMORY_AUTH_TOKEN>`. **Ainda é necessário [registrar o servidor em cada cliente](mcp-ai-memory.md)** e configurar hooks separadamente para captura automática. `AI_MEMORY_URL` por si só não registra o MCP. Veja a [instalação do ai-memory](https://github.com/akitaonrails/ai-memory/blob/main/docs/install.md).

O sandbox compartilha `/workspace`, mas nenhum agente encaminha comandos para ele automaticamente. Para usá-lo manualmente:

```sh
docker compose up -d ai-sandbox
docker compose exec ai-sandbox bash
```

Dentro dele, execute testes ou instale dependências. Mudanças em `/workspace` persistem em `src`; pacotes instalados fora do workspace são perdidos ao recriar o contêiner.
