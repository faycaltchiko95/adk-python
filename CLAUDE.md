# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Agent Development Kit (ADK)** - a Python framework for building, evaluating, and deploying AI agents. ADK is model-agnostic but optimized for Gemini and the Google ecosystem. The framework follows a code-first approach where agent logic, tools, and orchestration are defined directly in Python.

**Key Resources:**
- Official Docs: https://google.github.io/adk-docs/
- Samples: https://github.com/google/adk-samples
- Community: r/agentdevelopmentkit on Reddit

## Development Setup

### Installation

Install dependencies using `uv` (preferred) or `pip`:

```bash
# Using uv (recommended for development)
uv venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
uv sync --extra test --extra eval --extra dev

# Or using pip
pip install -e ".[dev,test,eval]"
```

### Environment Variables

Create a `.env` file in agent directories for configuration (see `tests/integration/.env.example` for reference). Common variables:
- `GOOGLE_API_KEY`: For Google AI API access
- `GOOGLE_CLOUD_PROJECT`: For Vertex AI backend
- `GOOGLE_CLOUD_REGION`: For Vertex AI backend

## Testing

### Run Unit Tests

```bash
# All unit tests (using pytest)
pytest tests/unittests

# Specific test file
pytest tests/unittests/agents/test_llm_agent.py

# With parallel execution
pytest tests/unittests -n auto

# Tests are configured in pyproject.toml under [tool.pytest.ini_options]
```

### Run Integration Tests

```bash
pytest tests/integration
```

### CI/CD

The project uses GitHub Actions for CI. See `.github/workflows/python-unit-tests.yml` for the CI configuration which tests against Python 3.9, 3.10, and 3.11.

## Code Quality

### Formatting

