# Asana MCP Server - AI Agent Installation Guide

This guide provides comprehensive installation instructions for AI agents installing the Asana MCP server in OI OS (Brain Trust 4) environments, as well as end-user setup instructions.

## Table of Contents

1. [AI Agent Quick Installation](#ai-agent-quick-installation)
2. [Prerequisites](#prerequisites)
3. [Full Installation Steps](#full-installation-steps)
4. [Configuring Authentication](#configuring-authentication)
5. [Connecting to OI OS](#connecting-to-oi-os)
6. [Configuring Parameter Extractors](#configuring-parameter-extractors)
7. [Creating Parameter Rules](#creating-parameter-rules)
8. [Creating Intent Mappings](#creating-intent-mappings)
9. [End User Setup](#end-user-setup)
10. [Verification & Testing](#verification--testing)
11. [Troubleshooting](#troubleshooting)
12. [Known Issues & Fixes](#known-issues--fixes)

---

## AI Agent Quick Installation

**⚠️ For AI Agents: Use Direct Calls for Reliability**

AI agents should prefer **direct `brain-trust4 call` commands** over natural language queries for maximum reliability. Natural language commands can timeout or have parameter extraction issues. Direct calls bypass intent mapping and parameter extraction, providing consistent results.

**Example:**
```bash
# ✅ RECOMMENDED for AI agents: Direct call
./brain-trust4 call mcp-server-asana asana_create_task '{"project_id": "1234567890123", "name": "New task"}'

# ⚠️ FALLBACK: Natural language (may timeout or fail parameter extraction)
./oi "asana create task in project 1234567890123 New task"
```

### Using Human-Friendly Names (Name Resolution)

The OI OS system includes a **universal name→ID mapping system** (`brain.json`) that allows using human-friendly names instead of cryptic numeric IDs. This system is located in the `ID MAPPING/` folder in your OI OS installation.

**Benefits:**
- Use project names like `"OI OS"` instead of `"1211709902166635"`
- Use workspace names like `"Good Wolf"` instead of numeric GIDs
- Auto-learns from API responses
- Auto-syncs when mappings are missing

**Example with Name Resolution:**
```bash
# Using resolve-params.sh wrapper (resolves names to IDs automatically)
cd "ID MAPPING"
./resolve-params.sh mcp-server-asana asana_create_task '{"project_id": "OI OS", "name": "New task"}'

# This automatically:
# 1. Looks up "OI OS" → "1211709902166635" in brain.json
# 2. Calls the API with the resolved ID
# 3. Learns new mappings from the response
```

**Syncing Mappings:**
```bash
# Sync all servers (including Asana)
cd "ID MAPPING"
./sync-all-mappings.sh

# Or sync just Asana
./sync-name-cache.sh mcp-server-asana
```

**Note:** For direct `brain-trust4 call` commands, you still need to use numeric IDs. The name resolution system works with the `resolve-params.sh` wrapper or will be integrated into the `brain-trust4` binary in future releases.

For AI agents using OI OS, execute the following commands:

```bash
# 1. Install the server
./oi install https://github.com/OI-OS/mcp-server-asana.git

# 2. Connect the server to OI OS
# NOTE: brain-trust4 automatically loads .env file from project root
cd MCP-servers/mcp-server-asana
cd ../../
./brain-trust4 connect mcp-server-asana node -- "$(pwd)/MCP-servers/mcp-server-asana/dist/index.js"

# 3. Create intent mappings and parameter rules (single optimized transaction)
sqlite3 brain-trust4.db << 'SQL'
BEGIN TRANSACTION;

-- Intent mappings for Asana MCP server (most common operations)
INSERT OR REPLACE INTO intent_mappings (keyword, server_name, tool_name, priority) VALUES 
('asana list workspaces', 'mcp-server-asana', 'asana_list_workspaces', 10),
('asana workspaces', 'mcp-server-asana', 'asana_list_workspaces', 10),
('asana show workspaces', 'mcp-server-asana', 'asana_list_workspaces', 10),
('asana search projects', 'mcp-server-asana', 'asana_search_projects', 10),
('asana find projects', 'mcp-server-asana', 'asana_search_projects', 10),
('asana list projects', 'mcp-server-asana', 'asana_search_projects', 10),
('asana search tasks', 'mcp-server-asana', 'asana_search_tasks', 10),
('asana find tasks', 'mcp-server-asana', 'asana_search_tasks', 10),
('asana list tasks', 'mcp-server-asana', 'asana_search_tasks', 10),
('asana get task', 'mcp-server-asana', 'asana_get_task', 10),
('asana show task', 'mcp-server-asana', 'asana_get_task', 10),
('asana create task', 'mcp-server-asana', 'asana_create_task', 10),
('asana add task', 'mcp-server-asana', 'asana_create_task', 10),
('asana new task', 'mcp-server-asana', 'asana_create_task', 10),
('asana update task', 'mcp-server-asana', 'asana_update_task', 10),
('asana edit task', 'mcp-server-asana', 'asana_update_task', 10),
('asana get project', 'mcp-server-asana', 'asana_get_project', 10),
('asana show project', 'mcp-server-asana', 'asana_get_project', 10);

-- Parameter rules for Asana MCP server
-- asana_list_workspaces: no required fields
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_list_workspaces', 'mcp-server-asana::asana_list_workspaces', '[]',
'{"opt_fields": {"FromQuery": "mcp-server-asana::asana_list_workspaces.opt_fields"}}', '[]');

-- asana_search_projects: name_pattern is required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_search_projects', 'mcp-server-asana::asana_search_projects', '["name_pattern"]',
'{"name_pattern": {"FromQuery": "mcp-server-asana::asana_search_projects.name_pattern"}, "workspace": {"FromQuery": "mcp-server-asana::asana_search_projects.workspace"}, "opt_fields": {"FromQuery": "mcp-server-asana::asana_search_projects.opt_fields"}}', '[]');

-- asana_search_tasks: workspace is required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_search_tasks', 'mcp-server-asana::asana_search_tasks', '["workspace"]',
'{"workspace": {"FromQuery": "mcp-server-asana::asana_search_tasks.workspace"}, "text": {"FromQuery": "mcp-server-asana::asana_search_tasks.text"}, "projects_any": {"FromQuery": "mcp-server-asana::asana_search_tasks.projects_any"}, "completed": {"FromQuery": "mcp-server-asana::asana_search_tasks.completed"}, "assignee_any": {"FromQuery": "mcp-server-asana::asana_search_tasks.assignee_any"}}', '[]');

-- asana_get_task: task_id is required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_get_task', 'mcp-server-asana::asana_get_task', '["task_id"]',
'{"task_id": {"FromQuery": "mcp-server-asana::asana_get_task.task_id"}, "opt_fields": {"FromQuery": "mcp-server-asana::asana_get_task.opt_fields"}}', '[]');

-- asana_create_task: project_id and name are required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_create_task', 'mcp-server-asana::asana_create_task', '["project_id", "name"]',
'{"project_id": {"FromQuery": "mcp-server-asana::asana_create_task.project_id"}, "name": {"FromQuery": "mcp-server-asana::asana_create_task.name"}, "notes": {"FromQuery": "mcp-server-asana::asana_create_task.notes"}, "due_on": {"FromQuery": "mcp-server-asana::asana_create_task.due_on"}, "assignee": {"FromQuery": "mcp-server-asana::asana_create_task.assignee"}}', '[]');

-- asana_update_task: task_id is required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_update_task', 'mcp-server-asana::asana_update_task', '["task_id"]',
'{"task_id": {"FromQuery": "mcp-server-asana::asana_update_task.task_id"}, "name": {"FromQuery": "mcp-server-asana::asana_update_task.name"}, "notes": {"FromQuery": "mcp-server-asana::asana_update_task.notes"}, "due_on": {"FromQuery": "mcp-server-asana::asana_update_task.due_on"}, "completed": {"FromQuery": "mcp-server-asana::asana_update_task.completed"}}', '[]');

-- asana_get_project: project_id is required
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('mcp-server-asana', 'asana_get_project', 'mcp-server-asana::asana_get_project', '["project_id"]',
'{"project_id": {"FromQuery": "mcp-server-asana::asana_get_project.project_id"}, "opt_fields": {"FromQuery": "mcp-server-asana::asana_get_project.opt_fields"}}', '[]');

COMMIT;
SQL

# 4. Generate/append parameter extractors to TOML file (REQUIRED for parameter extraction)
# ⚠️ CRITICAL: OI OS loads parameter_extractors.toml.default, not parameter_extractors.toml
# Add patterns to parameter_extractors.toml.default in project root
cat >> parameter_extractors.toml.default << 'ASANA_EXTRACTORS'

# ============================================================================
# ASANA MCP SERVER EXTRACTION PATTERNS
# ============================================================================

# Workspace GID - Extract Asana workspace GID (numeric string)
"workspace" = "regex:\\b\\d{13,}\\b"
"mcp-server-asana::asana_search_tasks.workspace" = "regex:(?:workspace|in)\\s+(\\d{13,})|(\\b\\d{13,}\\b)"

# Project ID/GID - Extract Asana project GID (numeric string)
"project_id" = "regex:\\b\\d{13,}\\b"
"mcp-server-asana::asana_create_task.project_id" = "regex:(?:project|in)\\s+(\\d{13,})|(\\b\\d{13,}\\b)"
"mcp-server-asana::asana_get_project.project_id" = "regex:(?:project|id)\\s+(\\d{13,})|(\\b\\d{13,}\\b)"

# Task ID/GID - Extract Asana task GID (numeric string)
"task_id" = "regex:\\b\\d{13,}\\b"
"mcp-server-asana::asana_get_task.task_id" = "regex:(?:task|id)\\s+(\\d{13,})|(\\b\\d{13,}\\b)"
"mcp-server-asana::asana_update_task.task_id" = "regex:(?:task|id)\\s+(\\d{13,})|(\\b\\d{13,}\\b)"

# Task name - Extract task name from query
"name" = "keyword:after_name"
"mcp-server-asana::asana_create_task.name" = "transform:regex:(?:create|add|new)\\s+task(?:\\s+in|\\s+project)?\\s*(?:\\d{13,})?\\s+(.+?)(?:\\s+with|\\s+due|\\s+assign|$)|trim"

# Project name pattern - Extract project name search pattern
"name_pattern" = "remove:search,find,list,projects,asana"
"mcp-server-asana::asana_search_projects.name_pattern" = "remove:search,find,list,projects,asana"

# Task notes/description - Extract task description
"notes" = "keyword:after_notes"
"mcp-server-asana::asana_create_task.notes" = "transform:regex:(?:with|description|notes?|desc)\\s+(.+)$|trim"

# Due date - Extract date in YYYY-MM-DD format
"due_on" = "regex:(\\d{4}-\\d{2}-\\d{2})|(?:due|by)\\s+(\\d{4}-\\d{2}-\\d{2})"
"mcp-server-asana::asana_create_task.due_on" = "regex:(?:due|by)\\s+(\\d{4}-\\d{2}-\\d{2})|(\\d{4}-\\d{2}-\\d{2})"

# Assignee - Extract assignee (can be "me" or user ID)
"assignee" = "regex:(?:assign|to)\\s+(me|\\d{13,})|(me|\\d{13,})"
"mcp-server-asana::asana_create_task.assignee" = "regex:(?:assign|to)\\s+(me|\\d{13,})|(me|\\d{13,})"

# Search text - Extract search query text
"text" = "remove:search,find,tasks,asana"
"mcp-server-asana::asana_search_tasks.text" = "remove:search,find,tasks,asana,workspace"

# Completed status - Extract boolean for completed tasks
"completed" = "conditional:if_contains:completed|then:default:true|else:if_contains:incomplete|then:default:false|else:default:"
"mcp-server-asana::asana_search_tasks.completed" = "conditional:if_contains:completed|then:default:true|else:if_contains:incomplete|then:default:false|else:default:"

# Projects filter - Extract project IDs for filtering
"projects_any" = "regex:(?:projects?|in)\\s+([\\d,]+)"
"mcp-server-asana::asana_search_tasks.projects_any" = "regex:(?:projects?|in)\\s+([\\d,]+)"

# Assignee filter - Extract user IDs for filtering
"assignee_any" = "regex:(?:assignee|assigned|to)\\s+([\\d,]+|me)"
"mcp-server-asana::asana_search_tasks.assignee_any" = "regex:(?:assignee|assigned|to)\\s+([\\d,]+|me)"

# Optional fields
"opt_fields" = "regex:(?:fields?|include)\\s+([\\w,]+)"
ASANA_EXTRACTORS
```

**Important Notes:**
- **`.env` file auto-loaded**: `brain-trust4 connect` automatically finds and loads `.env` from project root
- **For AI Agents**: Prefer direct calls (`./brain-trust4 call`) over natural language commands for reliability
- **For End Users**: Natural language commands (`./oi "asana create task..."`) provide better UX but may timeout
- **Direct Calls**: Use `./brain-trust4 call mcp-server-asana tool-name '{"param": "value"}'` to bypass intent mapping and parameter extraction

---

## Prerequisites

### Required Software

- **Node.js** (v18+ recommended)
- **npm** (comes with Node.js)
- **OI OS / Brain Trust 4** installed and running
- **Asana account** with API access

### Required Asana Access Token

You need an Asana Personal Access Token:

1. **Get Asana Access Token**:
   - Visit: https://app.asana.com/0/my-apps
   - Create a new personal access token
   - See: https://developers.asana.com/docs/personal-access-token

---

## Full Installation Steps

### Step 1: Install the Server

```bash
# From your OI OS project root
./oi install https://github.com/OI-OS/mcp-server-asana.git
```

**Alternative (if already installed):**
```bash
cd MCP-servers/mcp-server-asana
git pull  # Update if needed
npm install  # Update dependencies if needed
```

---

## Configuring Authentication

### Step 1: Get Asana Access Token

Extract your Asana personal access token from: https://app.asana.com/0/my-apps

### Step 2: Configure Environment Variables

Add to your OI OS project root `.env` file:

```bash
# Required: Asana authentication token
ASANA_ACCESS_TOKEN=your-asana-access-token-here

# Optional: Enable read-only mode (disables all write operations)
# READ_ONLY_MODE=true
```

**Important Security Notes:**
- Never commit `.env` files to version control
- Tokens provide access to your Asana workspace
- Use read-only mode for testing if needed

---

## Connecting to OI OS

### Step 1: Verify Installation

```bash
cd MCP-servers/mcp-server-asana
ls -la dist/index.js
# Ensure the built file exists
```

### Step 2: Connect the Server

From your OI OS project root:

```bash
./brain-trust4 connect mcp-server-asana node -- "$(pwd)/MCP-servers/mcp-server-asana/dist/index.js"
```

**Note:** The `brain-trust4 connect` command automatically finds and loads `.env` file from the project root. Environment variables (`ASANA_ACCESS_TOKEN`, etc.) will be automatically available to the server process.

### Step 3: Verify Connection

```bash
./oi list
# Should show "mcp-server-asana" in the server list

./oi status mcp-server-asana
# Should show server status and capabilities

# Test with direct call (most reliable method)
./brain-trust4 call mcp-server-asana asana_list_workspaces '{}'
```

---

## Configuring Parameter Extractors

Parameter extractors allow OI OS to automatically extract tool parameters from natural language queries.

**⚠️ CRITICAL: File Loading Priority**

OI OS loads parameter extractors from `parameter_extractors.toml.default` in the project root, **not** from `parameter_extractors.toml`. The system prioritizes the `.default` file, so patterns must be added there for them to be loaded.

**NOTE (Backup Option)**: For direct tool calls bypassing intent mapping and parameter extraction, use: `./brain-trust4 call mcp-server-asana tool-name '{"param": "value"}'`

### Location

Add to: `parameter_extractors.toml.default` in your OI OS project root (this is the file that's actually loaded).

### Asana Parameter Extractors

See the [AI Agent Quick Installation](#ai-agent-quick-installation) section for the complete extractor patterns.

---

## Creating Parameter Rules

**⚠️ CRITICAL: Parameter rules must be created in the database for parameter extraction to work.**

Parameter rules define which fields are required and how to extract them from natural language queries. The OI OS parameter engine **only extracts required fields** - optional fields are skipped even if extractors exist in `parameter_extractors.toml.default`.

### Why Parameter Rules Are Needed

- **Required fields are extracted**: The parameter engine processes required fields and invokes their extractors
- **Optional fields are skipped**: Optional fields are ignored during parameter extraction, even if extractors exist
- **Database-driven**: Parameter rules are stored in the `parameter_rules` table in `brain-trust4.db`

### Creating Parameter Rules

See the [AI Agent Quick Installation](#ai-agent-quick-installation) section for the complete SQL transaction to create all parameter rules.

### Verifying Parameter Rules

```bash
# List all Asana parameter rules
sqlite3 brain-trust4.db "SELECT tool_signature, required_fields FROM parameter_rules WHERE server_name = 'mcp-server-asana';"

# Check specific tool rule
sqlite3 brain-trust4.db "SELECT * FROM parameter_rules WHERE tool_signature = 'mcp-server-asana::asana_create_task';"
```

---

## Creating Intent Mappings

Intent mappings connect natural language queries to specific Asana MCP tools. Create them using SQL INSERT statements.

### Database Location

```bash
sqlite3 brain-trust4.db
```

### All Asana MCP Server Intent Mappings

See the [AI Agent Quick Installation](#ai-agent-quick-installation) section for the complete SQL statement to create all intent mappings.

### Verifying Intent Mappings

```bash
# List all Asana intent mappings
sqlite3 brain-trust4.db "SELECT * FROM intent_mappings WHERE server_name = 'mcp-server-asana' ORDER BY priority DESC;"

# Or use OI command
./oi intent list | grep asana
```

---

## End User Setup

### Quick Start for End Users

1. **Install Prerequisites**
   ```bash
   # Install Node.js (if not installed)
   brew install node  # macOS
   # or visit: https://nodejs.org/
   ```

2. **Install Server**
   ```bash
   ./oi install https://github.com/OI-OS/mcp-server-asana.git
   ```

3. **Get Asana Access Token**
   - Visit: https://app.asana.com/0/my-apps
   - Create a personal access token

4. **Configure Environment**
   ```bash
   # Create .env file in project root
   ASANA_ACCESS_TOKEN=your-asana-access-token-here
   ```

5. **Connect Server**
   ```bash
   ./brain-trust4 connect mcp-server-asana node -- "$(pwd)/MCP-servers/mcp-server-asana/dist/index.js"
   ```

---

## Verification & Testing

### Test Server Connection

```bash
# List all servers
./oi list

# Check Asana server status
./oi status mcp-server-asana

# Test tool discovery
./brain-trust4 tools mcp-server-asana
```

### Test Intent Mappings

```bash
# Test listing workspaces
./oi "asana list workspaces"

# Test searching projects
./oi "asana search projects Sprint"

# Test searching tasks
./oi "asana search tasks in workspace 1211711832059222"
```

### Direct Tool Calls (Recommended for AI Agents)

**NOTE**: For direct tool calls bypassing intent mapping and parameter extraction:

```bash
# List workspaces
./brain-trust4 call mcp-server-asana asana_list_workspaces '{}'

# Create task
./brain-trust4 call mcp-server-asana asana_create_task '{"project_id": "1234567890123", "name": "New task"}'

# Get task
./brain-trust4 call mcp-server-asana asana_get_task '{"task_id": "1234567890123"}'
```

---

## Troubleshooting

### Authentication Issues

**"Missing ASANA_ACCESS_TOKEN" Error**
- Verify `.env` file exists in project root
- Check token format (should be a long string)
- Ensure no extra spaces or quotes around token value
- Restart OI OS after adding token

**"Invalid token" Error**
- Token may have expired (create new token)
- Verify token is correct (no typos)
- Check if workspace restrictions apply

### Tool Execution Issues

**"Tool not found" Error**
- Verify server connection: `./oi status mcp-server-asana`
- Restart server connection

**Parameter Extraction Fails**
- Verify parameter extractors are in `parameter_extractors.toml.default`
- Check parameter rules exist in database
- Verify required fields are correctly marked in parameter rules
- **NOTE (Backup)**: Use direct calls if needed: `./brain-trust4 call mcp-server-asana tool-name '{"param": "value"}'`

### Connection Issues

**Server Won't Connect**
- Verify `dist/index.js` exists: `ls -la MCP-servers/mcp-server-asana/dist/index.js`
- Check Node.js is installed: `node --version`
- Verify `.env` file exists in project root with correct token
- Check that `brain-trust4` is loading `.env` (should happen automatically)
- Check logs for error messages

---

## Available Tools Reference

### Workspace Management
- `asana_list_workspaces()` - List all available workspaces
  - **Optional**: `opt_fields`

### Project Management
- `asana_search_projects(name_pattern, workspace, archived, opt_fields)` - Search for projects
  - **Required**: `name_pattern`
  - **Optional**: `workspace`, `archived`, `opt_fields`

- `asana_get_project(project_id, opt_fields)` - Get project details
  - **Required**: `project_id`
  - **Optional**: `opt_fields`

### Task Management
- `asana_search_tasks(workspace, filters...)` - Search tasks with advanced filtering
  - **Required**: `workspace`
  - **Optional**: `text`, `completed`, `projects_any`, `assignee_any`, and many more filters

- `asana_get_task(task_id, opt_fields)` - Get task details
  - **Required**: `task_id`
  - **Optional**: `opt_fields`

- `asana_create_task(project_id, name, notes, due_on, assignee, ...)` - Create new task
  - **Required**: `project_id`, `name`
  - **Optional**: `notes`, `due_on`, `assignee`, `followers`, `parent`, etc.

- `asana_update_task(task_id, name, notes, due_on, completed, ...)` - Update existing task
  - **Required**: `task_id`
  - **Optional**: `name`, `notes`, `due_on`, `completed`, etc.

**Note:** See full tool list with `./brain-trust4 tools mcp-server-asana`

---

## Name Resolution System

OI OS includes a universal name→ID mapping system that enables working with human-friendly names instead of cryptic IDs. This system is documented in the `ID MAPPING/` folder of your OI OS installation.

### Quick Reference

**Location:** `ID MAPPING/` folder in OI OS root directory

**Key Files:**
- `brain.json` - Central mapping file (workspaces, projects, etc.)
- `sync-all-mappings.sh` - Sync all MCP servers
- `sync-name-cache.sh` - Sync individual server
- `resolve-params.sh` - Resolve names to IDs + auto-learn

**Usage:**
```bash
# Sync Asana mappings
cd "ID MAPPING"
./sync-name-cache.sh mcp-server-asana

# Use names instead of IDs
./resolve-params.sh mcp-server-asana asana_create_task '{"project_id": "OI OS", "name": "task"}'
```

**Features:**
- ✅ Auto-learns from API responses
- ✅ Auto-syncs when mappings are missing
- ✅ Works universally for all MCP servers
- ✅ Simple JSON structure (easy to edit)

For complete documentation, see `ID MAPPING/README.md` in your OI OS installation.

---

## Additional Resources

- **Asana MCP Server Repository:** https://github.com/OI-OS/mcp-server-asana
- **Asana API Documentation:** https://developers.asana.com/docs
- **Personal Access Token Guide:** https://developers.asana.com/docs/personal-access-token
- **OI OS Documentation:** See `docs/` directory in your OI OS installation
- **MCP Protocol Specification:** https://modelcontextprotocol.io/
- **Name Resolution System:** See `ID MAPPING/` folder in OI OS installation

---

## Support

For issues specific to:
- **Asana MCP Server:** Open an issue at https://github.com/OI-OS/mcp-server-asana
- **OI OS Integration:** Check OI OS documentation or repository
- **General MCP Issues:** See MCP documentation at https://modelcontextprotocol.io/

---

**Last Updated:** 2025-11-08  
**Compatible With:** OI OS / Brain Trust 4, Claude Desktop, Cursor  
**Server Version:** Latest from OI-OS/mcp-server-asana

