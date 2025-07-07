# 🐞 Processo de Depuração e Resolução de Problemas - ADK MCP Tutorial

## 🎯 Visão Geral do Projeto e Desafio

**Objetivo:** Implementar e executar com sucesso um agente ADK (`Agent Development Kit`) que se conecta a servidores MCP (`Model Context Protocol`), tanto um local (`local_mcp`) quanto um remoto (`remote_mcp_agent` para o Notion), para demonstrar a capacidade de estender as funcionalidades de um LLM com ferramentas externas.

**Stack Tecnológica do Desafio:**

| Tecnologia | **PORQUÊ** Utilizada no Projeto | **COMO** Implementada no Projeto |
| :--- | :--- | :--- |
| **Google ADK** | • Framework principal para construir o agente de IA. | `LlmAgent` para definir o agente; `MCPToolset` para integrar ferramentas externas. |
| **MCP** | • Padrão aberto para desacoplar o agente das ferramentas. | `StdioServerParameters` para iniciar servidores MCP como subprocessos (`python`, `npx`, `docker`). |
| **Python** | • Linguagem base para o ADK e o servidor MCP local. | Scripts de agente (`agent.py`) e servidor (`server.py`). |
| **Node.js / npx** | • Dependência de ambiente para executar servidores MCP da comunidade (ex: Notion). | Comando `npx` especificado no `MCPToolset` para iniciar o `@notionhq/notion-mcp-server`. |
| **Docker** | • Alternativa para executar servidores MCP em contêineres isolados. | Comando `docker run` especificado no `MCPToolset` como uma alternativa ao `npx`. |
| **FastAPI / Uvicorn**| • Servidor web para a interface de depuração `adk web`. | Fornece a UI (`http://localhost:8000`) para interagir com o agente durante o desenvolvimento. |

---

## 🏗️ Arquitetura de Interação Agente-Servidor

O desafio central deste projeto reside na correta orquestração entre o agente ADK e os servidores de ferramentas MCP. A arquitetura é baseada em comunicação interprocessos via `stdio` (entrada/saída padrão).

**Fluxo de Execução:**

`ADK Web (Uvicorn)` → `LlmAgent` → `MCPToolset.get_tools()` → `asyncio.create_subprocess_exec()` → `Subprocesso (Servidor MCP)`

**Decisão Arquitetônica:** Utilizar `StdioServerParameters` dentro do `MCPToolset` para que o agente ADK gerencie o ciclo de vida do servidor de ferramentas.

### **PORQUÊ esta abordagem?**

1.  **Autocontenção:** O agente é responsável por iniciar suas próprias dependências de ferramentas, simplificando a execução para o usuário final.
2.  **Desacoplamento:** O agente não precisa conhecer a implementação interna da ferramenta, apenas o protocolo MCP.
3.  **Flexibilidade:** Permite conectar-se a qualquer servidor MCP que possa ser iniciado por um comando de linha (`python`, `npx`, `docker`, etc.).

### **COMO foi implementada?**

-   No `local_mcp/agent.py` e `remote_mcp_agent/agent.py`, a classe `MCPToolset` é instanciada com `StdioServerParameters`, especificando o `command` e `args` necessários para iniciar o respectivo servidor MCP. O ADK então lida com a criação do subprocesso e o estabelecimento da comunicação.

---

## 🚀 A Jornada de Depuração: Do Erro à Solução

Esta seção detalha os problemas críticos encontrados durante a configuração e execução do `remote_mcp_agent` e o processo de raciocínio para chegar à solução final.

