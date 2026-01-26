# GitHub Repositories for zabbix-mcp-server and Dependencies

All modified packages have been forked to the Vitexus organization with proper Debian package naming.

## Repositories Created

### 1. zabbix-mcp-server
- **URL**: https://github.com/Vitexus/zabbix-mcp-server
- **Description**: Zabbix MCP Server - FastMCP server for Zabbix monitoring
- **Status**: ✅ Pushed with DEPENDENCY_FIXES.md
- **Jenkinsfile**: ✅ Present in debian/Jenkinsfile

### 2. python3-pydocket
- **URL**: https://github.com/Vitexus/python3-pydocket
- **Description**: Debian package: Distributed background task system for Python (fork with fakeredis 2.29.0 fixes)
- **Original**: https://github.com/chrisguidry/docket
- **Changes**: Fixed fakeredis 2.29.0 compatibility
- **Jenkinsfile**: ✅ Present in debian/Jenkinsfile

### 3. python3-authlib
- **URL**: https://github.com/Vitexus/python3-authlib
- **Description**: Debian package: OAuth/OIDC library (fork with joserfc compatibility)
- **Original**: https://github.com/lepture/authlib
- **Changes**: Fixed joserfc BaseClaimsRegistry compatibility
- **Jenkinsfile**: ✅ Present in debian/Jenkinsfile

### 4. python3-py-key-value-shared
- **URL**: https://github.com/Vitexus/python3-py-key-value-shared
- **Description**: Debian package: Shared components for py-key-value (fork with namespace fixes)
- **Original**: https://github.com/strawgate/py-key-value (monorepo)
- **Changes**: Added namespace package __init__.py files
- **Jenkinsfile**: ✅ Present in debian/Jenkinsfile

### 5. python3-py-key-value-aio
- **URL**: https://github.com/Vitexus/python3-py-key-value-aio
- **Description**: Debian package: Async I/O components for py-key-value (fork with namespace fixes)
- **Original**: https://github.com/strawgate/py-key-value (monorepo)
- **Changes**: Added namespace package __init__.py file
- **Jenkinsfile**: ✅ Present in debian/Jenkinsfile

## Jenkins Pipeline Setup

All repositories contain `debian/Jenkinsfile` with the following configuration:

```groovy
@Library('jenkins-pipeline-tools') _

debianBuildPipeline {
    packageName = '<package-name>'
    distribution = 'trixie'
    buildDeps = true
}
```

### Creating Jenkins Jobs

To create Jenkins pipeline jobs at https://jenkins.proxy.spojenet.cz/job/Foregin/:

1. **Navigate to Jenkins**: https://jenkins.proxy.spojenet.cz/job/Foregin/

2. **Create New Item** for each package:
   - Click "New Item"
   - Enter name: `python3-pydocket`, `python3-authlib`, `python3-py-key-value-shared`, `python3-py-key-value-aio`, or `zabbix-mcp-server`
   - Select "Multibranch Pipeline"
   - Click OK

3. **Configure Branch Sources**:
   - Add source: GitHub
   - Repository HTTPS URL: `https://github.com/Vitexus/<repo-name>`
   - Credentials: Select appropriate GitHub credentials
   - Behaviors: Discover branches (All branches)

4. **Build Configuration**:
   - Mode: by Jenkinsfile
   - Script Path: `debian/Jenkinsfile`

5. **Scan Repository Triggers**:
   - Enable "Periodically if not otherwise run"
   - Interval: 1 day (or as desired)

6. **Save and Build**:
   - Click "Save"
   - Click "Scan Repository Now"

### Jenkins Job URLs (once created)

- https://jenkins.proxy.spojenet.cz/job/Foregin/job/zabbix-mcp-server/
- https://jenkins.proxy.spojenet.cz/job/Foregin/job/python3-pydocket/
- https://jenkins.proxy.spojenet.cz/job/Foregin/job/python3-authlib/
- https://jenkins.proxy.spojenet.cz/job/Foregin/job/python3-py-key-value-shared/
- https://jenkins.proxy.spojenet.cz/job/Foregin/job/python3-py-key-value-aio/

## Verification

All repositories are publicly accessible and contain:
- ✅ Source code with fixes
- ✅ Debian packaging in `debian/` directory
- ✅ `debian/Jenkinsfile` for CI/CD
- ✅ README.md with build/install instructions
- ✅ Proper commit history with co-author attribution

## Build Order

When building from scratch, follow this order to satisfy dependencies:

1. `python3-authlib` (no dependencies on other custom packages)
2. `python3-py-key-value-shared` (no dependencies on other custom packages)
3. `python3-py-key-value-aio` (depends on python3-py-key-value-shared)
4. `python3-pydocket` (depends on python3-py-key-value-aio, python3-py-key-value-shared)
5. `zabbix-mcp-server` (depends on all above)
