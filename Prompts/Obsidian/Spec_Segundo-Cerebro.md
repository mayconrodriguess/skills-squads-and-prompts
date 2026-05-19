# Segundo Cérebro com IA — Procedimento Completo

> **Este documento é executável.** Foi escrito para ser seguido por humanos OU por uma IA agente (Claude, GPT, Gemini, etc.) com acesso a terminal.
>
> **Objetivo final:** qualquer IA com suporte a MCP consegue ler, buscar, criar, editar e gerenciar notas Obsidian autonomamente.

---

## Arquitetura

```
Vault Obsidian (.md no disco/OneDrive)
        │
        ▼
Plugin "Local REST API"  ──►  HTTP em 127.0.0.1:27123 (auth via API key)
        │
        ▼
Servidor MCP "obsidian-mcp-server" (cyanheads, via npx)
        │
        ├──► Claude Desktop
        ├──► Claude Code (CLI)
        ├──► VS Code Copilot Agent
        ├──► Cursor
        └──► Qualquer cliente MCP-compatível
```

**Capacidades expostas:** ler nota, criar nota, editar nota, append, buscar (texto/regex/Dataview), gerenciar tags, manipular frontmatter, listar pastas, mover/renomear arquivos.

---

# Obsidian MCP - Auto Instalacao (AI) - Copiar e colar em sua IA

> [!IMPORTANT]
> Regra de execucao: antes de criar qualquer diretorio que nao exista, pergunte ao usuario se tem permissao para cria-lo.

## Objetivo

Configurar agentes de IA para acessar o vault do Obsidian pelo MCP nativo do plugin **Local REST API**.

Desde as versoes recentes do plugin, nao e necessario instalar servidor MCP de terceiro para Obsidian. O proprio plugin expoe o servidor MCP em:

```text
http://127.0.0.1:27123/mcp/
```

Use servidores terceiros apenas como fallback para clientes que nao suportam Streamable HTTP MCP.

## Arquitetura recomendada

```text
Cliente MCP / Agente IA
  -> Streamable HTTP MCP
    -> http://127.0.0.1:27123/mcp/
      -> Obsidian Local REST API
        -> Vault aberto no Obsidian
```

## Requisitos

- Obsidian instalado.
- Vault correto aberto no Obsidian.
- Plugin comunitario **Local REST API** instalado e ativado.
- HTTP server do plugin ativado na porta `27123`.
- API key copiada em `Settings -> Local REST API`.
- Cliente MCP com suporte a Streamable HTTP ou bridge `mcp-remote`.

## Passo 1 - Instalar o plugin Local REST API

Instale pelo Obsidian:

```text
Settings -> Community Plugins -> Browse -> Local REST API -> Install -> Enable
```

Depois abra:

```text
Settings -> Local REST API
```

Confirme:

- API key gerada.
- HTTP server habilitado.
- Porta HTTP `27123`.
- Se usar HTTPS, porta comum `27124` com certificado autoassinado.

## Passo 2 - Validar REST API

Servidor ativo, sem autenticacao:

```powershell
curl.exe -k http://127.0.0.1:27123/
```

Listar raiz do vault, com autenticacao:

```powershell
curl.exe -k -H "Authorization: Bearer <sua-api-key>" http://127.0.0.1:27123/vault/
```

Ler uma nota:

```powershell
curl.exe -k -H "Authorization: Bearer <sua-api-key>" http://127.0.0.1:27123/vault/path/to/note.md
```

> [!WARNING]
> Nunca grave API key em nota do vault, repositorio ou documento sincronizado. Use variavel de ambiente ou arquivo local fora do vault.

## Passo 3 - Validar MCP nativo

O endpoint MCP usa Streamable HTTP e exige header `Authorization`.

Um `GET` simples em `/mcp/` pode retornar `406 Not Acceptable` se o cliente nao enviar `Accept: application/json, text/event-stream`. Isso e normal.

Teste real de handshake MCP:

```powershell
$body = '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"mcp-check","version":"0.0.1"}}}'
$bodyPath = Join-Path $env:TEMP "obsidian-mcp-init.json"
$body | Set-Content -Path $bodyPath -NoNewline -Encoding ASCII

curl.exe -k -X POST `
  -H "Authorization: Bearer <sua-api-key>" `
  -H "Content-Type: application/json" `
  -H "Accept: application/json, text/event-stream" `
  --data-binary "@$bodyPath" `
  http://127.0.0.1:27123/mcp/
```

Resultado esperado:

```json
{
  "serverInfo": {
    "name": "obsidian-local-rest-api",
    "version": "1.0.0"
  }
}
```

## Passo 4 - Configurar Codex

Edite:

```text
C:\Users\Maycon\.codex\config.toml
```

Use a API key em variavel de ambiente do usuario, nao no TOML:

```powershell
[Environment]::SetEnvironmentVariable("OBSIDIAN_API_KEY", "<sua-api-key>", "User")
```

Adicione ao `config.toml`:

```toml
[mcp_servers.obsidian]
url = "http://127.0.0.1:27123/mcp/"
bearer_token_env_var = "OBSIDIAN_API_KEY"
startup_timeout_sec = 30
tool_timeout_sec = 60
enabled = true
```

Depois reinicie o Codex para carregar a variavel de ambiente e o servidor MCP.

## Passo 5 - Configurar Claude Code

Via CLI:

```bash
claude mcp add --transport http obsidian http://127.0.0.1:27123/mcp/ \
  --header "Authorization: Bearer <sua-api-key>"