| # | **PROBLEMA** (O que deu errado) | **ANÁLISE (PORQUÊ)** Aconteceu | **SOLUÇÃO (COMO)** Foi Resolvido | **RESULTADO & APRENDIZADO** |
|---|---------------------------------|---------------------------------|-----------------------------------|-----------------------------|
| 1 | **`NotImplementedError` no Windows:** O `adk web` falhava ao tentar criar um subprocesso para o servidor MCP. | A análise do traceback e a consulta à documentação do ADK (`adk-docs-mcp_tool.txt`) revelaram que o *event loop* padrão do `asyncio` no Windows é incompatível com a criação de subprocessos quando a funcionalidade de *hot-reload* do Uvicorn está ativa. | A solução documentada foi aplicada: o servidor foi iniciado com a flag `--no-reload` para usar um modo de execução compatível. **Comando:** `adk web --no-reload`. | Erro resolvido, o ADK passou a tentar iniciar o subprocesso. **Aprendizado:** A importância de compreender as particularidades da plataforma (Windows vs. Linux) e como as ferramentas de desenvolvimento (`hot-reload`) interagem com a pilha de tecnologia subjacente (`asyncio`). |
| 2 | **`FileNotFoundError` para `npx`:** Após corrigir o primeiro erro, o sistema reportou que o comando `npx` não foi encontrado. | O traceback indicava claramente que o sistema operacional não conseguiu localizar o executável `npx.exe`. Isso significava que o Node.js (que fornece o `npx`) não estava instalado ou seu diretório de instalação não estava na variável de ambiente `PATH` do sistema. A documentação do projeto (`readme.md`) confirmava a necessidade do Node.js. | O **Node.js (versão LTS)** foi instalado a partir do site oficial, garantindo que a opção "Adicionar ao PATH" estivesse marcada. O terminal foi reiniciado para que as novas variáveis de ambiente fossem carregadas. | O comando `npx` passou a ser encontrado pelo sistema. **Aprendizado:** A gestão de dependências de ambiente é tão crítica quanto a de bibliotecas de código. Um erro que parece ser de "código" pode, na verdade, ser um problema de configuração de infraestrutura local. Mas decidi manter como docker. |
| 3 | **`ValidationError` do Pydantic:** Ao tentar uma abordagem alternativa com Docker, e crucialmente, ao perceber que a raiz do problema era outra, este erro persistia. Ele ocorria quando o ADK tentava processar o esquema da ferramenta recebido do servidor MCP. | A análise profunda do traceback foi a chave. Ele mostrou que o `MCPTool` estava, por engano, chamando uma função de conversão de esquema do `OpenAPITool` (`.../openapi_tool/.../rest_api_tool.py`). Isso indicava um **bug interno na versão da biblioteca `google-adk==1.0.0`**. A hipótese tornou-se que a versão da dependência estava desatualizada e continha um bug já corrigido. | A solução definitiva foi atacar a causa raiz: a versão da biblioteca. O arquivo `requirements.txt` foi atualizado de `google-adk==1.0.0` para `google-adk==1.5.0`. As dependências foram reinstaladas com `pip install -r requirements.txt`. | **Todos os erros foram resolvidos.** O agente `remote_mcp_agent` inicializou e funcionou corretamente. **Aprendizado Principal:** Quando uma biblioteca se comporta de maneira inesperada e a análise do traceback aponta para uma falha interna, uma das soluções mais eficientes é verificar se existem versões mais recentes que corrigem o problema. Isso demonstra uma abordagem de engenharia madura, evitando a depuração de código que não controlamos e reforçando a importância da manutenção proativa de dependências. |
| 4 | **`ProcessLookupError` no `local_mcp`:** O agente falhava ao iniciar, com um erro de `ProcessLookupError` e um aviso sobre `StdioServerParameters` ser obsoleto. | O aviso da biblioteca foi a pista inicial, mas a verdadeira análise veio da decodificação da nova API. A estrutura antiga (`StdioServerParameters`) misturava responsabilidades. A nova estrutura (`StdioConnectionParams` envolvendo `StdioServerParameters`) reflete um **princípio de design de separação de responsabilidades**: `StdioServerParameters` agora define **O QUÊ** rodar (o comando), enquanto `StdioConnectionParams` define **COMO** se conectar (via `stdio`). Essa mudança, embora mais verbosa, torna a arquitetura mais extensível para futuros métodos de conexão (ex: TCP) sem alterar o `MCPToolset`. | A solução foi um processo iterativo de "diálogo com a API" usando o Pylance como guia. Cada erro do linter não foi apenas uma correção, mas uma pista para descobrir a nova arquitetura aninhada. O código foi reestruturado para `tools=[MCPToolset(connection_params=StdioConnectionParams(server_params=...))]`, alinhando-se com a nova estrutura de API e o tipo de dado esperado (`list`) pelo `LlmAgent`. | O erro foi resolvido. **Aprendizado de Engenharia:** Uma API que se torna mais verbosa geralmente sinaliza uma arquitetura mais madura e desacoplada. O aprendizado não é apenas "seguir o linter", mas **"decodificar a intenção arquitetônica por trás das mudanças de API"**. Compreender o "porquê" da evolução de uma biblioteca é uma marca de engenharia sênior, permitindo não apenas usar a ferramenta, mas antecipar sua trajetória e projetar soluções mais robustas. |

---

## ⚙️ **Configurações Críticas e Checklist Final**

Esta seção resume as configurações essenciais para garantir que o ambiente de desenvolvimento funcione corretamente após o processo de depuração.

### **🔧 Dependências de Ambiente**

| **Dependência** | **Versão/Estado Necessário** | **Comando de Verificação** | **Por que é Crítico** |
|---|---|---|---|
| **Python** | `3.8+` | `python --version` | Linguagem base do ADK e do projeto. |
| **Node.js** | `LTS` (Instalado e no `PATH`) | `npx -v` | Necessário para executar servidores MCP baseados em Node.js, como o do Notion. |
| **Docker** | `Desktop` (Rodando) | `docker --version` | Necessário se for usar a abordagem de contêiner para os servidores MCP. |

### **📦 Dependências de Projeto (`requirements.txt`)**

| **Biblioteca** | **Versão Crítica** | **Por que é Crítico** |
|---|---|---|
| `google-adk` | `==1.5.0` | A versão `1.0.0` continha um bug de conversão de esquema que impedia o funcionamento do `MCPToolset`. A atualização é **obrigatória**. |
| `mcp` | `==1.9.1` | Versão compatível com a do ADK. |

### **📋 Checklist de Verificação Rápida**

- [x] Node.js instalado e `npx` acessível no `PATH`.
- [x] Docker instalado e rodando (se necessário).
- [x] `requirements.txt` atualizado para `google-adk==1.5.0`.
- [x] Dependências Python instaladas (`pip install -r requirements.txt`).
- [x] Variável de ambiente `NOTION_API_KEY` definida no arquivo `.env`.
- [x] `adk web` é executado com a flag `--no-reload` no Windows.

### **🚀 Comando de Inicialização Final (Windows)**

```powershell
# No terminal, na raiz do projeto
# Garanta que o ambiente virtual (.venv) esteja ativo

# Inicia o servidor de desenvolvimento do ADK de forma compatível com o Windows
adk web --no-reload
```
