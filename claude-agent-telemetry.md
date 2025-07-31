# Claude Agent Telemetry System

## Project Overview
A security audit system for monitoring Claude Code agent activities through structured logging and centralized dashboard visualization.

**Problem Statement**: As AI agents become more prevalent, there's a need to monitor and audit their behavior to ensure they operate within defined security boundaries and don't stray from their intended purpose.

**Solution**: Implement hooks-based telemetry collection with Loki dashboard for comprehensive agent activity monitoring.

## Requirements

### Functional Requirements
- **FR-001**: Capture all Claude tool usage events (Read, Write, Edit, Bash, Grep, etc.)
- **FR-002**: Generate structured logs with context for each action
- **FR-003**: Support session-based and project-based activity grouping
- **FR-004**: Provide centralized dashboard for log visualization and querying
- **FR-005**: Enable post-incident forensic analysis of agent behavior
- **FR-006**: Scale to support multiple concurrent Claude sessions

### Non-Functional Requirements
- **NFR-001**: Zero impact on Claude Code performance
- **NFR-002**: Handle high-volume log ingestion without data loss
- **NFR-003**: Support historical data retention for audit compliance
- **NFR-004**: Provide sub-second query response times on dashboard
- **NFR-005**: Maintain data integrity and tamper-proof audit trail

## Technical Specifications

### Technology Stack
- **Trigger System**: Claude Code hooks
- **Log Format**: JSON structured logs
- **Storage**: Loki (log aggregation)
- **Visualization**: Grafana/Loki dashboard
- **Transport**: HTTP/HTTPS for log shipping

### Log Schema
```json
{
  "timestamp": "2025-07-30T10:30:45.123Z",
  "level": "INFO",
  "event_type": "tool_usage",
  "session_id": "abc123",
  "project_path": "/home/user/webapp",
  "project_name": "webapp",
  "tool_name": "Read",
  "action_details": {
    "file_path": "/src/components/Button.jsx",
    "operation": "read_file",
    "size_bytes": 1024
  },
  "context": {
    "command": "/implement user-authentication",
    "persona": "frontend",
    "reasoning": "reading existing component structure"
  },
  "metadata": {
    "claude_version": "3.5",
    "superclaude_version": "3.0.0",
    "user_id": "jeff"
  }
}
```

### Hook Implementation
- **Location**: `~/.claude/hooks/`
- **Trigger Points**: Pre/post tool execution
- **Data Collection**: Tool name, arguments, file paths, context
- **Transport**: Local file write + log shipper OR direct HTTP to Loki

### Architecture Diagram
```
Claude Code Session
       ↓ (tool usage)
    Hook Trigger
       ↓ (capture event)
  Telemetry Logger
       ↓ (structured JSON)
      Loki Storage
       ↓ (query/visualize)
   Dashboard (Grafana)
```

## Claude Code Hooks Implementation Guide

### Overview
Claude Code hooks are configuration-based scripts that execute at specific points during tool usage. This system enables real-time telemetry capture for security monitoring.

### Required Components

#### 1. Hook Script (`~/.claude/telemetry-hook.sh`)
```bash
#!/bin/bash
# Claude Code Telemetry Hook
# Receives JSON input via stdin and logs tool usage

# Read JSON input from stdin
JSON_INPUT=$(cat)

# Extract data from JSON input
TIMESTAMP=$(date -Iseconds)
PROJECT_PATH="${PWD}"
PROJECT_NAME=$(basename "$PROJECT_PATH")

# Parse Claude Code's actual JSON structure
TOOL_NAME=$(echo "$JSON_INPUT" | sed -n 's/.*"tool_name":"\([^"]*\)".*/\1/p')
SESSION_ID=$(echo "$JSON_INPUT" | sed -n 's/.*"session_id":"\([^"]*\)".*/\1/p')

# Default values if parsing fails
TOOL_NAME="${TOOL_NAME:-unknown}"
SESSION_ID="${SESSION_ID:-unknown}"

# Handle different tool types
case "$TOOL_NAME" in
  "Read")
    FILE_PATH=$(echo "$JSON_INPUT" | sed -n 's/.*"tool_input":.*"file_path":"\([^"]*\)".*/\1/p')
    EVENT_TYPE="file_read"
    ;;
  "Write")
    FILE_PATH=$(echo "$JSON_INPUT" | sed -n 's/.*"tool_input":.*"file_path":"\([^"]*\)".*/\1/p')
    EVENT_TYPE="file_write"
    ;;
  "Edit")
    FILE_PATH=$(echo "$JSON_INPUT" | sed -n 's/.*"tool_input":.*"file_path":"\([^"]*\)".*/\1/p')
    EVENT_TYPE="file_edit"
    ;;
  *)
    FILE_PATH=""
    EVENT_TYPE="tool_usage"
    ;;
esac

# Get file info if applicable
FILE_SIZE=0
OUTSIDE_SCOPE="false"
if [[ -n "$FILE_PATH" && -f "$FILE_PATH" ]]; then
    FILE_SIZE=$(stat -c%s "$FILE_PATH" 2>/dev/null || stat -f%z "$FILE_PATH" 2>/dev/null || echo 0)
    if [[ "$FILE_PATH" != "$PROJECT_PATH"* ]]; then
        OUTSIDE_SCOPE="true"
    fi
fi

# Create telemetry log entry
LOG_ENTRY=$(cat <<EOF
{
  "timestamp": "$TIMESTAMP",
  "level": "INFO",
  "event_type": "$EVENT_TYPE",
  "session_id": "$SESSION_ID",
  "project_path": "$PROJECT_PATH",
  "project_name": "$PROJECT_NAME",
  "tool_name": "$TOOL_NAME",
  "action_details": {
    "file_path": "$FILE_PATH",
    "size_bytes": $FILE_SIZE,
    "outside_project_scope": $OUTSIDE_SCOPE
  },
  "raw_input": $JSON_INPUT
}
EOF
)

# Log to telemetry file
echo "$LOG_ENTRY" >> "/tmp/claude-telemetry.jsonl"

# Return success to continue tool execution
echo '{"continue": true}'
exit 0
```

