# EasyProject MCP Server

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/rust-2024edition-orange.svg)](https://www.rust-lang.org)

A [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server for the [EasyProject](https://www.easyproject.com) API. It exposes 50+ tools that let AI assistants manage projects, issues, users, time entries, milestones, attachments, and more directly through the MCP stdio transport.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Client Setup](#client-setup)
  - [Claude Code](#claude-code)
  - [OpenCode](#opencode)
  - [Other MCP Clients](#other-mcp-clients)
- [Available Tools](#available-tools)
- [Usage Examples](#usage-examples)
- [Shell Completions](#shell-completions)
- [Development](#development)
- [Troubleshooting](#troubleshooting)
- [License](#license)
- [Contributing](#contributing)

## Features

- **Full MCP stdio transport** — communicates with any MCP-compatible client over standard input/output.
- **Project management** — create, update, archive, close, favorite, and delete projects; manage memberships and documents.
- **Issue tracking** — create, update, assign, complete, and delete issues; manage watchers, relations, and journals.
- **User management** — list, create, and update users; query workload aggregations.
- **Time tracking** — log, update, and delete time entries.
- **Milestones** — manage project versions and milestones.
- **Attachments** — upload base64 or local files and attach them to issues or projects.
- **Search & reporting** — cross-entity search, project reports, and dashboard data.
- **Workspace sandboxing** — restrict local file uploads to validated directory roots.
- **Tool allow-listing** — expose only a subset of tools via configuration.
- **Markdown-to-HTML** — Markdown in tool arguments is automatically rendered to safe HTML for EasyProject's HTML-backed fields (descriptions, notes, comments, etc.).
- **Shell completion generation** — built-in completions for `bash`, `zsh`, `fish`, `powershell`, and `elvish`.

## Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) 1.85+ (the project uses the 2024 edition)
- An active EasyProject instance
- An EasyProject API key (generated from **My account → API access key**)

## Installation

### Build from source

```bash
git clone https://github.com/bighub-ai/easyproject-mcp-server.git
cd easyproject-mcp-server
cargo build --release
```

The binary is produced at:

```
target/release/easyproject-mcp-server
```

### Install via Cargo from Git

```bash
cargo install --git https://github.com/bighub-ai/easyproject-mcp-server
```

### Pre-built binaries

Pre-built binaries for macOS, Linux, and Windows are available on the [GitHub Releases](https://github.com/bighub-ai/easyproject-mcp-server/releases) page (distributed via `cargo-dist`).

### Homebrew (macOS/Linux)

```bash
brew tap bighub-ai/homebrew-tap
brew install easyproject-mcp-server
```

### Using `just`

The repository includes a `justfile` with common tasks:

| Recipe | Description |
|--------|-------------|
| `just build` | Build the debug binary |
| `just build-release` | Build the release binary |
| `just test` | Run tests (excludes live-server tests) |
| `just test --live-server` | Run all tests including live-server tests |
| `just setup` | Copy `.env.example` to `.env` |
| `just fetch-schema` | Download the EasyProject OpenAPI schema |
| `just install-hooks` | Install pre-commit hooks |
| `just clean` | Remove generated artefacts and run `cargo clean` |

## Configuration

Configuration is loaded from multiple sources, in order of increasing priority:

1. **Built-in defaults**
2. **Standard per-user config file** (auto-discovered from the platform-specific config directory)
3. **`config.toml`** in the working directory
4. **Environment variables** (`EASYPROJECT_*`)
5. **CLI arguments**

Scalars are overridden by higher-priority sources. Arrays (e.g. `workspace_roots`) are **merged** across sources.

Use `--config <PATH>` to bypass auto-discovery and load a specific config file.

### Generating a starter config file

Run the built-in init command to generate a commented `config.toml`:

```bash
easyproject-mcp-server config init
```

### `config.toml` example

```toml
[easyproject]
base_url = "https://your-instance.easyproject.com"
api_key = "your-api-key"

[http]
timeout_seconds = 30
# user_agent = "EasyProject-MCP-Server/0.1.0"

# Local directories the server is allowed to read from for uploads.
# Invalid or non-existent paths are skipped with a warning.
workspace_roots = ["/home/user/projects", "./data"]

# Tool allow-list. Leave empty (the default) to expose all tools.
[tools]
enabled = []

# Markdown rendering options for HTML-backed fields.
# When an MCP tool argument contains Markdown, it is automatically rendered
# to HTML before being sent to the EasyProject API.
[markdown]
strikethrough = true
table = true
autolink = true
tasklist = true
tasklist_classes = true
gfm_quirks = true
# footnotes = false
# description_lists = false
# alerts = false
# hardbreaks = false
# github_pre_lang = false
```

### Environment variables

| Variable | Description | Required |
|----------|-------------|----------|
| `EASYPROJECT_BASE_URL` | Root URL of your EasyProject instance | Yes* |
| `EASYPROJECT_API_KEY` | API key for authentication | Yes* |

*The server will start without them, but any tool that calls the EasyProject API will fail at runtime.

### CLI arguments

You can override configuration at runtime:

| Flag | Description |
|------|-------------|
| `--config <PATH>` | Use a specific config file (disables auto-discovery) |
| `--base-url <URL>` | Override the EasyProject base URL |
| `--api-key <KEY>` | Override the API key |
| `--workspace-root <PATH>` | Add a workspace root (repeatable) |

## Client Setup

### Claude Code

Create a **project-scoped** `.mcp.json` in your project root, or a **user-scoped** `~/.claude.json`.

```json
{
  "mcpServers": {
    "easyproject": {
      "command": "easyproject-mcp-server",
      "args": [],
      "env": {
        "EASYPROJECT_BASE_URL": "https://your-instance.easyproject.com",
        "EASYPROJECT_API_KEY": "your-api-key"
      }
    }
  }
}
```

If the binary is not in your `PATH`, use the absolute path:

```json
{
  "command": "/absolute/path/to/easyproject-mcp-server"
}
```

You can also add the server via the Claude Code CLI:

```bash
claude mcp add --transport stdio easyproject -- easyproject-mcp-server
```

### OpenCode

Create an `opencode.json` (or `opencode.jsonc`) in your project root:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "easyproject": {
      "type": "local",
      "command": ["easyproject-mcp-server"],
      "enabled": true,
      "environment": {
        "EASYPROJECT_BASE_URL": "https://your-instance.easyproject.com",
        "EASYPROJECT_API_KEY": "your-api-key"
      }
    }
  }
}
```

Use an absolute path if the binary is not globally available:

```json
"command": ["/absolute/path/to/easyproject-mcp-server"]
```

Verify the connection:

```bash
opencode mcp list
opencode mcp debug easyproject
```

### Other MCP Clients

Any client that supports **stdio-based MCP servers** can use the same pattern shown for Claude Code (`mcpServers` block) or OpenCode (`mcp` block). The only requirements are:

- The client launches the binary as a subprocess.
- The `EASYPROJECT_BASE_URL` and `EASYPROJECT_API_KEY` environment variables are provided (or a `config.toml` is present in the working directory).

## Available Tools

### Projects

| Tool | Description |
|------|-------------|
| `list_projects` | List all projects with optional filtering and pagination |
| `get_project` | Get a single project by ID |
| `create_project` | Create a new project |
| `update_project` | Update an existing project |
| `delete_project` | Delete a project by ID |
| `archive_project` | Archive a project by ID |
| `unarchive_project` | Unarchive a project by ID |
| `close_project` | Close a project by ID |
| `reopen_project` | Reopen a project by ID |
| `favorite_project` | Favorite a project by ID |
| `unfavorite_project` | Unfavorite a project by ID |
| `list_project_memberships` | List memberships for a project |
| `create_project_membership` | Create a project membership |
| `update_project_membership` | Update a project membership |
| `delete_project_membership` | Delete a project membership |
| `list_project_documents` | List documents for a project |
| `create_project_document` | Create a project document |

### Issues

| Tool | Description |
|------|-------------|
| `list_issues` | List issues with optional filtering |
| `get_issue` | Get a single issue by ID |
| `create_issue` | Create a new issue |
| `update_issue` | Update an existing issue |
| `assign_issue` | Assign an issue to a user |
| `complete_issue` | Mark an issue as 100% done |
| `update_journal` | Update a journal entry |
| `list_issue_journals` | List journals for an issue |
| `delete_issue` | Delete an issue by ID |
| `list_issue_relations` | List relations for an issue |
| `create_issue_relation` | Create a relation between issues |
| `delete_issue_relation` | Delete a relation between issues |
| `add_issue_watcher` | Add a watcher to an issue |
| `remove_issue_watcher` | Remove a watcher from an issue |

### Users

| Tool | Description |
|------|-------------|
| `list_users` | List users |
| `get_user` | Get a single user by ID, or omit ID to get the current authenticated user |
| `get_user_workload` | Get aggregated workload for a user |
| `create_user` | Create a new user |
| `update_user` | Update an existing user |

### Time Entries

| Tool | Description |
|------|-------------|
| `list_time_entries` | List time entries |
| `get_time_entry` | Get a time entry by ID |
| `create_time_entry` | Create a time entry |
| `update_time_entry` | Update a time entry |
| `log_time` | Log time with sensible defaults |
| `delete_time_entry` | Delete a time entry by ID |

### Versions & Milestones

| Tool | Description |
|------|-------------|
| `list_milestones` | List versions/milestones |
| `get_milestone` | Get a single version/milestone by ID |
| `create_milestone` | Create a new version/milestone |
| `update_milestone` | Update a version/milestone |
| `delete_milestone` | Delete a version/milestone by ID |

### Attachments & Uploads

| Tool | Description |
|------|-------------|
| `get_attachment` | Get attachment metadata by ID |
| `upload_attachment` | Upload a file attachment (base64) |
| `attach_attachment` | Attach a file to an entity (issue or project) |
| `upload_local_attachment` | Read a local file from the workspace roots and upload it |
| `delete_attachment` | Delete an attachment by ID |

### Search, Reports & Dashboard

| Tool | Description |
|------|-------------|
| `search` | Search across entities |
| `generate_project_report` | Generate aggregated project report |
| `get_dashboard_data` | Get dashboard summary data |
| `reindex_search` | Trigger search reindex |

### Enumerations & Metadata

| Tool | Description |
|------|-------------|
| `list_custom_fields` | List custom fields |
| `list_time_entry_activities` | List time entry activities |
| `list_trackers` | List trackers from projects |
| `list_issue_categories` | List issue categories for a project |
| `list_issue_statuses` | List unique issue statuses |
| `list_priorities` | List unique issue priorities |
| `get_issue_enumerations` | Get enumerations derived from issue data |

## Usage Examples

### List projects

```json
{
  "method": "tools/call",
  "params": {
    "name": "list_projects",
    "arguments": {
      "limit": 10
    }
  }
}
```

### Create an issue

```json
{
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "project_id": 1,
      "tracker_id": 1,
      "subject": "Investigate performance regression",
      "description": "The API response time has increased significantly.",
      "assigned_to_id": 5,
      "priority_id": 2
    }
  }
}
```

### Assign an issue

```json
{
  "method": "tools/call",
  "params": {
    "name": "assign_issue",
    "arguments": {
      "id": 123,
      "assigned_to_id": 5
    }
  }
}
```

### Mark an issue as complete

```json
{
  "method": "tools/call",
  "params": {
    "name": "complete_issue",
    "arguments": {
      "id": 123
    }
  }
}
```

### Log time

```json
{
  "method": "tools/call",
  "params": {
    "name": "log_time",
    "arguments": {
      "project_id": 1,
      "hours": 2.5,
      "comments": "Implemented the new feature"
    }
  }
}
```

### Upload a local file

```json
{
  "method": "tools/call",
  "params": {
    "name": "upload_local_attachment",
    "arguments": {
      "file_path": "specs/requirements.pdf",
      "entity_type": "issue",
      "entity_id": 123
    }
  }
}
```

## Shell Completions

The binary supports automatic shell completion script generation via the `COMPLETE` environment variable.

### Manual installation

```bash
# Bash
COMPLETE=bash easyproject-mcp-server > /etc/bash_completion.d/easyproject-mcp-server

# Zsh
COMPLETE=zsh easyproject-mcp-server > /usr/local/share/zsh/site-functions/_easyproject-mcp-server

# Fish
COMPLETE=fish easyproject-mcp-server > ~/.config/fish/completions/easyproject-mcp-server.fish

# PowerShell
COMPLETE=powershell easyproject-mcp-server | Out-File -Encoding utf8 $PROFILE\..\Completions\easyproject-mcp-server.ps1

# Elvish
COMPLETE=elvish easyproject-mcp-server > ~/.config/elvish/lib/completions/easyproject-mcp-server.elv
```

### Homebrew

When installing via `brew install easyproject-mcp-server` from the `bighub-ai/homebrew-tap`, completions are installed automatically.

## Development

### Project structure

```
src/
├── main.rs              # Entry point and CLI parsing
├── lib.rs               # Library exports
├── api/                 # EasyProject API client and generated models
├── config/              # Configuration loading and structs
├── defaults/            # Dynamic default value resolution
├── markdown.rs          # Markdown-to-HTML rendering (comrak + ammonia)
├── server/              # MCP server runtime (stdio transport)
├── tools/               # MCP tool implementations
│   ├── mod.rs           # Tool router and handlers
│   ├── parameters.rs    # Request parameter structs
│   ├── status.rs        # Status response helpers
│   ├── helpers.rs       # Shared formatting helpers
│   ├── errors.rs        # Error mapping
│   └── json_map.rs      # JSON object utilities
└── workspace_root.rs    # Workspace root validation
```

### Running tests

```bash
# Standard tests (excludes live API tests)
cargo test

# Include live-server integration tests
cargo test -- --include-ignored --skip "api::Client"
```

Or using `just`:

```bash
just test
just test --live-server
```

### Debug output

The server writes structured logs to **stderr**. Because MCP stdio transport uses stdout exclusively for JSON-RPC messages, all diagnostic output is safely routed to stderr.

## Troubleshooting

### "Unauthorized" or 401 errors

- Verify that `EASYPROJECT_API_KEY` is correct.
- Ensure the API key has not expired (regenerate it from **My account → API access key**).
- Confirm the user associated with the key has sufficient permissions.

### "Connection refused" or timeout errors

- Verify that `EASYPROJECT_BASE_URL` is reachable from the machine running the server.
- Ensure the URL includes the scheme (`https://`) and has no trailing path.
- Check firewall or proxy settings.

### Tools not appearing in the client

- Check that the server started without errors (inspect stderr output).
- Verify the `EASYPROJECT_BASE_URL` and `EASYPROJECT_API_KEY` are set.
- If you configured `[tools] enabled = [...]`, ensure the desired tool names are spelled exactly as listed above.

### Local file upload fails

- Ensure the file path is within one of the configured `workspace_roots`.
- Verify the path exists and is readable by the server process.
- Check stderr for warnings about invalid or skipped workspace roots.

## License

This project is licensed under the [MIT License](LICENSE).

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request
