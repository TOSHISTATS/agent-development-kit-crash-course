# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a stateful multi-agent system built with Google ADK (Agent Development Kit) that demonstrates combining persistent state management with multi-agent delegation. It implements a customer service system for an online course platform where specialized agents handle different aspects of customer support while sharing a common session state.

## Running the Application

```bash
# Activate virtual environment from parent directory
source ../.venv/bin/activate  # macOS/Linux
..\.venv\Scripts\activate.bat  # Windows CMD
..\.venv\Scripts\Activate.ps1  # Windows PowerShell

# Run the main application
python main.py
```

## Dependencies

The project uses dependencies from the parent directory's `requirements.txt`:
- `google-adk[database]==1.20.0` - Core agent framework
- `google-generativeai==0.8.5` - Gemini model integration
- `python-dotenv==1.1.0` - Environment variable management
- Other supporting libraries (yfinance, psutil, litellm)

Environment variables are stored in `.env` (requires `GOOGLE_API_KEY`).

## Architecture

### Session State Management

The system uses `InMemorySessionService` to maintain persistent state across agent interactions:

```python
state = {
    "user_name": str,              # User's name
    "purchased_courses": [dict],   # List of course objects: {"id": str, "purchase_date": str}
    "interaction_history": [dict]  # List of interaction entries with timestamps
}
```

**Critical**: All agents share the same session state. State updates made by tools in sub-agents are immediately visible to all other agents through the session service.

### Multi-Agent Hierarchy

```
customer_service_agent (root)
├── policy_agent - Handles policy/refund questions
├── sales_agent - Manages course purchases
├── course_support_agent - Provides course content help
└── order_agent - Shows purchase history, processes refunds
```

The root agent (`customer_service_agent`) analyzes queries and delegates to specialized sub-agents. All agents receive state context through template variables in their instructions: `{user_name}`, `{purchased_courses}`, `{interaction_history}`.

### State Updates via Tools

Tools in sub-agents can modify state using `ToolContext`:

```python
def purchase_course(tool_context: ToolContext) -> dict:
    # State updates via direct assignment
    tool_context.state["purchased_courses"] = new_purchased_courses
    tool_context.state["interaction_history"] = new_interaction_history
```

**Important patterns**:
- Course data is stored as dictionaries: `{"id": "ai_marketing_platform", "purchase_date": "YYYY-MM-DD HH:MM:SS"}`
- Always copy existing state lists before modifying to avoid mutations
- Filter out empty/invalid entries when processing purchased courses
- Check course ownership by extracting IDs: `[course["id"] for course in courses if isinstance(course, dict)]`

### File Organization

```
customer_service_agent/
├── __init__.py                           # Required for ADK agent discovery
├── agent.py                              # Root agent with routing logic
└── sub_agents/
    ├── course_support_agent/
    │   ├── __init__.py                   # Package marker
    │   └── agent.py                      # Course support agent definition
    ├── order_agent/
    │   └── agent.py                      # Purchase history & refund tools
    ├── policy_agent/
    │   ├── __init__.py
    │   └── agent.py                      # Policy information agent
    └── sales_agent/
        ├── __init__.py
        └── agent.py                      # Purchase tool implementation
```

**ADK Discovery**: The `__init__.py` files are required in the root agent directory for ADK to discover and load the agent properly.

## Key Implementation Details

### Agent Runner Pattern

The main loop in `main.py` follows this pattern:
1. Create session with initial state
2. Initialize `Runner` with agent and session service
3. For each user input:
   - Add query to interaction history via `add_user_query_to_history()`
   - Process through `runner.run_async()` which returns event stream
   - Display state before and after processing
   - Extract and store agent responses

### State Utility Functions

Located in `utils.py`:
- `update_interaction_history()` - Generic function to add entries to history with timestamps
- `add_user_query_to_history()` - Convenience wrapper for user queries
- `add_agent_response_to_history()` - Convenience wrapper for agent responses
- `display_state()` - Pretty-print current session state with formatting
- `call_agent_async()` - Process queries and handle event streams

### Tool Implementation Best Practices

When implementing tools that modify state:
1. Get current state values via `tool_context.state.get()`
2. Validate current state (e.g., check if course already owned)
3. Create new lists/dicts (don't mutate in place)
4. Filter out invalid entries (empty strings, non-dict items)
5. Update state via assignment: `tool_context.state[key] = new_value`
6. Return structured dict with `status`, `message`, and relevant data

### Conditional Agent Access

The system implements access control through instruction logic:
- `course_support_agent` should only help users who own "ai_marketing_platform"
- The root agent checks purchased courses before routing to course support
- Agents use state templates to access purchase info in their instructions

## Working with This Codebase

### Adding a New Sub-Agent

1. Create new directory under `customer_service_agent/sub_agents/`
2. Add `__init__.py` for package recognition
3. Create `agent.py` with `Agent` definition
4. Include state templates in instruction: `{user_name}`, `{purchased_courses}`, `{interaction_history}`
5. Import and add to root agent's `sub_agents` list in `customer_service_agent/agent.py`

### Modifying State Structure

If adding new state fields:
1. Update `initial_state` dict in `main.py`
2. Add corresponding template variable to agent instructions
3. Update `display_state()` in `utils.py` to show new fields
4. Update any tools that read/write the new field

### Testing State Persistence

The system logs state before and after each interaction. Use `display_state()` to verify:
- State changes from tool calls persist correctly
- Purchased courses are stored as proper dict objects
- Interaction history accumulates chronologically
- Changes made by one agent are visible to subsequent agents
