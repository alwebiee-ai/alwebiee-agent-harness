# Alwebiee Agent Harness

A minimal, dependency-light harness for running an LLM agent with tool calling, persistent memory, file access, and Model Context Protocol (MCP) support — all in ~270 lines of readable Python.

No frameworks, no abstractions. Just an async chat loop, a dictionary of tools, and an OpenAI-compatible API client.

## Features

- **Async agent loop** — standard OpenAI-style tool-calling loop (`agent.py`), running on `asyncio`.
- **Multi-provider LLM support** — OpenRouter, Ollama, and OpenAI via a single `AsyncOpenAI` client factory.
- **Local tools** — read/write files in a sandboxed `workspace/` directory, plus list files.
- **Persistent memory** — the agent remembers facts about you across sessions in a plain `MEMORY.md` file, appended to automatically via the `save_memory` tool.
- **MCP integration** — launches stdio MCP servers (shipped with `time` and `fetch`) and merges their tools into the agent's toolset.
- **Slash commands** — inspect models, tools, and memory from the REPL without leaving the chat.

## Architecture

```
You ──> REPL (agent.py)
          │
          ├── command.py      slash commands (/help, /tools, /memory, ...)
          │
          └── run_agent()
                │
                ├── config.py       provider, model, system prompt
                ├── utils.py        AsyncOpenAI client factory
                ├── tools.py        local tools + JSON schemas
                │     └── memory.py MEMORY.md persistence
                └── mcp_servers.py  stdio MCP servers (time, fetch)
```

A typical turn:

1. `agent.py` builds the message list, injecting the contents of `MEMORY.md` into the system prompt.
2. It calls the LLM with the full tool schema list.
3. If the model requests tool calls, they are executed and the results are appended back as `tool` messages — looping until the model returns a final answer.
4. Local tools run directly; MCP tools are dispatched over the MCP protocol.

## Project structure

| File | Purpose |
| --- | --- |
| `agent.py` | Entry point: async REPL + the tool-calling agent loop |
| `config.py` | Provider/model selection and the agent system prompt |
| `utils.py` | `configure_llm_client()` — builds the `AsyncOpenAI` client per provider |
| `tools.py` | Local tool functions and their OpenAI-format JSON schemas |
| `memory.py` | Load/append the agent's `MEMORY.md` memory file |
| `mcp_servers.py` | Launch MCP servers, merge their tools, dispatch calls |
| `command.py` | Slash-command handling for the REPL |
| `workspace/` | Sandboxed directory the agent's file tools operate in |
| `MEMORY.md` | Persistent agent memory (append-only Markdown) |

## Requirements

- Python **>= 3.14**
- [uv](https://docs.astral.sh/uv/) (recommended) or `pip`
- `uvx` on your PATH if you use the bundled MCP servers

Dependencies: `mcp`, `openai`, `pydantic`, `python-dotenv`.

## Installation

```bash
git clone <this-repo>
cd alwebiee-agent-harness

uv sync            # or: pip install -e .
```

## Configuration

Create a `.env` file in the project root (see `env-example`):

```bash
cp env-example .env
```

```ini
OPENAI_API_KEY=xxxxxxxxxx
OPENROUTER_API_KEY=xxxxxxxxxx
```

Then pick your provider and model in `config.py`:

```python
LLM_PROVIDER = "openrouter"                              # "openrouter" | "ollama" | "openai"
LLM_MODEL = "nvidia/nemotron-3-super-120b-a12b:free"     # any model your provider serves
```

| Provider | Base URL | API key |
| --- | --- | --- |
| `openrouter` | `https://openrouter.ai/api/v1` | `OPENROUTER_API_KEY` |
| `ollama` | `http://localhost:11434/v1` | none (placeholder value) |
| `openai` | OpenAI default | `OPENAI_API_KEY` |

For Ollama, make sure the server is running (`ollama serve`) and the model is pulled.

## Usage

```bash
uv run agent.py
```

```
 - [mcp] connected 'time' with 1 tools ['get_current_time']
 - [mcp] connected 'fetch' with 1 tools ['fetch']
Agent Harness (using nvidia/nemotron-3-super-120b-a12b:free with 7 tools) - CTRL+C to exit

You: What files are in my workspace?
 - [tool] list_files({})
...
Agent: You have `hello.txt` and `research.md` in your workspace...
```

### Slash commands

| Command | Description |
| --- | --- |
| `/help` | List available commands |
| `/models` | Show available models |
| `/tools` | List the agent's tools and descriptions |
| `/memory` | Print the agent's current memory |
| `/exit` | Quit |

## Built-in tools

| Tool | Description |
| --- | --- |
| `list_files` | List files in `workspace/` |
| `read_file` | Read a file from `workspace/` |
| `write_file` | Create or overwrite a file in `workspace/` |
| `load_memory` | Read the agent's `MEMORY.md` |
| `save_memory` | Append a fact to `MEMORY.md` |

MCP servers add their own tools on top of these at startup.

## Extending

**Add a local tool** — write the function and register it in both `TOOLS` and `TOOL_SCHEMAS` in `tools.py`:

```python
def greet(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"

TOOLS["greet"] = greet
TOOL_SCHEMAS.append({
    "type": "function",
    "function": {
        "name": "greet",
        "description": "Greet someone by name.",
        "parameters": {
            "type": "object",
            "properties": {"name": {"type": "string", "description": "Who to greet."}},
            "required": ["name"],
        },
    },
})
```

**Add an MCP server** — add an entry to `MCP_SERVERS` in `mcp_servers.py`:

```python
MCP_SERVERS["github"] = StdioServerParameters(
    command="uvx",
    args=["mcp-server-github"],
    env={"GITHUB_TOKEN": os.getenv("GITHUB_TOKEN")},
)
```

Its tools are discovered and merged automatically at startup.

**Add an LLM provider** — add a branch to `configure_llm_client()` in `utils.py` returning an `AsyncOpenAI` instance with the right `base_url`/`api_key`.

## Known limitations

- The agent loop dispatches tool calls through the local `TOOLS` dict, so MCP-provided tools (e.g. `get_current_time`, `fetch`) are advertised to the model but not yet routed through `mcp_servers.call_tool()` — calling them raises a `KeyError`. Wiring `call_tool()` into the loop is the obvious next step.
- `save_memory` is append-only; `MEMORY.md` grows forever and is never summarized or pruned.
- The system prompt, model, and provider are edited in source rather than set at runtime.
- No retries, token limits, or context-window truncation.