Uses `pyink` (Google's fork of Black) with Google style guide:

```bash
# Format code
pyink src/ tests/

# Configuration in pyproject.toml:
# - line-length: 80
# - pyink-indentation: 2 spaces
```

### Import Sorting

Uses `isort` with Google conventions:

```bash
# Sort imports
isort src/ tests/

# Configuration enforces:
# - Single import per line
# - Alphabetical sorting within sections
```

### Linting

Uses `pylint` following Google Python style guide:

```bash
# Run pylint
pylint src/google/adk

# Configuration in pylintrc (Google style guide based)
```

## CLI Commands

The `adk` CLI is the main entry point (defined in `src/google/adk/cli/cli_tools_click.py`):

### Create New Agent

```bash
adk create my_agent_name
adk create my_agent_name --model gemini-2.0-flash --api_key YOUR_KEY
```

### Run Agent Interactively

```bash
adk run path/to/agent_folder
adk run path/to/agent_folder --save_session  # Save session on exit
```

### Evaluate Agent

```bash
adk eval path/to/agent_folder path/to/eval_set.evalset.json
adk eval agent_folder eval_set.json --print_detailed_results
```

### Start Web Development UI

```bash
adk web path/to/agent_folder
adk web path/to/agent_folder --session_db_url postgresql://...
```

### Deploy Agent

```bash
adk deploy [target] path/to/agent_folder
```

## Architecture

### Core Components

1. **Agents** (`src/google/adk/agents/`)
   - `BaseAgent`: Abstract base for all agents
   - `LlmAgent` / `Agent`: Main agent class with LLM integration
   - `LoopAgent`: Agent that runs in a loop until exit condition
   - `ParallelAgent`: Executes multiple agents in parallel
   - `SequentialAgent`: Executes multiple agents sequentially
   - `LangGraphAgent`: Integration with LangGraph framework
   - `RemoteAgent`: For agent-to-agent communication

2. **Tools** (`src/google/adk/tools/`)
   - `BaseTool`: Abstract base for all tools
   - `FunctionTool`: Wraps Python functions as tools
   - `LongRunningFunctionTool`: For async/long operations
   - Built-in tools: `google_search`, `built_in_code_execution`, `load_memory`, `load_artifacts`, `exit_loop`, etc.
   - Tool integrations: APIHub, OpenAPI specs, Google APIs, MCP (Model Context Protocol), Application Integration

3. **Models** (`src/google/adk/models/`)
   - `BaseLlm`: Abstract LLM interface
   - `GoogleLlm`: Google AI/Gemini integration
   - `AnthropicLlm`: Anthropic Claude integration
   - `LiteLlm`: LiteLLM integration for multi-provider support
   - `LLMRegistry`: Model registry for managing LLM instances

4. **Sessions** (`src/google/adk/sessions/`)
   - `Session`: Conversation state management
   - `InMemorySessionService`: Development/testing
   - `DatabaseSessionService`: SQL database persistence
   - `VertexAiSessionService`: Vertex AI integration

5. **Memory** (`src/google/adk/memory/`)
   - `BaseMemoryService`: Abstract memory interface
   - `InMemoryMemoryService`: In-memory storage
   - `VertexAiRagMemoryService`: RAG-based memory with Vertex AI

6. **Artifacts** (`src/google/adk/artifacts/`)
   - `BaseArtifactService`: File/data storage abstraction
   - `InMemoryArtifactService`: Development/testing
   - `GcsArtifactService`: Google Cloud Storage integration

7. **Flows** (`src/google/adk/flows/`)
   - `BaseLlmFlow`: Agent execution flow control
   - `SingleFlow`: Single LLM call flow
   - `AutoFlow`: Automatic tool calling and multi-turn conversations

8. **Runners** (`src/google/adk/runners.py`)
   - `Runner`: Orchestrates agent execution with session/artifact/memory services

9. **Code Executors** (`src/google/adk/code_executors/`)
   - Execute code safely in isolated environments
   - `ContainerCodeExecutor`: Docker-based execution

10. **Evaluation** (`src/google/adk/evaluation/`)
    - Framework for evaluating agent performance
    - Uses Vertex AI evaluation metrics

### Multi-Agent Systems

ADK supports hierarchical multi-agent architectures through the `sub_agents` parameter:

```python
coordinator = LlmAgent(
    name="coordinator",
    model="gemini-2.0-flash",
    sub_agents=[greeter_agent, task_agent]
)
```

Agents can transfer control using the `transfer_to_agent` tool automatically provided to parent agents.

### Agent Callbacks

Agents support lifecycle callbacks:
- `before_agent_callback`: Before agent execution
- `after_agent_callback`: After agent execution
- `before_model_callback`: Before LLM call
- `after_model_callback`: After LLM response
- `before_tool_callback`: Before tool execution
- `after_tool_callback`: After tool returns

### Context Objects

- `InvocationContext`: Full agent invocation context (passed to agent)
- `CallbackContext`: Context for callbacks
- `ToolContext`: Context passed to tool functions
- `ReadonlyContext`: Read-only view for instruction providers

## Build and Distribution

### Building Package

Uses `flit` as the build backend:

```bash
# Build distribution
flit build

# Publish to PyPI (requires credentials)
flit publish
```

### Package Structure

- Package name: `google-adk`
- Module name: `google.adk`
- Entry point: `adk` CLI command
- Python support: 3.9, 3.10, 3.11, 3.12, 3.13

## Important Development Notes

1. **Agent Structure**: Agents are organized in folders with an `agent.py` file that exports a `root_agent` variable
2. **Session Files**: `.session.json` files store conversation history and state
3. **Input Files**: `.input.json` files contain state and queries for testing
4. **Eval Sets**: `.evalset.json` files define evaluation test cases
5. **Environment Loading**: The CLI automatically loads `.env` files from agent directories
6. **Streaming**: ADK supports streaming responses with `StreamingMode.ON` or `StreamingMode.AUTO`
7. **A2A Integration**: ADK integrates with the A2A (Agent-to-Agent) protocol for remote agent communication

## Testing Patterns

- Unit tests are in `tests/unittests/` mirroring the `src/google/adk/` structure
- Integration tests are in `tests/integration/`
- Test utilities and fixtures in `tests/unittests/utils.py` and `tests/unittests/conftest.py`
- Mock LLM responses for deterministic testing
- Use `pytest-asyncio` for async test support
- Use `pytest-mock` for mocking

## Dependencies Management

- Core dependencies in `[project.dependencies]`
- Development tools in `[project.optional-dependencies.dev]`
- Testing tools in `[project.optional-dependencies.test]`
- Evaluation tools in `[project.optional-dependencies.eval]`
- Optional extensions in `[project.optional-dependencies.extensions]`

## Contributing

- All PRs require an associated Issue (except small docs/typo fixes)
- Keep PRs small and focused (one concern per PR)
- Add tests for new functionality
- CLA required (see CONTRIBUTING.md)
- Follow Google's Open Source Community Guidelines
