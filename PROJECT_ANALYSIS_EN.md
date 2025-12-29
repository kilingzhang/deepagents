# DeepAgents Project Deep Dive Analysis

## Project Overview

**DeepAgents** is an open-source AI agent framework built on LangGraph, focusing on solving cost and reliability challenges for long-horizon tasks. The project implements core design patterns of modern AI agent systems, including task planning, computer access capabilities, and sub-agent delegation mechanisms.

### Core Positioning
- **Type**: AI Agent Development Framework
- **License**: MIT
- **Language**: Python (>=3.11)
- **Main Dependencies**: LangChain, LangGraph, Anthropic Claude

### Origin and Design Philosophy
The project draws inspiration from successful commercial agent systems like [Claude Code](https://code.claude.com/docs) and [Manus](https://www.youtube.com/watch?v=6_BcCthVvb8), making these design principles open source. According to [METR research](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/), average agent task length doubles every 7 months, driving the need for better long-horizon task management mechanisms.

---

## Project Architecture

### 1. Repository Structure

```
deepagents/
├── README.md                    # Main project documentation
├── LICENSE                      # MIT License
├── .gitignore                  # Git ignore configuration
└── libs/                       # Core libraries
    ├── deepagents/             # Core agent framework
    ├── deepagents-cli/         # Command-line interface
    ├── acp/                    # Agent Client Protocol integration
    └── harbor/                 # Evaluation and benchmarking
```

### 2. Four Core Modules

#### 2.1 deepagents - Core Framework
**Version**: 0.3.1  
**Description**: General-purpose deep agent framework with sub-agent spawning, todo list management, and mock filesystem

**Main Components**:
```python
deepagents/
├── graph.py                    # Agent graph creation and configuration
├── backends/                   # Backend storage systems
│   ├── protocol.py            # Backend protocol definitions
│   ├── state.py               # State-based ephemeral storage
│   ├── filesystem.py          # Real filesystem backend
│   ├── store.py               # Persistent storage backend
│   ├── composite.py           # Composite backend (route paths)
│   └── sandbox.py             # Sandbox execution backend
└── middleware/                # Middleware system
    ├── filesystem.py          # Filesystem tools middleware
    ├── subagents.py          # Sub-agent delegation middleware
    └── patch_tool_calls.py   # Tool call patching middleware
```

**Core Features**:
- **Flexible Backend System**: Supports ephemeral state, local filesystem, persistent storage, and sandbox environments
- **Middleware Architecture**: Extend agent capabilities through composable middleware
- **Sub-agent Support**: Create specialized sub-agents for independent tasks
- **Model Support**: Defaults to Claude Sonnet 4.5, but supports any LangChain-compatible model

#### 2.2 deepagents-cli - Command Line Interface
**Version**: 0.0.12  
**Description**: Interactive terminal agent, similar to a command-line version of Claude Code

**Core Features**:
- **Built-in Tools**: File operations, shell commands, web search, sub-agent delegation
- **Skills System**: Extensible domain-specific capabilities (progressive disclosure pattern)
- **Persistent Memory**: Maintains user preferences, coding style, and project context across sessions
- **Project-Aware**: Automatically detects project roots and loads project-specific configurations
- **Sandbox Support**: Supports remote sandboxes like Modal, Runloop, Daytona

**Built-in Tools**:
| Tool | Description |
|------|-------------|
| `ls` | List directory contents |
| `read_file` | Read file contents |
| `write_file` | Create or overwrite files |
| `edit_file` | Precise string replacement edits |
| `glob` | File pattern matching |
| `grep` | Text search |
| `shell/execute` | Execute shell commands |
| `web_search` | Web search (Tavily API) |
| `fetch_url` | Fetch webpage as Markdown |
| `task` | Delegate to sub-agents |
| `write_todos` | Create task lists |

**Skills System Design**:
```
~/.deepagents/<agent_name>/
├── agent.md              # Global personality/style
└── skills/               # Skills directory
    ├── web-research/
    │   └── SKILL.md
    └── langgraph-docs/
        └── SKILL.md

project-root/
└── .deepagents/
    ├── agent.md          # Project-specific instructions
    └── skills/           # Project-specific skills
```

Skills use **progressive disclosure pattern** (Anthropic's best practice):
1. Scan skills directory at startup
2. Extract YAML metadata (name + description)
3. Inject skills list into system prompt
4. Only read full SKILL.md content when task matches
5. Execute workflow defined in skill

#### 2.3 acp - Agent Client Protocol
**Version**: 0.0.1  
**Status**: Work in Progress  
**Description**: Agent Client Protocol integration support

**Requirements**:
- Python >= 3.14
- Dependencies: agent-client-protocol >= 0.6.2

#### 2.4 harbor - Evaluation Integration
**Version**: 0.0.1  
**Description**: Evaluate agent performance on Terminal Bench 2.0 using Harbor framework

**Main Features**:
- **Sandbox Environments**: Supports Docker, Modal, Daytona, E2B, etc.
- **Automatic Test Execution**: Run and verify agent behavior
- **Reward Scoring**: 0.0-1.0 scoring based on test pass rate
- **Trajectory Logging**: ATIF (Agent Trajectory Interchange Format)
- **LangSmith Integration**: Complete tracing and observability

**Terminal Bench 2.0**:
- 90+ tasks across multiple domains
- Test domains: Software engineering, biology, security, gaming, etc.
- Example tasks:
  - `path-tracing`: Reverse-engineer C program from rendered image
  - `chess-best-move`: Find optimal move using chess engine
  - `git-multibranch`: Complex git operations with merge conflicts
  - `sqlite-with-gcov`: Build SQLite with code coverage

---

## Core Design Patterns

### 1. Middleware Architecture

DeepAgents uses **middleware pattern** as core extension mechanism. Each middleware handles specific functionality and can be composed:

**Default Middleware Stack** (in execution order):
1. **TodoListMiddleware** - Task planning and progress tracking
2. **FilesystemMiddleware** - File operations and context offloading
3. **SubAgentMiddleware** - Sub-agent delegation
4. **SummarizationMiddleware** - Context summarization (170k token trigger)
5. **AnthropicPromptCachingMiddleware** - Prompt caching (cost reduction)
6. **PatchToolCallsMiddleware** - Fix dangling tool calls after interrupts
7. **HumanInTheLoopMiddleware** - Human approval (optional)

**Middleware Interface**:
```python
class AgentMiddleware:
    tools: list[BaseTool] = []  # Injected tools
    
    async def on_model_request(
        self, 
        request: ModelRequest, 
        runtime: ToolRuntime
    ) -> ModelRequest | Command:
        """Pre-request processing"""
        
    async def on_model_response(
        self, 
        response: ModelResponse, 
        runtime: ToolRuntime
    ) -> ModelResponse | Command:
        """Post-response processing"""
```

### 2. Backend System

**Backend Protocol Hierarchy**:
```python
BackendProtocol                  # Base file operations
└── SandboxBackendProtocol      # Extended: Shell execution capability
```

**Available Backend Implementations**:

| Backend Type | Storage | Persistence | Shell Execution | Use Case |
|-------------|---------|-------------|-----------------|----------|
| **StateBackend** | Agent state | Ephemeral | ❌ | Default, lightweight tasks |
| **FilesystemBackend** | Local disk | Persistent | ✅ | Local development |
| **StoreBackend** | LangGraph Store | Cross-session | ❌ | Long-term memory |
| **CompositeBackend** | Routed hybrid | Mixed | Depends on route | Hybrid strategy |

**CompositeBackend Example** (Hybrid Memory):
```python
backend = CompositeBackend(
    default=StateBackend(),
    routes={
        "/memories/": StoreBackend(store=InMemoryStore()),
    }
)
```
- Files under `/memories/` persist across all sessions
- Other paths remain ephemeral
- Implements **working files ephemeral + knowledge base persistent** hybrid strategy

### 3. Sub-agent Delegation

**Core Concept**: Delegate complex tasks to specialized sub-agents with isolated contexts via `task` tool

**Delegation Workflow**:
```
Main Agent → task(prompt, subagent_type) → Sub-agent (isolated context)
                                            ↓
                                       Execute task
                                            ↓
Main Agent ← Result message ←──────────────┘
```

**Advantages**:
- **Context Isolation**: Each sub-agent has independent context window
- **Parallel Execution**: Can spawn multiple sub-agents for independent subtasks
- **Specialization**: Different sub-agents can have different tools, models, prompts
- **Token Optimization**: Avoid main agent context bloat

**Sub-agent Types**:
1. **General-purpose sub-agent**: Same capabilities as main agent, for isolating complex tasks
2. **Specialized sub-agent**: Custom tools/prompts/models for specific domains

**Usage Example**:
```python
# Define research sub-agent
research_subagent = {
    "name": "research-agent",
    "description": "Used for in-depth research questions",
    "system_prompt": "You are an expert researcher",
    "tools": [internet_search],
    "model": "openai:gpt-4o",
}

agent = create_deep_agent(subagents=[research_subagent])
```

### 4. Task Planning (TodoList)

**Design Principle**: Plan before executing for complex multi-step tasks

**Tools**:
- `write_todos(items: list[TodoItem])` - Create/update task list
- `read_todos()` - Read current task state

**Task Structure**:
```python
class TodoItem(TypedDict):
    task: str           # Task description
    completed: bool     # Completion status
```

**Usage Guidelines** (auto-injected into system prompt):
- ✅ Complex multi-step tasks should use todos
- ✅ Mark tasks as completed after finishing
- ❌ Simple tasks don't need todos
- ❌ Don't over-granularize tasks

### 5. Context Management

**Automatic Context Offloading**:
When tool output is too large, FilesystemMiddleware automatically saves it as a file:

```
Tool Execution → Large Output → Detect threshold → Write to /tmp/tool_output_xxx.txt
                                                    ↓
                Return: "Result saved to /tmp/tool_output_xxx.txt"
```

**Automatic Summarization**:
When context exceeds 170k tokens (or 85% of model capacity), SummarizationMiddleware triggers:
- Summarize old messages
- Keep recent 6 messages (or 10% tokens)
- Use same model for summarization

### 6. Progressive Disclosure

**Problem**: Giving agents too many tools/docs causes context confusion

**Solution**: Skills system uses Anthropic's progressive disclosure pattern

**Implementation**:
1. **Initial State**: Only show skill name and brief description
   ```
   Available Skills:
   - web-research: For web research tasks
   - langgraph-docs: LangGraph documentation lookup
   ```

2. **On-demand Loading**: When task matches, agent uses `read_file` to load full SKILL.md

3. **Execution**: Follow detailed step-by-step instructions in SKILL.md

**Advantages**:
- Reduce initial token consumption
- Lower agent confusion risk
- Support large skill libraries

---

## Technical Implementation Details

### 1. Filesystem Tools

**Path Validation**:
```python
def _validate_path(path: str) -> str:
    """Validate and normalize path to prevent directory traversal attacks"""
    # All paths must start with /
    # Normalize to forward slashes
    # Prevent ../ path traversal
```

**Core Tool Implementations**:

#### ls - List Directory
```python
def ls(path: str = "/") -> str:
    """List directory contents, show type (file/directory)"""
```

#### read_file - Read File
```python
def read_file(
    path: str,
    offset: int = 0,    # Starting line number
    limit: int = 500    # Number of lines to read
) -> str:
    """Paginated file reading with automatic line numbers"""
```

#### write_file - Write File
```python
def write_file(path: str, content: str) -> str:
    """Create or overwrite file"""
```

#### edit_file - Edit File
```python
def edit_file(path: str, old_str: str, new_str: str) -> str:
    """Precise string replacement
    
    Requirements:
    - old_str must uniquely match in file
    - Preserve leading/trailing whitespace
    - Reject if not unique
    """
```

#### glob - Pattern Matching
```python
def glob(pattern: str, root: str = "/") -> str:
    """Find files matching pattern
    
    Examples: **/*.py, src/**/*.ts
    """
```

#### grep - Text Search
```python
def grep(
    pattern: str,
    path: str = "/",
    case_insensitive: bool = False
) -> str:
    """Search text pattern, return matching files and line numbers"""
```

#### execute - Execute Command
```python
def execute(command: str, timeout: int = 60) -> str:
    """Execute shell command in sandbox
    
    Requirement: backend implements SandboxBackendProtocol
    """
```

### 2. Security Considerations

**Trust Model**: "Trust the LLM" (similar to Claude Code)

**Security Boundaries**:
- ❌ **Don't rely on** LLM self-restraint
- ✅ **Enforce** security boundaries at tool/sandbox level
- ✅ **Path Validation**: Prevent directory traversal attacks
- ✅ **Sandbox Isolation**: execute tool runs in isolated environment
- ✅ **Human Approval**: Sensitive operations require user approval (HITL)

**HITL (Human-in-the-Loop) Configuration**:
```python
agent = create_deep_agent(
    interrupt_on={
        "write_file": {"allowed_decisions": ["approve", "edit", "reject"]},
        "execute": {"allowed_decisions": ["approve", "edit", "reject"]},
    }
)
```

### 3. Model Configuration

**Default Model**: Claude Sonnet 4.5 (claude-sonnet-4-5-20250929)
```python
def get_default_model() -> ChatAnthropic:
    return ChatAnthropic(
        model_name="claude-sonnet-4-5-20250929",
        max_tokens=20000,
    )
```

**Supported Model Providers**:
- **Anthropic**: Claude 3/4 series
- **OpenAI**: GPT-4o, GPT-5-mini, O1/O3 series
- **Google**: Gemini 1.5/3 series

**Model Configuration Methods**:
```python
# Method 1: Pass model name
agent = create_deep_agent(model="openai:gpt-4o")

# Method 2: Pass model instance
from langchain.chat_models import init_chat_model
model = init_chat_model("openai:gpt-4o")
agent = create_deep_agent(model=model)
```

### 4. Prompt Engineering

**System Prompt Composition**:
```
[Base Prompt] + [Middleware-injected Tool Instructions] + [Custom System Prompt]
```

**Base Prompt**:
```
"In order to complete the objective that the user asks of you, 
 you have access to a number of standard tools."
```

**Middleware Auto-injected Content**:
1. **TodoListMiddleware**:
   - When to use todos
   - How to mark completed
   - Best practices
   - When not to use todos

2. **FilesystemMiddleware**:
   - All filesystem tools list
   - Paths must start with /
   - Each tool's purpose and parameters
   - Context offloading mechanism

3. **SubAgentMiddleware**:
   - Available sub-agent types and their tools
   - When to use/not use sub-agents
   - Parallel execution guidance
   - Sub-agent lifecycle

**Custom Prompt Best Practices**:
- ✅ Define domain-specific workflows
- ✅ Provide concrete examples
- ✅ Add specialized guidance
- ✅ Define stopping criteria and resource limits
- ❌ Don't re-explain standard tools (already covered by middleware)
- ❌ Don't contradict default instructions

---

## Performance Optimization

### 1. Prompt Caching (Anthropic)

**AnthropicPromptCachingMiddleware**:
- Automatically caches system prompts
- Significantly reduces cost for repeated calls
- Silently fails for non-Anthropic models

**Effect**:
- System prompts are typically long (contains all tool instructions)
- After caching, subsequent calls cost 90%+ less

### 2. Context Summarization

**SummarizationMiddleware Trigger Conditions**:

**For models with profile support**:
```python
trigger = ("fraction", 0.85)  # 85% of capacity
keep = ("fraction", 0.10)      # Keep 10%
```

**For other models**:
```python
trigger = ("tokens", 170000)   # 170k tokens
keep = ("messages", 6)         # Keep last 6 messages
```

**Summarization Strategy**:
- Use same model for summarization
- Keep recent interaction history
- Summarize old conversation history

### 3. Tool Call Batching

**Sub-agent Parallelization**:
```python
# Agent can call multiple sub-agents in parallel in one message
task("Research LeBron James", subagent_type="research")
task("Research Michael Jordan", subagent_type="research")
task("Research Kobe Bryant", subagent_type="research")
```

### 4. Filesystem Optimization

**Context Offloading**:
- Large tool outputs automatically saved as files
- Avoid context window pollution
- Read results on-demand

**Paginated Reading**:
```python
read_file("/large_file.txt", offset=0, limit=500)   # First 500 lines
read_file("/large_file.txt", offset=500, limit=500) # Next 500 lines
```

---

## Development Workflow

### 1. Local Development

**Installation**:
```bash
# Using pip
pip install deepagents deepagents-cli

# Using uv (recommended)
uv venv
uv pip install deepagents-cli
```

**Run CLI**:
```bash
# Basic usage
deepagents

# Specify model
deepagents --model gpt-4o

# Auto-approve (skip HITL)
deepagents --auto-approve

# Use remote sandbox
deepagents --sandbox modal
```

### 2. Programmatic Usage

**Basic Example**:
```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    tools=[custom_tool],
    system_prompt="You are an expert programming assistant",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Create a FastAPI app"}]
})
```

**Advanced Configuration**:
```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    model="openai:gpt-4o",
    tools=[web_search, database_tool],
    system_prompt="Specialized data analysis assistant",
    backend=FilesystemBackend(root_dir="/path/to/project"),
    subagents=[
        {
            "name": "researcher",
            "description": "Web research expert",
            "system_prompt": "Perform deep web research",
            "tools": [web_search],
        }
    ],
    interrupt_on={
        "write_file": True,
        "execute": True,
    }
)
```

### 3. Testing

**Run Tests**:
```bash
# deepagents core tests
cd libs/deepagents
make test

# CLI tests
cd libs/deepagents-cli
make test

# Integration tests
make test_integration
```

**Test Coverage**:
- 46 test files
- Unit tests + Integration tests
- pytest + pytest-asyncio

### 4. Evaluation

**Harbor Benchmarking**:
```bash
# Docker local test (1 task)
harbor run --agent-import-path deepagents_harbor:DeepAgentsWrapper \
  --dataset terminal-bench@2.0 -n 1 --env docker

# Daytona cloud test (10 tasks)
harbor run --agent-import-path deepagents_harbor:DeepAgentsWrapper \
  --dataset terminal-bench@2.0 -n 10 --env daytona
```

**LangSmith Tracing**:
```bash
# Configure tracing
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY="lsv2_..."
export LANGSMITH_EXPERIMENT="deepagents-v1"

# Run evaluation
make run-terminal-bench-daytona

# Add feedback scores
python scripts/harbor_langsmith.py add-feedback jobs/terminal-bench/... \
  --project-name deepagents-v1
```

---

## Use Cases

### 1. Code Assistant

**Scenario**: Terminal code assistant similar to Claude Code

**Configuration**:
```bash
# ~/.deepagents/agent/agent.md
I am a senior Python developer
- Use type hints
- Follow PEP 8
- Prefer dataclasses
- Write comprehensive docstrings

# Skills
~/.deepagents/agent/skills/
├── langgraph-docs/   # LangGraph documentation lookup
└── testing/          # Testing best practices
```

**Usage**:
```bash
deepagents
> "Create a FastAPI app with user authentication"
```

### 2. Research Assistant

**Scenario**: Automated web research and synthesis

**Configuration**:
```python
from deepagents import create_deep_agent

research_agent = create_deep_agent(
    tools=[tavily_search, fetch_url],
    system_prompt="""
    You are a research expert. When researching:
    1. Break down into multiple research questions
    2. Launch sub-agents in parallel to research each
    3. Synthesize results into coherent report
    4. Cite all sources
    """,
    subagents=[{
        "name": "researcher",
        "description": "Deep research on single topic",
        "system_prompt": "Conduct thorough research and return detailed findings",
        "tools": [tavily_search, fetch_url],
    }]
)
```

### 3. DevOps Automation

**Scenario**: Automate project setup and deployment

**Project Configuration**:
```bash
# my-project/.deepagents/agent.md
Project: FastAPI microservice
Architecture: Docker + Kubernetes
Deployment: Google Cloud Run

Deployment process:
1. Run tests
2. Build Docker image
3. Push to GCR
4. Update Cloud Run service

# my-project/.deepagents/deployment.md
[Detailed deployment steps and credentials]
```

### 4. Data Analysis

**Scenario**: Automated data analysis pipeline

**Configuration**:
```python
analysis_agent = create_deep_agent(
    tools=[sql_query, pandas_tool, plot_tool],
    system_prompt="""
    You are a data analyst. When analyzing data:
    1. Understand data schema
    2. Perform exploratory data analysis
    3. Generate visualizations
    4. Provide actionable insights
    """,
    backend=FilesystemBackend(root_dir="/data/analysis")
)
```

---

## Comparison with Other Frameworks

| Feature | DeepAgents | LangGraph (Native) | AutoGPT | CrewAI |
|---------|-----------|-------------------|---------|--------|
| **Base Framework** | LangGraph | LangGraph | Custom | LangChain |
| **Filesystem** | ✅ Built-in | ❌ | ✅ | ❌ |
| **Sub-agents** | ✅ Isolated context | ✅ Manual config | ✅ | ✅ Role-based |
| **Task Planning** | ✅ TodoList | ❌ | ✅ | ✅ |
| **Shell Execution** | ✅ Sandboxed | ❌ | ✅ | ❌ |
| **Middleware System** | ✅ Composable | ✅ Custom needed | ❌ | ❌ |
| **Prompt Caching** | ✅ Automatic | ❌ | ❌ | ❌ |
| **Context Management** | ✅ Auto-summarize | ❌ | ❌ | ❌ |
| **CLI Tool** | ✅ | ❌ | ✅ | ❌ |
| **Evaluation Integration** | ✅ Harbor | ❌ | ❌ | ❌ |
| **Learning Curve** | Medium | High | Medium | Low |
| **Customizability** | High | Highest | Medium | Medium |

**DeepAgents Advantages**:
- Out-of-box toolset
- Focus on long-horizon tasks
- Strong context management
- Complete evaluation workflow
- Production-grade security considerations

**When to Choose DeepAgents**:
- ✅ Need filesystem operations
- ✅ Long-horizon, complex tasks
- ✅ Need parallel subtasks
- ✅ Value cost optimization (caching, summarization)
- ✅ Need terminal interaction

**When to Choose Others**:
- LangGraph Native: Need complete custom control flow
- CrewAI: Simple multi-agent collaboration
- AutoGPT: Continuous autonomous operation

---

## Best Practices

### 1. System Prompt Design

**Recommended Structure**:
```markdown
# Role Definition
You are a [domain] expert

# Workflow
When handling [task type]:
1. [First step]
2. [Second step]
3. [Verification step]

# Quality Standards
- [Standard 1]
- [Standard 2]

# Tool Usage Guidance
- For [scenario], use [tool]
- Avoid [anti-pattern]
```

### 2. Sub-agent Usage

**When to Use Sub-agents**:
- ✅ Independent complex subtasks
- ✅ Parallelizable tasks
- ✅ Need context isolation
- ✅ Specialized tool requirements

**When Not to Use**:
- ❌ Simple single-step operations
- ❌ Need state sharing
- ❌ Tightly coupled steps

**Prompt Tips**:
```python
# ✅ Good sub-agent prompt
"Research Python 3.12 new features. Include:
- All new syntax features
- Performance improvements
- Deprecation warnings
Return structured Markdown report with code examples."

# ❌ Poor sub-agent prompt
"Research Python 3.12"  # Too vague
```

### 3. Filesystem Usage

**Path Conventions**:
```python
/                   # Root directory
/tmp/              # Temporary files
/data/             # Data files
/code/             # Code files
/memories/         # Persistent memory (if using CompositeBackend)
```

**Edit Best Practices**:
```python
# ✅ Good edit_file usage
old_str = """def process_data(data):
    return data"""
new_str = """def process_data(data: dict) -> dict:
    \"\"\"Process input data.\"\"\"
    return data"""

# ❌ Poor edit_file usage (not unique)
old_str = "data"  # Too common, will have multiple matches
```

### 4. Task Planning

**When to Use TodoList**:
- 5+ step tasks
- Need progress tracking
- Long-horizon tasks that may need resumption

**TodoList Example**:
```python
write_todos([
    {"task": "Understand existing codebase structure", "completed": False},
    {"task": "Design new feature API", "completed": False},
    {"task": "Implement core functionality", "completed": False},
    {"task": "Write unit tests", "completed": False},
    {"task": "Write integration tests", "completed": False},
    {"task": "Update documentation", "completed": False},
])
```

### 5. Memory Management

**Global vs Project Memory**:

**Global agent.md** (`~/.deepagents/agent/agent.md`):
```markdown
# General Preferences
- Use descriptive variable names
- Add type hints
- Follow Google-style docstrings

# Communication Style
- Concise explanations
- Ask for confirmation before actions
```

**Project agent.md** (`project/.deepagents/agent.md`):
```markdown
# Project: E-commerce API
- Use Pydantic for validation
- All endpoints require authentication
- Follow RESTful conventions

# Testing
- Use pytest
- Target 90% coverage
- Mock external API calls
```

### 6. Security Practices

**HITL Configuration**:
```python
# Production - Strict
interrupt_on={
    "write_file": True,
    "execute": True,
    "web_search": True,
}

# Development - Relaxed
interrupt_on={
    "execute": True,  # Only shell execution needs approval
}

# Automation - No approval
# Don't set interrupt_on or use --auto-approve
```

**Sandbox Usage**:
```bash
# ✅ Safe - Remote sandbox
deepagents --sandbox modal

# ⚠️ Caution - Local execution
deepagents  # shell tool runs locally
```

---

## Technical Debt and Future Directions

### Current Limitations

1. **ACP Module**: Still in development (WIP)
2. **Test Coverage**: Core functionality covered, but some edge cases may be missing
3. **Documentation**: Mainly in English, limited Chinese documentation
4. **Model Support**: Optimized for Claude, other models may perform differently

### Future Directions

Possible development directions inferred from codebase and documentation:

1. **More Model Support**
   - Better open-source model integration
   - Model-specific optimizations

2. **Enhanced Memory System**
   - Vector database integration
   - Semantic retrieval
   - Automatic knowledge extraction

3. **Richer Skills Ecosystem**
   - Skills marketplace
   - Community-contributed skills
   - Skill composition

4. **Improved Evaluation**
   - More benchmarks
   - Automated performance tracking
   - A/B testing framework

5. **Enterprise Features**
   - Team collaboration
   - Audit logs
   - Access control

6. **Tool Extensions**
   - Browser automation
   - API integrations
   - Database operations

---

## Code Quality

### Code Statistics

- **Core Framework**: ~4,811 lines of Python code
- **Tests**: 46 test files
- **Packages**: 4 independent packages

### Toolchain

**Code Quality**:
- **Linter**: Ruff (replaces pylint, flake8, isort)
- **Type Checking**: mypy (strict mode)
- **Formatting**: Ruff format
- **Testing**: pytest + pytest-asyncio + pytest-cov

**Ruff Configuration**:
```toml
[tool.ruff]
line-length = 150  # deepagents
# line-length = 100  # deepagents-cli

[tool.ruff.lint]
select = ["ALL"]  # Enable all rules
ignore = [
    "COM812",   # Conflicts with formatter
    "ISC001",   # Conflicts with formatter
    "C901",     # Complexity checks
    "PLR0913",  # Too many arguments
]

[tool.ruff.lint.pydocstyle]
convention = "google"  # Google-style docstrings
```

**Mypy Configuration**:
```toml
[tool.mypy]
strict = true
ignore_missing_imports = true
```

### Development Workflow

**Local Development**:
```bash
# Install dependencies
uv sync --all-groups

# Run tests
make test

# Code checking
ruff check .
mypy .

# Formatting
ruff format .
```

**Release Process**:
```bash
# Build
python -m build

# Publish
twine upload dist/*
```

---

## Community and Resources

### Official Resources

- **Documentation**: https://docs.langchain.com/oss/python/deepagents/overview
- **API Reference**: https://reference.langchain.com/python/deepagents/
- **Source Code**: https://github.com/langchain-ai/deepagents
- **Quickstarts**: https://github.com/langchain-ai/deepagents-quickstarts

### Community

- **Twitter**: @LangChainAI
- **Slack**: https://www.langchain.com/join-community
- **Reddit**: r/LangChain

### Learning Path

**Beginner**:
1. Read main README
2. Run quickstart examples
3. Try CLI (`deepagents`)
4. Explore example skills

**Intermediate**:
1. Create custom tools
2. Configure sub-agents
3. Implement custom middleware
4. Set up project memory

**Advanced**:
1. Custom backend implementations
2. Harbor evaluation integration
3. LangSmith tracing and optimization
4. Contribute to core framework

---

## Summary

### Core Value Proposition

DeepAgents is a **production-ready AI agent framework** designed for **long-horizon tasks**. It provides:

1. **Complete Toolset**: Filesystem, shell, web search, sub-agents
2. **Smart Context Management**: Auto-summarization, context offloading, prompt caching
3. **Extensible Architecture**: Middleware system, pluggable backends, custom tools
4. **Developer-Friendly**: CLI tool, persistent memory, project awareness
5. **Production-Grade**: Security considerations, HITL, evaluation integration, tracing

### Technical Highlights

1. **Middleware Architecture**: Clear separation of concerns, easy to extend
2. **Backend Abstraction**: Flexible storage strategies (ephemeral/persistent/hybrid)
3. **Sub-agent System**: Context isolation and parallel execution
4. **Progressive Disclosure**: Efficient skill/tool management
5. **Automatic Optimization**: Caching, summarization, context offloading

### Applicable Scenarios

**Best For**:
- Software development assistants
- Research and analysis tasks
- DevOps automation
- Data processing pipelines
- Complex multi-step workflows

**Not Ideal For**:
- Simple Q&A
- Real-time chat applications
- Millisecond-response scenarios
- Highly custom control flows (consider using LangGraph directly)

### Ecosystem Positioning

DeepAgents occupies a **pragmatic** position in the AI agent ecosystem:
- Easier to get started than native LangGraph
- Better suited for long-horizon tasks than AutoGPT/CrewAI
- Provides open-source alternative to Claude Code-like experience
- Deep integration with LangChain/LangGraph ecosystem

---

## Appendix: Glossary

| Term | Explanation |
|------|-------------|
| Deep Agent | AI agent capable of handling complex long-horizon tasks |
| Middleware | Composable agent capability extension modules |
| Backend | Abstraction layer for file storage and execution environments |
| Subagent | Specialized agent with isolated context |
| Sandbox | Isolated code execution environment |
| HITL | Human-in-the-Loop workflow for human decision involvement |
| Context Offloading | Saving large outputs as files to avoid context pollution |
| Progressive Disclosure | Design pattern for loading detailed information on-demand |
| Long-horizon Task | Complex tasks requiring multiple steps and extended time |
| Skill | Encapsulated specialized workflow and knowledge |

---

**Document Version**: 1.0  
**Generated Date**: 2025-12-29  
**Based on Code Version**: deepagents 0.3.1, deepagents-cli 0.0.12  
**Author**: AI Code Analysis Assistant
