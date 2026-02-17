# AGENTS.md - TrendRadar Development Guide

## Project Overview

TrendRadar is a Python-based trending news aggregation and analysis tool with an MCP (Model Context Protocol) server for AI integration. The project consists of:
- `trendradar/` - Main application package
- `mcp_server/` - MCP server implementation using FastMCP 2.0
- `config/` - Configuration files
- `output/` - Data output directory

## Build, Lint, and Test Commands

### Installation & Dependencies

```bash
# Install using uv (recommended)
uv sync

# Or using pip
pip install -e .
```

### Running the Application

```bash
# Run main TrendRadar application
python -m trendradar

# Run MCP server in stdio mode (default)
python -m mcp_server.server

# Run MCP server in HTTP mode
python -m mcp_server.server --transport http --port 3333
```

### Development Commands

```bash
# Check version
cat version

# Run setup script (Mac)
./setup-mac.sh
```

### Single Test Execution

**Note:** This project currently has **no test suite** implemented. If adding tests, use pytest:

```bash
# Run all tests
pytest

# Run a single test file
pytest tests/test_specific.py

# Run a single test function
pytest tests/test_specific.py::test_function_name

# Run tests matching a pattern
pytest -k "test_pattern"
```

### Type Checking

The project uses Python type hints. For type checking (if added):

```bash
# Using mypy (if configured)
mypy trendradar/ mcp_server/
```

## Code Style Guidelines

### General Conventions

- **Language**: Python 3.10+
- **Encoding**: UTF-8 (use `# coding=utf-8` at the top of files)
- **Line Length**: No hard limit, but keep lines reasonably short (<120 chars when practical)
- **Indentation**: 4 spaces

### Imports

**Organize imports in the following order** (separated by blank lines):
1. Standard library imports
2. Third-party imports
3. Local application imports

```python
# coding=utf-8
"""
Module docstring
"""

import os
import json
from typing import Dict, List, Optional, Tuple

import requests
from fastmcp import FastMCP

from trendradar.context import AppContext
from trendradar.core import load_config
from trendradar.crawler import DataFetcher
```

### Type Hints

- Use type hints for all function parameters and return types
- Use `Optional[T]` instead of `T | None` for compatibility
- Use `Dict`, `List`, `Tuple` from typing (not built-in dict/list)

```python
def process_data(
    config: Dict[str, Any],
    platforms: Optional[List[str]] = None,
    limit: int = 50
) -> Tuple[List[Dict], Optional[str]]:
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Modules | snake_case | `data_query.py` |
| Classes | PascalCase | `AppContext` |
| Functions | snake_case | `get_latest_news` |
| Variables | snake_case | `news_data` |
| Constants | UPPER_SNAKE_CASE | `DEFAULT_TIMEZONE` |
| Private functions | _snake_case | `_internal_helper` |

### Docstrings

Use Google-style docstrings with Chinese comments (project is primarily Chinese-language):

```python
def fetch_data(platforms: List[str]) -> Dict[str, Any]:
    """
    获取指定平台的数据

    Args:
        platforms: 平台ID列表

    Returns:
        平台数据字典

    Raises:
        ValueError: 平台列表为空时
    """
```

### Error Handling

- Use custom exception classes for domain-specific errors
- Include error codes and suggestions in exceptions
- Catch specific exceptions, avoid bare `except:`

```python
class MCPError(Exception):
    """MCP工具错误基类"""

    def __init__(self, message: str, code: str = "MCP_ERROR", suggestion: Optional[str] = None):
        super().__init__(message)
        self.code = code
        self.message = message
        self.suggestion = suggestion

    def to_dict(self) -> dict:
        """转换为字典格式"""
        return {"code": self.code, "message": self.message}


# Usage
try:
    result = fetch_data(platforms)
except ValueError as e:
    logger.error(f"Invalid input: {e}")
    raise
```

### File Organization

- One class per file is preferred for public classes
- Use `__init__.py` to expose public API from packages
- Group related functionality into modules under appropriate packages:
  - `trendradar/core/` - Core functionality
  - `trendradar/crawler/` - Data fetching
  - `trendradar/storage/` - Data persistence
  - `trendradar/notification/` - Notification dispatch
  - `trendradar/ai/` - AI integration
  - `mcp_server/tools/` - MCP tool implementations
  - `mcp_server/services/` - Business logic services

### Async/Await

- Use `asyncio` for MCP server (FastMCP is async-based)
- Use `asyncio.to_thread()` for wrapping synchronous code in async functions

```python
@mcp.tool
async def get_data(param: str) -> str:
    tools = _get_tools()
    result = await asyncio.to_thread(tools['data'].get_something, param=param)
    return json.dumps(result, ensure_ascii=False, indent=2)
```

### Configuration

- Configuration is YAML-based (`config/config.yaml`)
- Use `AppContext` class to encapsulate configuration access
- Avoid global configuration state; pass context explicitly

### JSON Handling

- Always use `ensure_ascii=False` and `indent=2` for JSON output
- Return JSON strings from MCP tools (not Python objects)

```python
return json.dumps({
    "key": "value",
    "description": "说明"
}, ensure_ascii=False, indent=2)
```

### String Formatting

- Use f-strings for simple interpolation
- Use `.format()` for complex cases
- Keep Chinese text in strings for user-facing messages

```python
print(f"监控平台数量: {len(config['PLATFORMS'])}")
print("当前北京时间: {}".format(now.strftime('%Y-%m-%d %H:%M:%S')))
```

### Logging and Output

- Use `print()` for user-facing output (CLI tool)
- Use appropriate log levels for debugging
- Include timestamps and context in log messages

## MCP Server Development

### Tool Registration

Register tools using the `@mcp.tool` decorator:

```python
@mcp.tool
async def tool_name(param: str) -> str:
    """Tool docstring for AI agents"""
    # Implementation
    return json.dumps(result, ensure_ascii=False, indent=2)
```

### Resource Registration

Register resources using the `@mcp.resource` decorator:

```python
@mcp.resource("config://platforms")
async def get_platforms() -> str:
    return json.dumps({...}, ensure_ascii=False, indent=2)
```

### Error Responses

Return error responses as JSON:

```python
return json.dumps({
    "success": False,
    "error": {
        "code": "ERROR_CODE",
        "message": "错误信息"
    }
}, ensure_ascii=False, indent=2)
```

## Key File Locations

- Main entry: `trendradar/__main__.py`
- MCP server: `mcp_server/server.py`
- Configuration: `config/config.yaml`
- Frequency words: `config/frequency_words.txt`
- Output: `output/` directory

## Common Tasks

### Adding a new MCP tool

1. Add tool function in appropriate `mcp_server/tools/` module
2. Register with `@mcp.tool` decorator in `mcp_server/server.py`
3. Document with detailed docstring including Args, Returns, Examples

### Adding a new notification channel

1. Add sender implementation in `trendradar/notification/senders.py`
2. Register in `NotificationDispatcher`
3. Add format renderer if needed in `trendradar/notification/renderers.py`