```

Ou em configuracao JSON:

```json
{
  "mcpServers": {
    "obsidian": {
      "type": "http",
      "url": "http://127.0.0.1:27123/mcp/",
      "headers": {
        "Authorization": "Bearer <sua-api-key>"
      }
    }
  }
}
```

## Passo 6 - Configurar Claude Desktop

Claude Desktop nao suporta HTTP MCP remoto de forma nativa em algumas instalacoes. Use `mcp-remote` como bridge.

Arquivo Windows:

```text
%APPDATA%\Claude\claude_desktop_config.json
```

Configuracao:

```json
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": [
        "mcp-remote@latest",
        "http://127.0.0.1:27123/mcp/",
        "--header",
        "Authorization: Bearer <sua-api-key>"
      ]
    }
  }
}
```

Reinicie o Claude Desktop apos salvar.

## Passo 7 - Configurar Cursor

Arquivo global ou de projeto:

```text
~/.cursor/mcp.json
.cursor/mcp.json
```

Configuracao:

```json
{
  "mcpServers": {
    "obsidian": {
      "url": "http://127.0.0.1:27123/mcp/",
      "headers": {
        "Authorization": "Bearer <sua-api-key>"
      }
    }
  }
}
```

## Tools MCP disponiveis

| Tool | Descricao |
|------|-----------|
| `vault_list` | Lista arquivos e subdiretorios. |
| `vault_read` | Le conteudo, frontmatter, tags, links e stats. |
| `vault_write` | Cria ou sobrescreve arquivo. |
| `vault_append` | Adiciona conteudo ao final do arquivo. |
| `vault_patch` | Edita heading, block reference ou frontmatter especifico. |
| `vault_delete` | Deleta arquivo do vault. |
| `vault_get_document_map` | Lista headings, block refs e campos de frontmatter. |
| `active_file_get_path` | Retorna caminho do arquivo aberto. |
| `periodic_note_get_path` | Retorna caminho da nota periodica atual. |
| `search_query` | Busca via JsonLogic em metadados. |
| `search_simple` | Busca full-text nativa do Obsidian. |
| `tag_list` | Lista tags com contagem. |
| `command_list` | Lista comandos do Obsidian. |
| `command_execute` | Executa comando por ID. |
| `open_file` | Abre arquivo na UI do Obsidian. |

## Endpoints REST uteis

| Endpoint | Metodos | Descricao |
|----------|---------|-----------|
| `/vault/{path}` | GET PUT PATCH POST DELETE | Ler, escrever ou deletar arquivo. |
| `/active/` | GET PUT PATCH POST DELETE | Operar no arquivo ativo. |
| `/periodic/{period}/` | GET PUT PATCH POST DELETE | Nota periodica atual. |
| `/search/simple/` | POST | Busca full-text. |
| `/search/` | POST | Busca estruturada via JsonLogic. |
| `/commands/` | GET | Listar comandos. |
| `/commands/{commandId}` | POST | Executar comando. |
| `/tags/` | GET | Listar tags. |
| `/open/{path}` | POST | Abrir arquivo na UI. |
| `/` | GET | Status do servidor e auth. |
| `/mcp/` | GET POST | Servidor MCP nativo do plugin. |

## Diagnostico rapido

| Sintoma | Causa provavel | Correcao |
|---------|----------------|----------|
| `401 Unauthorized` | API key errada ou vault diferente aberto. | Copiar a chave do vault ativo em Settings -> Local REST API. |
| `ECONNREFUSED` | Obsidian fechado ou HTTP server desligado. | Abrir Obsidian e ativar HTTP server. |
| `406 Not Acceptable` em GET `/mcp/` | Header `Accept` ausente. | Normal; testar com POST initialize ou cliente MCP real. |
| Cliente nao lista tools | Cliente nao reiniciado ou variavel de ambiente nao carregada. | Reiniciar cliente e validar env. |
| Certificado invalido | Uso de HTTPS autoassinado. | Usar HTTP 27123 local ou confiar o certificado. |

## Estado desta instalacao

| Item | Valor |
|------|-------|
| Vault ativo observado | `G:\Meu Drive\.01_OBSIDIAN` |
| HTTP MCP | `http://127.0.0.1:27123/mcp/` |
| Auth | `Authorization: Bearer <api-key>` |
| Codex | Usa `bearer_token_env_var = "OBSIDIAN_API_KEY"` |
| Servidor terceiro | Nao necessario para o fluxo principal |

## Relacionados

- [[Obsidian MCP - Guia Manual (Humano)]]
- [[Obsidian MCP - README]]
- [[Procedimento - Obsidian sempre conectado via MCP]]
