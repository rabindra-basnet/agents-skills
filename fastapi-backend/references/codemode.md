# Code Execution Mode

Let agents execute Python code in a sandboxed environment. Use this for data analysis,
calculations, and dynamic code generation. Never execute untrusted code without isolation.

## Sandboxed execution with DuckDB

```python
# app/features/agents/codemode/executor.py
import ast
import duckdb
from app.core.logging import get_logger

logger = get_logger(__name__)

ALLOWED_MODULES = {"math", "json", "datetime", "collections", "itertools", "functools"}
BLOCKED_FUNCTIONS = {"eval", "exec", "compile", "__import__", "open", "execfile"}

class CodeExecutor:
    def __init__(self):
        self.conn = duckdb.connect(":memory:")

    def validate_code(self, code: str) -> bool:
        """Validate code is safe to execute."""
        try:
            tree = ast.parse(code)
        except SyntaxError:
            return False

        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    if alias.name.split(".")[0] not in ALLOWED_MODULES:
                        return False
            if isinstance(node, ast.Name) and node.id in BLOCKED_FUNCTIONS:
                return False
            if isinstance(node, ast.Attribute) and node.attr in BLOCKED_FUNCTIONS:
                return False

        return True

    async def execute(self, code: str, timeout: int = 30) -> dict:
        """Execute Python code in sandbox."""
        if not self.validate_code(code):
            return {"error": "Code contains forbidden modules or functions"}

        logger.info("Code execution started", extra={"code_length": len(code)})

        try:
            # Execute with DuckDB context
            result = self.conn.execute(code).fetchall()
            logger.info("Code execution completed", extra={"rows": len(result)})
            return {"result": result, "status": "success"}
        except Exception as e:
            logger.error("Code execution failed", extra={"error": str(e)})
            return {"error": str(e), "status": "failed"}
```

## Agent tool

```python
# app/features/agents/codemode/tool.py
from app.features.agents.codemode.executor import CodeExecutor
from app.core.logging import get_logger

logger = get_logger(__name__)

executor = CodeExecutor()

async def execute_code(ctx: dict, *, code: str) -> dict:
    """Execute Python code safely."""
    result = await executor.execute(code)
    logger.info("Code executed", extra={"status": result["status"]})
    return result
```

## Usage pattern

```python
# Agent can execute code for data analysis
await execute_code(ctx, code="""
import duckdb
result = conn.execute('SELECT * FROM data WHERE value > 100').fetchall()
print(f"Found {len(result)} rows")
""")
```

## DO NOT

- **Never** execute untrusted code without AST validation — always validate first.
- **Never** allow `import os`, `import sys`, `import subprocess` — they escape the sandbox.
- **Never** allow `eval()`, `exec()`, `compile()` — they bypass validation.
- **Never** skip timeout on code execution — infinite loops will hang the worker.
- **Never** execute code in the main process — use a separate worker/container.
- **Never** store executed code logs with full source — they may contain sensitive data.
- **Never** allow file I/O operations — they escape the sandbox.
- **Never** skip resource limits (memory, CPU) — runaway code will consume resources.
- **Never** execute code that modifies system state — keep it read-only where possible.
