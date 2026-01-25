# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is a Debian packaging repository for **zabbix-mcp-server**, a Model Context Protocol (MCP) server that provides comprehensive Zabbix monitoring API integration. The server enables AI assistants and tools to interact with Zabbix systems through MCP.

**Key technologies:**
- Python 3.10+ with FastMCP framework
- python-zabbix-utils for Zabbix API integration
- Debian packaging using pybuild and setuptools
- Environment-based configuration with python-dotenv

## Common Development Commands

### Python Development
```bash
# Install dependencies using uv package manager
uv sync

# Run the server directly
uv run python src/zabbix_mcp_server.py

# Run with startup script (recommended - includes validation)
uv run python scripts/start_server.py

# Run test suite
uv run python scripts/test_server.py

# Enable debug logging
DEBUG=1 uv run python scripts/start_server.py
```

### Debian Package Building
```bash
# Build binary package (no source package, no signing)
dpkg-buildpackage -us -uc -b

# Check package contents
dpkg -c ../zabbix-mcp-server_*.deb

# Check package metadata
dpkg -I ../zabbix-mcp-server_*.deb

# Run lintian quality checks
lintian ../zabbix-mcp-server_*.deb
```

### Configuration
```bash
# Set up environment for testing
cp config/.env.example .env
# Edit .env with your Zabbix server credentials

# Required environment variables:
# - ZABBIX_URL: Zabbix server API endpoint
# - ZABBIX_TOKEN: API token (or ZABBIX_USER/ZABBIX_PASSWORD)
# - READ_ONLY: Set to "true" for read-only mode (default: true)
```

## Architecture

### Core Design Pattern

The server is a **single-file MCP implementation** (`src/zabbix_mcp_server.py`, ~1561 lines) that follows a decorator-based tool registration pattern:

1. **FastMCP initialization**: Creates MCP server instance at module level
2. **Tool registration**: Each Zabbix API method is exposed as an `@mcp.tool()` decorated function
3. **Client management**: Singleton ZabbixAPI client with lazy initialization and environment-based authentication
4. **Read-only enforcement**: All write operations validate against `READ_ONLY` environment variable via `validate_read_only()`
5. **Response formatting**: All tools return JSON-formatted strings via `format_response()`

### Key Architectural Decisions

**Why single file?** The entire server logic lives in one file because:
- Each tool function follows identical patterns (get client → build params → call API → format response)
- Minimal abstraction layers - tools map directly to Zabbix API endpoints
- Configuration is purely environment-based (no complex config parsing)

**Authentication flow:**
1. `get_zabbix_client()` called on first tool invocation
2. Client authenticates using either API token (preferred) or username/password
3. Global `zabbix_api` variable stores authenticated client for reuse
4. SSL verification controlled by `VERIFY_SSL` environment variable

**Read-only mode:**
- Default is READ_ONLY=true for safety
- All create/update/delete operations call `validate_read_only()` first
- Raises ValueError if write attempted in read-only mode
- GET operations (host_get, item_get, etc.) bypass this check

### Transport Architecture

The server supports two transport modes (configured via `ZABBIX_MCP_TRANSPORT`):

1. **STDIO (default)**: For MCP clients like Claude Desktop
   - Uses stdin/stdout for communication
   - No additional configuration needed

2. **streamable-http**: For web integrations
   - Requires `AUTH_TYPE=no-auth` (enforced at startup)
   - Configurable host/port via `ZABBIX_MCP_HOST` and `ZABBIX_MCP_PORT`
   - Supports stateless mode via `ZABBIX_MCP_STATELESS_HTTP`

### Tool Categories (11 major groups)

All tools follow the pattern: `{category}_{operation}(params) -> str`

