# Technical Clarifications for MVP Implementation

This document clarifies implementation-specific decisions and constraints for the MVP.

---

## Deployment Model

**LOCAL HTTP ONLY** - No Lambda/AWS deployment for this MVP

**Implications**:
- No IAM authentication for calendar service API calls
- No AWS SigV4 request signing
- Simple HTTP requests with standard authentication (if any)
- Calendar service runs locally or on accessible HTTP endpoint
- Bot runs as a local process (cron job or systemd service)

---

## Database Configuration

**Supabase Tables**: 2 tables (see `scripts/supabase_schema.sql`)

1. **workflows** - Workflow instances and state
   - Indexed by: group_id, status
   - Contains: metadata + `state_data` JSONB column with serialized LangGraph state
   - Query pattern: Get active workflow for group (includes state in same query)

2. **conversations** - All messages
   - Indexed by: group_id, user_id, timestamp
   - Query pattern: Get last N messages for context

**Manual Setup**: SQL provided in file, user will run manually in Supabase dashboard

**Database Simplicity**:
- No stored procedures or functions
- No triggers (updated_at set explicitly in code)
- Just tables, indexes, and constraints
- All business logic in Python

---

## LLM Configuration

**Provider**: OpenAI (ChatGPT)
**Model**: GPT-4 or GPT-4-turbo (configurable)
**Temperature**: 0.3 (for consistent extractions)

**System Prompt**: Externalized to `ai_prompts/system_prompt.txt`
- Loaded at runtime
- Formatted with variables: current_datetime, sender_name, sender_squad, sender_role, user_message
- Can be edited without code changes

---

## Workflow Behavior

### Parameter Extraction Strategy

**Multi-parameter extraction**: Extract as many parameters as possible from one message

Example flows:

**Flow 1: All parameters in one message**
```
User: "Squad 42 can't make Saturday night"
Bot: [Extracts: squad=42, date=Saturday, shift=night, action=noCrew]
Bot: [Validates and executes]
Bot: "Marked Squad 42 as unavailable for Saturday night shift (12/23 1800-0600)"
```

**Flow 2: Iterative parameter collection**
```
User: "Can't make it Saturday"
Bot: [Extracts: date=Saturday, missing: squad, shift, action]
Bot: "Which squad won't be available on Saturday?"

User: "Squad 42"
Bot: [Extracts: squad=42, missing: shift, action]
Bot: "What time on Saturday - day shift (6am-6pm) or night shift (6pm-6am)?"

User: "Night"
Bot: [Extracts: shift=night, action inferred as noCrew]
Bot: [Validates and executes]
Bot: "Marked Squad 42 as unavailable for Saturday night shift (12/23 1800-0600)"
```

### Missing Parameter Priority

When asking for clarifications, prioritize in this order:
1. Squad (most critical)
2. Date
3. Time/shift
4. Action (often can be inferred)

### Validation Rules

Before executing:
1. Verify squad exists in roster
2. Check if squad is currently scheduled
3. Count remaining active crews
4. If count would drop to 0, send CRITICAL warning but still execute if confirmed
5. Validate date/time formats

---

## Message Storage Policy

**Store ALL messages** from GroupMe, not just shift-related ones

**Rationale**:
- Provides full conversation context
- Enables future features (analytics, user behavior)
- Disk is cheap, context is valuable

**Retention**: No automatic deletion for MVP (can be added later)

**Query Pattern**:
```sql
-- Get last 20 messages for context
SELECT * FROM conversations
WHERE group_id = ?
ORDER BY timestamp DESC
LIMIT 20
```

---

## Workflow Lifecycle

```
NEW
  ↓
WAITING_FOR_INPUT (when asking clarifying question)
  ↓ (user responds)
READY (all parameters collected)
  ↓
EXECUTING (calling calendar API)
  ↓
COMPLETED (success) or EXPIRED (timeout)
```

**Expiration**: 24 hours from creation (configurable)

**Expiration Handling**:
- Application code checks for expired workflows on startup and during each poll cycle
- Simple UPDATE query: `UPDATE workflows SET status='EXPIRED' WHERE expires_at < NOW()`
- Logged in application logs (e.g., "Expired 3 workflow(s)")
- Expired workflows cannot be resumed
- User must start fresh request

