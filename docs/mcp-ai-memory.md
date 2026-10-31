# Registrar o MCP do ai-memory

Este guia registra o servidor em cada CLI executada pelo Compose. Ele não configura os hooks de captura automática de sessões. Use os comandos na raiz deste projeto.

## Antes de começar

Configure `AI_MEMORY_AUTH_TOKEN` e `OPENROUTER_API_KEY` no `.env` e inicie os serviços desejados:

```sh
docker compose --profile agents up -d ai-memory codex claude antigravity opencode
```

O Compose passa o token para cada CLI em `AI_MEMORY_AUTH_TOKEN`. O endpoint é `http://ai-memory:49374/mcp` **dentro dos contêineres**; no host, o endereço é `http://127.0.0.1:49374/mcp`. As CLIs devem ter sido autenticadas conforme [clis.md](clis.md). Não cole o token no código nem em arquivos versionados.

## Codex

Registre o servidor no volume de configuração do Codex. O token será lido do ambiente ao conectar:

```sh
docker compose exec codex codex mcp add ai-memory --url http://ai-memory:49374/mcp --bearer-token-env-var AI_MEMORY_AUTH_TOKEN
docker compose exec codex codex mcp list
```

Abra uma sessão nova com `docker compose exec codex codex` e use `/mcp` para verificar o servidor e suas ferramentas. A listagem confirma o registro; a conexão real deve aparecer como disponível na sessão.

## Claude Code

```sh
docker compose exec claude sh -c 'claude mcp add --scope user --transport http --header "Authorization: Bearer $AI_MEMORY_AUTH_TOKEN" ai-memory http://ai-memory:49374/mcp'
docker compose exec claude claude mcp list
```

O comando executa a expansão do token dentro do contêiner, sem exigir que ele seja digitado no terminal do host. O cabeçalho resultante fica gravado na configuração pessoal do Claude, no volume `claude_home`; não compartilhe essa configuração ou a saída de comandos que possam exibir o cabeçalho. Abra uma nova sessão `docker compose exec claude claude` e consulte `/mcp` para conferir a conexão.

## Antigravity CLI

```sh
docker compose exec antigravity sh -c 'agy mcp add --type http --header "Authorization: Bearer $AI_MEMORY_AUTH_TOKEN" ai-memory http://ai-memory:49374/mcp'
docker compose exec antigravity agy mcp list
```

O cabeçalho resultante fica gravado na configuração pessoal do Antigravity, no volume `antigravity_home`. Não compartilhe o arquivo de configuração nem saídas que possam mostrar o token. Abra uma nova sessão com `docker compose exec antigravity agy` e consulte `/mcp` para verificar a conexão.

## OpenCode

A imagem deste projeto instala o OpenCode 1.x. Adicione o objeto abaixo à chave `mcp` do arquivo global `/home/node/.config/opencode/opencode.json` **dentro do contêiner**. Se o arquivo ou a chave já existirem, preserve as outras configurações e mescle a entrada. O token é lido do ambiente ao abrir o OpenCode.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "ai-memory": {
      "type": "remote",
      "url": "http://ai-memory:49374/mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:AI_MEMORY_AUTH_TOKEN}"
      }
    }
  }
}
```

Para criar ou mesclar a entrada sem substituir as outras chaves de um arquivo JSON existente, execute:

```sh
docker compose exec -T opencode node <<'NODE'
const fs = require('fs');
const path = require('path');
const file = path.join(process.env.HOME, '.config/opencode/opencode.json');
fs.mkdirSync(path.dirname(file), { recursive: true });
const config = fs.existsSync(file) ? JSON.parse(fs.readFileSync(file, 'utf8')) : {};
config.mcp ??= {};
config.mcp['ai-memory'] = {
  type: 'remote',
  url: 'http://ai-memory:49374/mcp',
  enabled: true,
  oauth: false,
  headers: { Authorization: 'Bearer {env:AI_MEMORY_AUTH_TOKEN}' },
};
fs.writeFileSync(file, JSON.stringify(config, null, 2) + '\n');
NODE
docker compose exec opencode opencode mcp list
```

O script substitui somente a entrada `ai-memory`; se o JSON existente for inválido, ele falha sem gravar. Abra uma nova sessão com `docker compose exec opencode opencode` e confira o estado do servidor.

## Conferência e troca de token

Peça ao agente, por exemplo: “Use o ai-memory para consultar as memórias disponíveis deste projeto.” Verifique na interface se a ferramenta MCP foi chamada e se a resposta veio do servidor. Uma lista de servidores registrados não garante que a autenticação ou a chamada da ferramenta funcionou.

Se alterar o token no `.env`, recrie o servidor e os clientes para receberem o novo ambiente:

```sh
docker compose --profile agents up -d --force-recreate ai-memory codex claude antigravity opencode
```

Codex e OpenCode leem a variável atual. No Claude, remova o registro antigo com `docker compose exec claude claude mcp remove --scope user ai-memory` e execute novamente o comando de registro acima. No Antigravity, execute novamente o comando de registro acima, que atualiza a entrada existente. Reinicie as sessões interativas para carregar a alteração. Preserve os volumes para manter logins e configurações.

Se houver erro, confira `docker compose logs ai-memory`, a existência do token no `.env`, a URL interna e o estado exibido pelo cliente. Não envie tokens ou logs com credenciais para terceiros. A configuração MCP não instala os [hooks do ai-memory](https://github.com/akitaonrails/ai-memory/blob/main/docs/install.md); esses têm configuração própria.

Referências: [Codex MCP](https://developers.openai.com/resources/docs-mcp), [Claude Code MCP](https://code.claude.com/docs/en/mcp), [Antigravity MCP](https://antigravity.google/docs/mcp), [OpenCode MCP 1.x](https://opencode.ai/docs/mcp-servers/).
