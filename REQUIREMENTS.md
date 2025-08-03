# 🧾 Product Requirements Document (PRD)

## 📌 Title

**Multi-Environment DevContainer with Pluggable Secret Backend Support**

---

## 🧠 Overview

This DevContainer system provides secure, multi-environment development with secrets injected from pluggable password managers. It uses a unified interface that delegates secret resolution to backend-specific adapters, supporting different secret formats and authentication methods per backend.

**Current Status**: 1Password fully implemented, Bitwarden and LastPass planned for v2.0.

---

## 🎯 Goals

### Primary Goals
- Enable secure local and Codespaces development with environment-specific secrets
- Support multiple secret backends through a pluggable architecture
- Maintain zero secrets in source control
- Provide fast, reliable secret injection during container startup

### Secondary Goals
- Enable teams to use their preferred password managers
- Support offline development with cached secrets (optional)
- Provide clear error messages and fallback strategies

---

## 👥 Target Users

- **Primary**: Development teams using DevContainers with enterprise password managers
- **Secondary**: Individual developers wanting secure local development
- **Tertiary**: DevOps teams managing multi-environment deployments

---

## 📂 Functional Requirements

### 1. **Environment Management**
- Support configurable environments via `DEPLOY_ENV` (default: `dev`)
- Standard environments: `dev`, `test`, `staging`, `prod`
- Custom environments allowed via configuration
- Secrets output to: `.devcontainer/secrets/.env.${DEPLOY_ENV}`

### 2. **Backend Architecture**
- **Primary Backend**: 1Password CLI (fully implemented)
- **Planned Backends**: Bitwarden, LastPass, HashiCorp Vault
- Backend selection via `SECRET_BACKEND` environment variable (default: `1password`)
- Each backend implements standard interface: `fetch-secrets <template-path> <output-path> <environment>`

### 3. **Secret Template System**
- **Local Template**: `.devcontainer/secrets/env.template.local` (gitignored, developer-specific)
- **Shared Template**: `.devcontainer/secrets/env.template.example` (committed, with placeholder values)
- Template format: Backend-agnostic with plugin-specific resolution
- Validation against required environment variables

### 4. **Authentication Strategies**
- **Local Development**: Interactive CLI authentication
- **Codespaces**: Service principals, API tokens, or GitHub Secrets integration
- **CI/CD**: Service account authentication
- Fallback to cached secrets when backend unavailable

### 5. **Error Handling & Validation**
- Pre-flight checks for required CLI tools and authentication
- Secret validation against template requirements
- Graceful fallback to cached secrets (24-hour expiry)
- Clear error messages with remediation steps

---

## ⚙️ Non-Functional Requirements

### Performance
- Container startup impact: <10 seconds for secret injection
- Support for up to 50 secrets per environment
- Concurrent secret fetching where supported by backend

### Security
- Secrets never written to logs or temporary files in plaintext
- Automatic cleanup of secret files on container shutdown
- Encrypted caching with user-specific keys
- Audit logging of secret access (where supported by backend)

### Reliability
- 99% success rate for secret injection with proper authentication
- Graceful degradation when backend unavailable
- Retry logic with exponential backoff for network issues

### Compatibility
- **Platforms**: Linux (primary), macOS (supported), Windows/WSL2 (best effort)
- **DevContainer**: Compatible with VS Code and GitHub Codespaces
- **Shells**: bash 4.0+, zsh (with bash compatibility)

---

## 📁 Directory Structure

```
.devcontainer/
├── devcontainer.json
├── scripts/
│   ├── fetch-secrets.sh           # Main orchestrator
│   ├── validate-secrets.sh        # Validation logic
│   └── backends/
│       ├── 1password.sh          # ✅ Full implementation
│       ├── bitwarden.sh          # 🟡 Planned v2.0
│       ├── lastpass.sh           # 🟡 Planned v2.0
│       └── backend-interface.sh  # Standard interface definition
├── secrets/
│   ├── .env.*                    # Generated files (gitignored)
│   ├── .cache/                   # Encrypted cache (gitignored)
│   ├── env.template.example      # Committed example
│   └── env.template.local        # Developer-specific (gitignored)

.gitignore                        # Updated with secret exclusions
README.md
```

---

## 🔐 Secret Backend Specifications

### 1. **Backend Interface Standard**
All backends must implement:
```bash
fetch-secrets <template-path> <output-path> <environment>
# Returns: 0 (success), 1 (auth error), 2 (missing secrets), 3 (backend unavailable)
```

### 2. **Template Format**
**Backend-Agnostic Format:**
```bash
# Comments supported
API_KEY=@secret:api-keys/service-api-key
DATABASE_URL=@secret:databases/app-db-${DEPLOY_ENV}/connection-string
OPTIONAL_SECRET=@secret:optional/feature-flag@default:false
```

**Format Specification:**
- `@secret:` prefix indicates secret reference
- `@default:` suffix provides fallback value
- `${DEPLOY_ENV}` variable substitution supported
- Lines without `@secret:` treated as literal values

### 3. **1Password Implementation**
```bash
# Template Reference
API_KEY=@secret:Private/MyApp-API-Key/credential
# Resolves to: op read "op://Private/MyApp-API-Key/credential"
```

### 4. **Bitwarden Implementation (Planned)**
```bash
# Template Reference  
API_KEY=@secret:MyApp-Secrets/API-Key
# Resolves to: bw get password "API-Key" --search "MyApp-Secrets"
```

---

## 🔄 Authentication Flows

