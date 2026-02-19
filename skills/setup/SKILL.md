---
name: setup
description: Install cclogviewer MCP server and create reflection-setup.json for this project
allowed-tools: [Bash, Read, Write, AskUserQuestion,
                mcp__cclogviewer__list_projects]
---

# Reflection Plugin Setup

Install the cclogviewer MCP server and configure this project for reflection.

## Part A: Install cclogviewer MCP

### Step 1: Check if already installed

```bash
which cclogviewer-mcp
```

If found, skip to Step 4.

### Step 2: Install cclogviewer-mcp

Try these methods in order:

**Option A: Download pre-built binary**
```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then ARCH="amd64"; fi
if [ "$ARCH" = "aarch64" ]; then ARCH="arm64"; fi

mkdir -p ~/.local/bin
curl -L "https://github.com/SeleznovIvan/cclogviewer/releases/latest/download/cclogviewer-mcp-${OS}-${ARCH}" -o ~/.local/bin/cclogviewer-mcp
chmod +x ~/.local/bin/cclogviewer-mcp
```

**Option B: Go install (requires Go 1.21+)**
```bash
go install github.com/SeleznovIvan/cclogviewer/cmd/cclogviewer-mcp@latest
```

**Option C: Build from source**
```bash
git clone https://github.com/SeleznovIvan/cclogviewer.git /tmp/cclogviewer
cd /tmp/cclogviewer && make build-mcp
cp cclogviewer-mcp ~/.local/bin/
```

### Step 3: Verify installation

```bash
cclogviewer-mcp --version
```

### Step 4: Register with Claude Code

```bash
claude mcp add cclogviewer cclogviewer-mcp
```

## Part B: Create reflection-setup.json

### Step 5: Discover project name

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
```

Call `mcp__cclogviewer__list_projects()` to discover available projects.
Match the current directory to a project. If ambiguous, ask user to select with AskUserQuestion.

### Step 6: Create reflection-setup.json

```bash
mkdir -p "$PROJECT_ROOT/.claude"
```

Write `$PROJECT_ROOT/.claude/reflection-setup.json`:
```json
{
  "project_name": "discovered-project-name",
  "project_root": "/absolute/path/to/project",
  "created_at": "ISO timestamp",
  "reflection_history": []
}
```

### Step 7: Confirm setup complete

Report to user:
- cclogviewer MCP: installed/already present
- Project discovered: {project_name}
- reflection-setup.json: created at {path}
- Ready to use: `/reflect direct`, `/reflect agent <name>`, `/reflect skill <name>`
