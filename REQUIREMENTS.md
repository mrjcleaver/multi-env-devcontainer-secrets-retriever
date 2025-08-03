# 🧾 Product Requirements Document (PRD)

## 📌 Title

**Multi-Environment DevContainer with Pluggable Secret Backend Support**

---

## 🧠 Overview

This DevContainer system supports multiple development environments (e.g., `dev`, `test`, `prod`) with the ability to inject secrets from various password managers. It uses a unified interface (`fetch-env.sh`) that delegates secret resolution to backend-specific scripts. Currently, **1Password** is implemented. **Bitwarden** and **LastPass** are stubbed for future development.

---

## 🎯 Goals

- Enable local and Codespaces development with secrets injected per environment.
- Allow different developers to use their preferred secrets manager.
- Keep secrets out of source control.
- Provide a clean, extensible architecture for secret backends.

---

## 👥 Target Users

- Developers working in **DevContainers**
- Teams using different **password managers**
- Projects requiring **multi-environment config**

---

## 📂 Functional Requirements

### 1. **Environment Support**
- Allow secret injection based on an environment variable (`DEPLOY_ENV`)
- Environments supported: `dev`, `test`, `prod`
- Secrets should be output to: `.devcontainer/secrets/env.${DEPLOY_ENV}`

### 2. **Password Manager Backend Support**
- Support at least one functional secrets backend (1Password)
- Stub support for:
  - Bitwarden
  - LastPass
- Use `SECRET_BACKEND` environment variable to select backend

### 3. **Environment Templates**
- Use `.devcontainer/secrets/env.template` to define logical secret keys
- This file is committed to Git and acts as the contract for required secrets

### 4. **Backend Scripts**
- Each backend script should:
  - Accept a template path and output path
  - Fetch secrets for each key
  - Output valid `.env` format

### 5. **Integration with DevContainer**
- Use `devcontainer.json`:
  - Pass generated `.env.${DEPLOY_ENV}` into container with `--env-file`
  - Run `fetch-env.sh` as `postCreateCommand`

---

## ⚙️ Non-Functional Requirements

- Scripts must be **cross-platform** compatible (focus on Linux/macOS; Windows optional)
- Secrets must **never be committed**
- Solution must be extensible for more managers (e.g., Doppler, Vault)
- Simple CLI-based workflow (no GUI dependencies)

---

## 📁 Directory Structure

```
.devcontainer/
├── devcontainer.json
├── fetch-env.sh
├── secrets/
│   ├── env.template
│   └── backends/
│       ├── fetch-from-1password.sh  ✅ implemented
│       ├── fetch-from-bitwarden.sh  🟡 stub
│       └── fetch-from-lastpass.sh   🟡 stub

.gitignore
README.md
```

---

## 🔐 Secrets Handling Details

### 1. **1Password CLI Integration**
- Secrets are stored as item fields in a vault
- Template uses format: `MY_SECRET=project/myapp-dev/MY_SECRET`
- Script calls:  
  ```bash
  value=$(op read "op://$ref")
  ```

### 2. **Template Behavior**
- Each line in `env.template`: `ENV_VAR=SECRET_REFERENCE`
- Backends must resolve `SECRET_REFERENCE` and assign value to `ENV_VAR`

---

## ✅ Usage Flow

### Developer:
```bash
export DEPLOY_ENV=dev
export SECRET_BACKEND=1password
eval $(op signin)
devcontainer up
```

### VS Code UI:
- The `DEPLOY_ENV` and `SECRET_BACKEND` can also be set in `.bashrc` or VS Code `settings.json`

---

## 🚫 Exclusions

- No GUI support for secret entry
- No built-in support for syncing secrets to cloud
- Codespaces integration is optional (via GitHub Secrets or 1Password Connect)

---

## 📈 Future Enhancements

- Support `Teller`, `Vault`, `doppler`, or cloud KMS systems
- Add `fetch-secrets.validate.sh` to check for missing keys
- Add caching and fallback for offline work

---

## 📚 Deliverables

- ✅ DevContainer config (`devcontainer.json`)
- ✅ `fetch-env.sh` logic
- ✅ 1Password integration
- ✅ `.env.template` example
- ✅ README with clear usage instructions
- ✅ `.gitignore` to exclude resolved secrets

---

## 🧪 Testing Plan

- [x] Validate secrets are correctly fetched using 1Password
- [x] Confirm `.env.dev` is created and used in container
- [x] Ensure `.env.*` files are ignored by Git
- [x] Stub scripts do not crash execution
