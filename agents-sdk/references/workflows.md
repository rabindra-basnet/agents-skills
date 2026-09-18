# Workflows Integration

## Overview

Agents handle real-time communication; Workflows handle durable execution. Together they enable:

- Long-running background tasks with automatic retries
- Human-in-the-loop approval flows
- Multi-step pipelines that survive failures

| Use Case | Recommendation |
|----------|----------------|
| Chat/messaging | Agent only |
| Quick API calls (<30s) | Agent only |
| Background processing (<30s) | Agent `queue()` |
| Long-running tasks (>30s) | Agent + Workflow |
| Human approval flows | Agent + Workflow |

## AgentWorkflow Base Class

```python
from agents_sdk import AgentWorkflow, workflow_step

class ProcessingWorkflow(AgentWorkflow):
    @workflow_step("process")
    async def process(self, params: dict, step):
        # Durable step - retries on failure
        result = await step.do("process", lambda: processData(params["data"]))
        
        # Non-durable: progress reporting
        await self.report_progress({"step": "process", "percent": 0.5})
        
        # Non-durable: broadcast to connected clients
        self.broadcast_to_clients({"type": "update", "task_id": params["task_id"]})
        
        # Durable: merge state via step
        await step.merge_agent_state({"last_processed": params["task_id"]})
        
        # Durable: report completion
        await step.report_complete(result)
        
        return result
```

## Configuration

```python
from agents_sdk import AgentManager

manager = AgentManager()

@manager.workflow("processing-workflow")
class ProcessingWorkflow(AgentWorkflow):
    # ... workflow implementation
    pass
```

## Agent Methods for Workflows

```python
# Start a workflow
instance = await self.run_workflow("ProcessingWorkflow", {"task_id": "123", "data": "..."})

# Send event to waiting workflow
await self.send_workflow_event("ProcessingWorkflow", workflow_id, {"type": "approve"})

# Query workflows
workflow = await self.get_workflow(workflow_id)
workflows = await self.get_workflows(status="running")

# Control workflows
await self.approve_workflow(workflow_id)
await self.reject_workflow(workflow_id)
await self.terminate_workflow(workflow_id)
await self.pause_workflow(workflow_id)
await self.resume_workflow(workflow_id)

# Delete workflows
await self.delete_workflow(workflow_id)
await self.delete_workflows(status="complete", before=datetime.now())
```

## Lifecycle Callbacks

```python
class MyAgent(Agent):
    async def on_workflow_progress(self, workflow_name: str, workflow_id: str, progress: dict):
        # Workflow reported progress via self.report_progress()
        self.broadcast({"type": "progress", "workflow_id": workflow_id, "progress": progress})

    async def on_workflow_complete(self, workflow_name: str, workflow_id: str, result=None):
        # Workflow finished successfully
        pass

    async def on_workflow_error(self, workflow_name: str, workflow_id: str, error: Exception):
        # Workflow failed
        pass

    async def on_workflow_event(self, workflow_name: str, workflow_id: str, event: dict):
        # Workflow received an event via self.send_workflow_event()
        pass
```

## Human-in-the-Loop

```python
# In workflow: wait for approval
approved = await step.wait_for_event("approval", timeout_days=7)

if not approved.get("approved"):
    raise ValueError("Rejected")

# From agent: approve or reject
await self.approve_workflow(workflow_id)  # Sends {"approved": True}
await self.reject_workflow(workflow_id)   # Sends {"approved": False}
```