**One Active Workflow Per Group**:
- Before starting new workflow, check for active workflows
- If found, reject new request with message: "Please complete the current request first"

---

## Logging Strategy

**Framework**: Python standard logging module

**Handlers**:
1. Console (INFO and above)
2. File: `logs/chatbot.log` (DEBUG and above)
3. File: `logs/errors.log` (ERROR and above)

**Log Rotation**: Not implemented in MVP (can use logrotate externally)

**Key Events to Log**:
- Message received from GroupMe
- Message stored in database
- Workflow created/resumed
- LLM API calls (request + response summary)
- Tool executions
- Parameter extraction results
- Validation warnings
- Calendar API calls
- Execution results
- Errors with full stack traces

**Log Format**:
```
2025-12-21 14:30:45 - module_name - INFO - Message text
2025-12-21 14:30:46 - module_name - ERROR - Error message
```

---

## Error Handling

### API Failures

**Calendar API**:
- Retry up to 3 times with exponential backoff
- Log each attempt
- If all attempts fail, mark workflow as failed and notify in chat

**GroupMe API** (sending messages):
- Retry up to 2 times
- Log failures but don't crash the process
- Failed bot messages should not block workflow execution

**Supabase**:
- Log connection errors
- Retry transient failures (connection timeouts)
- Hard fail on schema errors (bug in our code)

**LLM API**:
- Retry on rate limits (with backoff)
- Retry on transient errors
- Hard fail on invalid API key (config error)
- Log all API calls for debugging

### Graceful Degradation

- If LLM fails to extract parameters, ask user to rephrase
- If calendar API is down, queue the command and retry later (future enhancement)
- If database is down, log error and fail fast (critical dependency)

---

## Testing Approach

**MVP Testing**: Manual testing only

**Test Scenarios** (from Implementation Plan):
1. Simple single-message request
2. Multi-turn conversation
3. Overlapping workflow rejection
4. Workflow expiration

**Future**: Unit tests for core logic (parameter extraction, validation)

**Logging as Testing Tool**:
- Comprehensive logging allows debugging via log inspection
- Each test run should be fully traceable through logs

---

## Security Considerations

**Supabase**:
- Use service role key (full access)
- Enable Row Level Security (RLS) policies
- Only service role can read/write

**API Keys**:
- Stored in .env file
- Never commit to git
- .env in .gitignore

**GroupMe**:
- Validate message sender is in roster (authorized users only)
- Ignore messages from unknown users

**Input Validation**:
- Pydantic models for all data structures
- Validate dates, times, squad numbers
- Sanitize inputs before sending to LLM or calendar API

---

## Future Enhancements (Out of Scope for MVP)

These are explicitly NOT in the MVP but may be added later:

1. Multiple concurrent workflows per group
2. Workflow conflict resolution
3. Visual workflow inspection (web UI)
4. Multiple LLM providers
5. Non-shift workflows (other use cases)
6. Message retention/cleanup policies
7. User permission levels
8. Workflow templates
9. Command queuing and batch execution
10. Webhook-based GroupMe integration (vs polling)

---

## Dependencies

**Python Version**: 3.11+

**Key Libraries**:
```
langchain
langchain-openai
langgraph
supabase-py
pydantic
pydantic-settings
requests
python-dotenv
```

**Installation**:
```bash
pip install langchain langchain-openai langgraph supabase pydantic pydantic-settings requests python-dotenv
```

(A requirements.txt will be created during implementation)

---

## Deployment

**For MVP**: Local execution

**Options**:
1. Cron job (every 2 minutes)
   ```
   */2 * * * * cd /path/to/project && python -m src.poll_messages
   ```

2. Systemd timer (Linux)
   ```
   [Unit]
   Description=Poll GroupMe messages

   [Timer]
   OnBootSec=1min
   OnUnitActiveSec=2min

   [Install]
   WantedBy=timers.target
   ```

3. While loop with sleep (simple but effective)
   ```bash
   while true; do
       python -m src.poll_messages
       sleep 120
   done
   ```

**Note**: Process must stay running to maintain state. If killed, workflows resume from database on restart.
