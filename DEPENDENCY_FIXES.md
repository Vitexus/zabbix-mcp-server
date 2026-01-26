# Dependency Fixes for zabbix-mcp-server

This document describes the fixes applied to dependency packages to make `zabbix-mcp` functional.

## Fixed Packages

### 1. python3-pydocket (0.17.1-1)

**Issue**: Incompatibility with fakeredis 2.29.0
- `FakeServer` moved from `fakeredis.aioredis` to `fakeredis._server`
- `_patch_fakeredis_lua_runtime()` function uses APIs removed in fakeredis 2.29.0

**Fix**: Modified `/src/docket/_redis.py`:
```python
# Line 24-26: Updated TYPE_CHECKING import
if typing.TYPE_CHECKING:
    # FakeServer moved to fakeredis._server in fakeredis 2.29.0
    FakeServer = typing.Any  # type: ignore

# Line 257-263: Fixed runtime import and disabled incompatible patching
from fakeredis.aioredis import FakeConnection
from fakeredis._server import FakeServer  # FakeServer moved in fakeredis 2.29.0

# Apply Lua runtime patch on first use
# Disabled - API incompatibility with fakeredis 2.29.0
# _patch_fakeredis_lua_runtime()
```

**Commit**: `8cce59b - Fix fakeredis 2.29.0 compatibility`

**Repository**: `/home/vitex/Projects/Packaging/python3-docket`

---

### 2. python3-authlib (1.6.6-1)

**Issue**: Incompatibility with joserfc 1.1.0
- `BaseClaimsRegistry` was renamed to `ClaimsRegistry` in newer joserfc versions
- authlib 1.6.6 required by fastmcp >= 1.6.5

**Fix**: Modified `/authlib/oauth2/claims.py`:
```python
from joserfc.errors import InvalidClaimError
try:
    from joserfc.jwt import BaseClaimsRegistry
except ImportError:
    # BaseClaimsRegistry was renamed to ClaimsRegistry in newer joserfc
    from joserfc.rfc7519.registry import ClaimsRegistry as BaseClaimsRegistry
from joserfc.jwt import Claims
from joserfc.jwt import JWTClaimsRegistry
from joserfc.registry import Header
```

**Commit**: `86de209 - Fix joserfc compatibility`

**Repository**: `/home/vitex/Projects/Packaging/python3-authlib`

---

### 3. python3-py-key-value-shared (0.3.0-1)

**Issue**: Missing `__init__.py` files causing packages not to be discovered by setuptools

**Fix**: Added missing files:
- `src/key_value/__init__.py` - Namespace package declaration
- `src/key_value/shared/type_checking/__init__.py` - Empty module marker

Content of namespace package `__init__.py`:
```python
"""Namespace package for key_value."""
__path__ = __import__("pkgutil").extend_path(__path__, __name__)
```

**Commit**: `e1d473b - Add missing __init__.py files for namespace packages`

**Repository**: `/home/vitex/Projects/Packaging/python-py-key-value-aio/key-value/key-value-shared`

---

### 4. python3-py-key-value-aio (0.3.0-1)

**Issue**: Missing `__init__.py` file for namespace package

**Fix**: Added `src/key_value/__init__.py` with namespace package declaration:
```python
"""Namespace package for key_value."""
__path__ = __import__("pkgutil").extend_path(__path__, __name__)
```

**Commit**: `a9135b7 - Add missing __init__.py for namespace package support`

**Repository**: `/home/vitex/Projects/Packaging/python-py-key-value-aio/key-value/key-value-aio`

---

## Additional System Packages Installed

The following packages were installed from Debian repositories:

- `python3-beartype` (0.20.2-1)
- `python3-exceptiongroup` (1.2.2-1)
- `python3-joserfc` (1.1.0-1)
- `python3-pathable` (0.4.4-1)
- `python3-pyrsistent` (0.20.0-2+b1)
- `python3-pythonjsonlogger` (3.3.0-1) - Replaced custom python3-python-json-logger
- `python3-watchfiles` (0.24.0-1+b2)
- `python3-websockets` (15.1-1)

## Verification

After applying all fixes and rebuilding packages:

```bash
$ zabbix-mcp --help
# Successfully displays FastMCP 2.14.4 banner and Zabbix MCP Server info
```

## Notes

- All packages use `--force-overwrite` during installation due to the `key_value/__init__.py` file being shared between py-key-value-shared and py-key-value-aio (namespace package pattern)
- The authlib patch uses try/except to maintain backward compatibility with both old and new joserfc versions
- The pydocket Lua runtime patching was disabled entirely as it's a workaround for a memory leak and the API changed in fakeredis 2.29.0