#### 2. Configuration (`~/.claude/settings.json`)
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/telemetry-hook.sh"
          }
        ]
      }
    ]
  }
}
```

### Claude Code JSON Input Structure
When a tool is used, Claude Code sends JSON via stdin to the hook script:

```json
{
  "session_id": "62d108fd-1cd1-4ec9-8aad-e322e94a6db5",
  "transcript_path": "/home/user/.claude/projects/.../session.jsonl",
  "cwd": "/current/working/directory",
  "hook_event_name": "PreToolUse",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/path/to/file.txt",
    "limit": 10
  }
}
```

### Available Hook Events
- **PreToolUse**: Executes before any tool runs
- **PostToolUse**: Executes after tool completion
- **UserPromptSubmit**: Runs when user submits a prompt
- **Stop**: Runs when the main agent finishes responding
- **SessionStart**: Runs when starting a new session

### Setup Instructions

1. **Create the hook script**:
   ```bash
   nano ~/.claude/telemetry-hook.sh
   chmod +x ~/.claude/telemetry-hook.sh
   ```

2. **Configure Claude Code**:
   ```bash
   nano ~/.claude/settings.json
   # Add the hooks configuration above
   ```

3. **Test the setup**:
   ```bash
   # Clear previous logs
   rm -f /tmp/claude-telemetry.jsonl
   
   # Use Claude Code tools (Read, Write, Edit)
   # Check logs
   cat /tmp/claude-telemetry.jsonl
   ```

4. **Restart Claude Code**: Configuration changes require session restart

### Security Features
- **Scope monitoring**: Detects file access outside project directory
- **Session tracking**: Each Claude session gets unique identifier
- **Tool coverage**: Captures all Claude Code tools (Read, Write, Edit, Bash, etc.)
- **Real-time logging**: Immediate capture of all tool usage

### Troubleshooting
- **No logs generated**: Check hook script permissions and configuration syntax
- **Parse errors**: Verify JSON structure matches expected format
- **Missing tools**: Ensure `matcher: "*"` captures all tools
- **Permission issues**: Verify write access to log directory

### Production Considerations
- **Log rotation**: Implement rotation for `/tmp/claude-telemetry.jsonl`
- **Persistent storage**: Move logs to `/var/log/` for permanent retention
- **Performance**: Hook adds ~1-5ms overhead per tool usage
- **Security**: Secure hook script and configuration files appropriately

## Implementation Phases

### Phase 1: Core Telemetry (MVP)
**Acceptance Criteria**:
- [ ] Hook captures Read, Write, Edit tool usage
- [ ] Generates structured JSON logs locally
- [ ] Basic log schema implemented
- [ ] Session and project identification working

### Phase 2: Log Aggregation
**Acceptance Criteria**:
- [ ] Loki instance configured and running
- [ ] Log shipping from hooks to Loki working
- [ ] Basic dashboard showing tool usage over time
- [ ] Query functionality for filtering by session/project

### Phase 3: Enhanced Context
**Acceptance Criteria**:
- [ ] Capture SuperClaude command context
- [ ] Include persona and reasoning information
- [ ] Add file content change tracking (diffs)
- [ ] Implement comprehensive tool coverage (Bash, Grep, etc.)

### Phase 4: Dashboard & Analytics
**Acceptance Criteria**:
- [ ] Rich Grafana dashboard with multiple views
- [ ] Security-focused panels (unusual access patterns)
- [ ] Historical trend analysis
- [ ] Export capabilities for compliance reporting

## Success Criteria

### Measurable Outcomes
1. **Coverage**: 100% of tool usage events captured
2. **Performance**: <5ms overhead per tool execution
3. **Reliability**: 99.9% log delivery success rate
4. **Usability**: Security incidents detectable within dashboard queries
5. **Scalability**: Support for 10+ concurrent Claude sessions

### Security Monitoring Capabilities
- Detect agents accessing files outside project scope
- Identify unusual tool usage patterns
- Track command execution that deviates from stated purpose
- Monitor file modification patterns for compliance

## Dependencies

### Technical Dependencies
- Claude Code hooks system
- Loki log aggregation platform
- Grafana for dashboard visualization
- JSON parsing and HTTP libraries for log shipping

### Infrastructure Requirements
- Loki instance (local or cloud)
- Storage for log retention (size TBD based on usage)
- Network connectivity for log shipping
- Dashboard hosting environment

## Example Use Cases

### Security Audit Scenario
```
Query: Show all file access outside /src/ directory for session abc123
Expected Result: List of Read/Write operations with full context
Purpose: Verify agent stayed within assigned project scope
```

### Behavioral Analysis
```
Query: Tool usage patterns for user authentication implementation
Expected Result: Timeline of Read → Edit → Write operations
Purpose: Understand normal vs anomalous implementation patterns
```

## Future Considerations
- Real-time alerting for security violations
- Machine learning for anomaly detection
- Integration with external security tools
- Multi-tenant support for team environments
- Permission profile enforcement integration