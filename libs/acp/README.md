# ACP

DeepAgents integration with Agent Client Protocol (ACP).

## Status

The ACP integration is **functional and tested** with support for core ACP features:

### ✅ Implemented Features
- **Session Management**: Create and manage agent sessions
- **Streaming Responses**: Stream AI messages and thoughts in real-time
- **Tool Calling**: Execute tools with progress tracking
- **Human-in-the-Loop**: Request user approval for sensitive operations
- **Plan/Todo Updates**: Track agent planning and progress
- **Cancellation**: Cancel running sessions

### 🔧 Technical Details
- Compatible with ACP protocol version 0.7+
- Uses snake_case field naming (per ACP 0.7+ standard)
- All tests passing (5/5)
- Python 3.11+ compatible

### 📝 Notes
- Edit functionality is not supported (ACP protocol limitation)
- Full session persistence requires LangGraph checkpointer serialization
- Extension methods (`extMethod`) are marked as optional per ACP spec

## Installation

```bash
pip install deepagents-acp
```

## Usage

```python
from deepagents_acp.server import main
import asyncio

# Run the ACP server
asyncio.run(main())
```

Or use the CLI:

```bash
deepacp
```

## Testing

```bash
# Run tests
pytest tests/ -v --disable-socket --allow-unix-socket

# Run with coverage
pytest tests/ --cov=deepagents_acp
```