- **Host Management**: host_get, host_create, host_update, host_delete
- **Host Groups**: hostgroup_get, hostgroup_create, hostgroup_update, hostgroup_delete
- **Items**: item_get, item_create, item_update, item_delete
- **Triggers**: trigger_get, trigger_create, trigger_update, trigger_delete
- **Templates**: template_get, template_create, template_update, template_delete
- **Problems & Events**: problem_get, event_get, event_acknowledge
- **History & Trends**: history_get, trend_get
- **Users**: user_get, user_create, user_update, user_delete
- **Proxies**: proxy_get, proxy_create, proxy_update, proxy_delete
- **Maintenance**: maintenance_get, maintenance_create, maintenance_update, maintenance_delete
- **Configuration**: configuration_export, configuration_import, apiinfo_version, etc.

## Debian Packaging Structure

### Critical Debian Files

- `debian/control`: Package metadata, dependencies, and description
  - Architecture is `all` (pure Python, no compiled code)
  - Dependencies include `python3-fastmcp`, `python3-zabbix-utils`, `python3-dotenv`
  - Uses `${python3:Depends}` for automatic Python dependency resolution

- `debian/rules`: Build instructions using `dh` with pybuild
  - Minimal file: delegates to `dh $@ --with python3 --buildsystem=pybuild`
  - pybuild automatically handles Python package installation via pyproject.toml

- `debian/py3dist-overrides`: Maps Python package names to Debian package names
  - Required because upstream PyPI names don't match Debian package names
  - Format: `fastmcp python3-fastmcp` (one mapping per line)

- `debian/changelog`: Version history in Debian format
  - Version format: `{upstream_version}-{debian_revision}` (e.g., 1.1.0-1)
  - Distribution: `unstable` for standard packages

### Build System Integration

The package uses **pybuild with pyproject.toml**:
- `pyproject.toml` defines project metadata and entry points
- Entry point `zabbix-mcp` → `src.zabbix_mcp_server:main` creates `/usr/bin/zabbix-mcp` executable
- pybuild automatically installs Python modules to `/usr/lib/python3/dist-packages/`
- No need for manual `debian/install` file - pybuild handles everything

### Packaging Workflow

When you modify the package:
1. Update `debian/changelog` using `dch` or manually (maintain Debian format)
2. If adding Python dependencies: update `debian/control` AND `debian/py3dist-overrides`
3. Rebuild: `dpkg-buildpackage -us -uc -b`
4. Verify: Check `dpkg -I` output shows correct dependencies

## Environment Configuration

### Authentication Methods (choose one)

**Method 1: API Token (recommended)**
```bash
ZABBIX_URL=https://zabbix.example.com
ZABBIX_TOKEN=your_token_here
```

**Method 2: Username/Password**
```bash
ZABBIX_URL=https://zabbix.example.com
ZABBIX_USER=admin
ZABBIX_PASSWORD=password
```

### Security Modes

**Read-only mode** (default, safe for monitoring):
```bash
READ_ONLY=true  # Blocks all create/update/delete operations
```

**Full mode** (enables write operations):
```bash
READ_ONLY=false  # or "0" or "no"
```

### SSL Verification

```bash
VERIFY_SSL=true   # Default: verify certificates
VERIFY_SSL=false  # Disable for self-signed certs (testing only)
```

## Testing Strategy

The test suite (`scripts/test_server.py`) validates:
- Authentication mechanisms (token and username/password)
- Read-only mode enforcement
- Transport configuration validation
- Basic tool functionality

**No unit tests exist** - testing requires a real Zabbix server instance. The test script performs integration testing against a live Zabbix API.

## MCP Client Integration

For Claude Desktop integration, see `MCP_SETUP.md`. Key points:

- Use `uv run --directory /path/to/repo python src/zabbix_mcp_server.py`
- Configure environment variables in MCP client config
- Recommended: Use startup script for better error handling
- Can use `.env` file instead of MCP config for credentials

## Docker Support

The repository includes Docker support (see `Dockerfile` and `docker-compose.yml`):
- Uses Python 3.12 slim base image
- Installs uv package manager
- Configures environment variables
- Runs with `uv run python src/zabbix_mcp_server.py`

Build and run:
```bash
docker build -t zabbix-mcp-server .
# or
docker compose up -d
```
