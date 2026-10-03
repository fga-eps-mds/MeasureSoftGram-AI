# MeasureSoftGram AI (MCP Server)

Servidor **MCP (Model Context Protocol)** oficial que expõe os dados e análises de qualidade do **MeasureSoftGram** como ferramentas (*tools*) para modelos de linguagem (LLMs) e agentes de IA (Claude Code, Claude Desktop, Cursor, VS Code Copilot, Antigravity, etc.).

## O que é possível fazer

Com esse servidor MCP conectado, um LLM consegue:

- Listar organizações, produtos, repositórios e releases cadastrados
- Consultar características, subcaracterísticas, medidas e métricas suportadas
- Verificar matrizes de balanceamento, metas (*goals*) e configurações de releases
- Acessar séries históricas, últimos valores calculados (*TSQMI*) e comparativos entre planejado e realizado
- Navegar pela árvore de relacionamentos entre entidades do modelo de qualidade do MeasureSoftGram

---

## Variáveis de Ambiente do Contêiner MCP

O servidor MCP autentica automaticamente na API do `MeasureSoftGram-Service` durante a inicialização utilizando as variáveis de ambiente abaixo:

| Variável | Obrigatória | Padrão | Descrição |
| :--- | :---: | :---: | :--- |
| `SERVICE` | **Sim** | — | URL base da API v1 do `MeasureSoftGram-Service` (ex.: `https://msgram.lappis.rocks/api/v1/` para produção ou `http://service:8080/api/v1/` na rede Docker local). |
| `MSGRAM_USER` | **Sim** | — | Nome de usuário cadastrado no `MeasureSoftGram-Service`. |
| `MSGRAM_PASSWORD` | **Sim** | — | Senha do usuário cadastrado no `MeasureSoftGram-Service`. |
| `MCP_TRANSPORT` | Não | `streamable-http` | Protocolo de transporte do FastMCP: `streamable-http` (HTTP na porta `8000`), `sse` ou `stdio`. |

---

## Como Executar e Vincular à sua IA (Imagem Oficial DockerHub)

A imagem oficial é publicada automaticamente no DockerHub (`measuresoftgram/ai`) a cada release na branch `main`, disponibilizando as seguintes tags:
- `measuresoftgram/ai:latest` — última versão estável publicada
- `measuresoftgram/ai:<major>.<minor>.<patch>` (ex.: `0.1.0`) — versão exata imutável
- `measuresoftgram/ai:<major>.<minor>` (ex.: `0.1`) — última correção da série *minor*
- `measuresoftgram/ai:sha-<7-chars>` — imagem rastreável ao commit Git exato

Você pode conectar seu agente de IA ao contêiner de **duas formas**:

### Opção 1 — Contêiner HTTP (`streamable-http` na porta `8000`)

1. Suba o contêiner oficial mapeando a porta `8000` e informando a URL do serviço e suas credenciais:

```bash
docker run -d \
  --name msgram-mcp \
  -p 8000:8000 \
  -e SERVICE="https://msgram.lappis.rocks/api/v1/" \
  -e MSGRAM_USER="seu_usuario" \
  -e MSGRAM_PASSWORD="sua_senha" \
  -e MCP_TRANSPORT="streamable-http" \
  measuresoftgram/ai:latest
```

*(Alternativamente, passe um arquivo `.mcp.env` local com `--env-file ./env-vars/.mcp.env`).*

2. Adicione a configuração abaixo no arquivo de servidores MCP do seu agente/IDE:

```json
{
  "mcpServers": {
    "measuresoftgram": {
      "type": "streamable-http",
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

### Opção 2 — Execução Direta sob Demanda via `stdio` (`docker run -i --rm`)

Caso prefira que o próprio cliente MCP (Claude Desktop, Cursor, VS Code, etc.) inicie e encerre o contêiner automaticamente via entrada/saída padrão (`stdio`) sem expor portas na máquina:

```json
{
  "mcpServers": {
    "measuresoftgram": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e", "SERVICE=https://msgram.lappis.rocks/api/v1/",
        "-e", "MSGRAM_USER=seu_usuario",
        "-e", "MSGRAM_PASSWORD=sua_senha",
        "-e", "MCP_TRANSPORT=stdio",
        "measuresoftgram/ai:latest"
      ]
    }
  }
}
```

> **Dica para `MeasureSoftGram-Service` rodando no host local (`localhost:8080`):**  
> Ao usar a Opção 2 apontando para um backend local fora da mesma rede compose, adicione `"--add-host=host.docker.internal:host-gateway"` nos `args` do Docker e defina `SERVICE=http://host.docker.internal:8080/api/v1/`.

---

## Estrutura de pastas

```txt
src/msgram_mcp/
├── __init__.py
├── server.py                      # ponto de entrada — sobe o FastMCP e registra as tools
├── client.py                      # cliente HTTP compartilhado entre as tools
├── auth/
│   └── msgram_auth.py             # autenticação com o msgram-service
└── tools/
    ├── __init__.py
    ├── balance_matrix.py
    ├── entity_relationship_tree.py
    ├── goals.py
    ├── historical_values.py
    ├── latest_values.py
    ├── organizations.py
    ├── releases.py
    ├── repositories.py
    ├── supported_characteristics.py
    ├── supported_measures.py
    └── supported_metrics.py

tests/msgram_mcp/
├── test_client.py
├── test_server.py
├── auth/
│   └── test_msgram_auth.py
└── tools/
    ├── test_balance_matrix.py
    ├── test_entity_relationship_tree.py
    ├── test_goals.py
    ├── test_historical_values.py
    ├── test_latest_values.py
    ├── test_organizations.py
    ├── test_releases.py
    ├── test_repositories.py
    ├── test_supported_characteristics.py
    ├── test_supported_measures.py
    └── test_supported_metrics.py
```

---

## Desenvolvimento Local (`compose.dev.yaml`)

### Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

### Configuração do ambiente

Copie os arquivos de variáveis de ambiente de exemplo:

```bash
mkdir -p env-vars
cp env-vars-example/.service.env env-vars/.service.env
cp env-vars-example/.mcp.env env-vars/.mcp.env
```

Preencha o `env-vars/.mcp.env` com as credenciais locais:

```env
SERVICE=http://service:8080/api/v1/
MSGRAM_USER=admin
MSGRAM_PASSWORD=admin
MCP_TRANSPORT=streamable-http
```

### Subindo a stack de desenvolvimento

```bash
docker compose -f compose.dev.yaml up --build
```

Os serviços disponíveis após subir:

| Serviço        | URL                   | Descrição |
|----------------|-----------------------|-----------|
| `msgram-service` | http://localhost:8080 | API Backend do MeasureSoftGram |
| `mcp-server`     | http://localhost:8000 | Servidor MCP com *hot-reload* (`watchfiles`) |
| `mcp-inspector`  | http://localhost:6274 | Interface visual do MCP Inspector para testar as *tools* |

### Rodando os testes

```bash
docker compose -f compose.dev.yaml exec mcp-server pytest -v
```