### Local Development
```bash
# 1Password
export SECRET_BACKEND=1password
eval $(op signin)  # Interactive authentication
devcontainer up

# Bitwarden (planned)
export SECRET_BACKEND=bitwarden
bw login && bw unlock
devcontainer up
```

### GitHub Codespaces
```bash
# Option 1: GitHub Secrets (for simple cases)
# Store secrets as CODESPACE_SECRET_* in repo settings

# Option 2: 1Password Connect (recommended)
# Set OP_CONNECT_HOST and OP_CONNECT_TOKEN in Codespaces secrets

# Option 3: Service Principal
# Set backend-specific service credentials in Codespaces secrets
```

---

## 📋 DevContainer Integration

### devcontainer.json Configuration
```json
{
  "name": "Multi-Env DevContainer",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {},
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "postCreateCommand": ".devcontainer/scripts/fetch-secrets.sh",
  "runArgs": [
    "--env-file=.devcontainer/secrets/.env.${localEnv:DEPLOY_ENV:dev}"
  ],
  "containerEnv": {
    "DEPLOY_ENV": "${localEnv:DEPLOY_ENV:dev}",
    "SECRET_BACKEND": "${localEnv:SECRET_BACKEND:1password}"
  },
  "mounts": [
    "source=${localWorkspaceFolder}/.devcontainer/secrets/.cache,target=/tmp/secret-cache,type=bind"
  ]
}
```

---

## ✅ Usage Flows

### First-Time Setup
```bash
# 1. Copy example template
cp .devcontainer/secrets/env.template.example .devcontainer/secrets/env.template.local

# 2. Configure your secret references
# Edit .devcontainer/secrets/env.template.local

# 3. Set environment
export DEPLOY_ENV=dev
export SECRET_BACKEND=1password

# 4. Authenticate (backend-specific)
eval $(op signin)

# 5. Open in DevContainer
code .
```

### Daily Development
```bash
# Environment is remembered in your shell profile
devcontainer up  # Automatically fetches secrets for your configured environment
```

### Environment Switching
```bash
export DEPLOY_ENV=test
devcontainer rebuild  # Fetches test environment secrets
```

---

## 🧪 Testing Strategy

### Unit Tests
- [ ] Backend interface compliance for all implementations
- [ ] Template parsing with various formats
- [ ] Error handling for authentication failures
- [ ] Secret validation logic

### Integration Tests
- [ ] End-to-end secret injection with 1Password
- [ ] DevContainer startup with injected secrets
- [ ] Fallback behavior when backend unavailable
- [ ] Multi-environment switching

### Security Tests
- [ ] Verify no secrets in container logs
- [ ] Confirm gitignore effectiveness
- [ ] Test secret cleanup on container shutdown
- [ ] Validate encrypted caching

### Codespaces Tests
- [ ] GitHub Secrets integration
- [ ] Service principal authentication
- [ ] Network connectivity handling

---

## 🚫 Explicit Non-Requirements

- **GUI Secret Management**: CLI-only approach
- **Secret Synchronization**: No automatic secret syncing between environments
- **Multi-Tenant Support**: Single-user/team focus
- **Windows Native Support**: WSL2 required on Windows
- **Secret Generation**: Only retrieval, not creation of new secrets

---

## 📈 Roadmap

### v1.0 (Current)
- ✅ 1Password integration
- ✅ Basic multi-environment support
- ✅ DevContainer integration
- ✅ Local development workflow

### v1.1 (Next)
- [ ] Secret validation and pre-flight checks
- [ ] Encrypted caching with fallback
- [ ] Improved error messages and troubleshooting
- [ ] Codespaces integration guide

### v2.0 (Future)
- [ ] Bitwarden integration
- [ ] LastPass integration
- [ ] HashiCorp Vault support
- [ ] Performance optimizations
- [ ] Advanced template features

---

## 📚 Deliverables

### Core Implementation
- ✅ `devcontainer.json` with proper environment handling
- ✅ `fetch-secrets.sh` orchestrator script
- ✅ 1Password backend implementation
- ✅ Backend interface specification
- [ ] Secret validation script
- [ ] Error handling and fallback logic

### Documentation
- [ ] README with comprehensive setup instructions
- [ ] Backend development guide for contributors
- [ ] Troubleshooting guide
- [ ] Security best practices guide

### Configuration
- ✅ `.gitignore` with comprehensive secret exclusions
- ✅ Example template file
- [ ] VS Code settings recommendations
- [ ] Shell profile configuration examples

---

## 🔍 Success Metrics

### Developer Experience
- Setup time: <5 minutes for new developers
- Daily overhead: <30 seconds for secret refresh
- Error resolution: <2 minutes average with provided documentation

### Security
- Zero secret leaks in source control (monitored via git-secrets)
- 100% of secrets properly excluded from logs
- Encrypted caching adoption rate >80%

### Adoption
- Support for 3+ secret backends by v2.0
- Usage across multiple project types
- Community contributions to backend implementations

---

## 🆘 Risk Mitigation

### Risk: Backend Service Unavailability
- **Mitigation**: Encrypted local caching with configurable TTL
- **Fallback**: Manual secret override file for emergencies

### Risk: Authentication Token Expiry
- **Mitigation**: Automatic token refresh where supported
- **Fallback**: Clear error messages with re-authentication steps

### Risk: Secret Reference Mismatches
- **Mitigation**: Template validation before secret fetching
- **Fallback**: Detailed error reporting with suggested fixes

### Risk: Performance Impact on Container Startup
- **Mitigation**: Concurrent secret fetching and local caching
- **Monitoring**: Startup time metrics and alerting

---

*This PRD addresses security, reliability, and extensibility concerns while maintaining the original vision of pluggable secret management for DevContainers.*
