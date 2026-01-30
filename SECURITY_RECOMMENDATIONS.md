# Security Recommendations for Tauri Skills

This document provides specific, actionable recommendations for improving the security guidance in the Tauri Skills documentation.

## Critical Security Warnings to Add

### 1. CSP Default Warning (HIGH PRIORITY)

**File**: `tauri/tauri-csp/SKILL.md`

**Add this warning section near the beginning:**

```markdown
## ⚠️ CRITICAL: CSP Not Enabled by Default

**Default Configuration**: Tauri's default CSP is `null`, which provides **NO XSS PROTECTION**.

You MUST explicitly configure a Content Security Policy in your application. Without CSP configuration, your application is vulnerable to Cross-Site Scripting attacks.

**Minimum Secure Configuration:**
```json
{
  "app": {
    "security": {
      "csp": {
        "default-src": "'self'",
        "script-src": "'self'",
        "style-src": "'self' 'unsafe-inline'",
        "img-src": "'self' data: https:",
        "connect-src": "ipc: http://ipc.localhost"
      }
    }
  }
}
```

**Never deploy to production without configuring CSP.**
```

### 2. Brownfield Pattern Warning (HIGH PRIORITY)

**File**: `tauri/tauri-ipc/SKILL.md`

**Add this warning in the Brownfield Pattern section:**

```markdown
## ⚠️ Security Considerations for Brownfield Pattern

The brownfield pattern (Tauri's default) has **NO ISOLATION LAYER** between the frontend and backend. This means:

**Vulnerabilities:**
- Supply chain attacks in frontend dependencies can directly invoke backend commands
- Compromised JavaScript code has full access to all registered commands
- No cryptographic validation of IPC messages

**When Brownfield is Acceptable:**
- Internal tools with full control over the supply chain
- Applications with minimal attack surface
- Development/prototyping phase

**When to Use Isolation Pattern Instead:**
- Production applications handling sensitive data
- Applications loading third-party JavaScript
- Applications with external dependencies
- Public-facing applications

**Mitigation in Brownfield:**
- Implement strict input validation in ALL Rust commands
- Use allowlists, never blacklists
- Minimize the number of exposed commands
- Regular dependency audits (`cargo audit`, `npm audit`)
```

### 3. Scope Validation Clarification (HIGH PRIORITY)

**File**: `tauri/tauri-scope/SKILL.md`

**Add this section:**

```markdown
## ⚠️ Critical: Scope Enforcement is NOT Automatic

**IMPORTANT**: Defining scopes in your configuration does NOT automatically enforce them. Commands must manually validate scopes.

**Incorrect Assumption:**
```rust
// ❌ WRONG: This does NOT check scopes automatically
#[tauri::command]
fn read_file(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path).map_err(|e| e.to_string())
}
```

**Correct Implementation:**
```rust
// ✅ CORRECT: Manual scope validation required
#[tauri::command]
fn read_file(app: tauri::AppHandle, path: String) -> Result<String, String> {
    // Validate path against scope
    if !app.path_resolver().is_allowed(&path) {
        return Err("Path not in allowed scope".to_string());
    }
    std::fs::read_to_string(&path).map_err(|e| e.to_string())
}
```

**Key Points:**
- Scopes define policy, but commands must enforce it
- Use Tauri's path resolver API for validation
- Test scope enforcement with both allowed and denied paths
- Consider using the `fs` plugin which has built-in scope checking
```

## Comprehensive Deny List Template

**Files**: `tauri/tauri-scope/SKILL.md`, `tauri/tauri-permissions/SKILL.md`

**Add this comprehensive template:**

```markdown
## Comprehensive Sensitive Directory Deny List

Protect sensitive directories by explicitly denying access:

```toml
# SSH Keys and Credentials
[[scope.deny]]
path = "$HOME/.ssh/**"

# GPG/PGP Keys
[[scope.deny]]
path = "$HOME/.gnupg/**"

# Kubernetes Configuration
[[scope.deny]]
path = "$HOME/.kube/**"

# AWS Credentials
[[scope.deny]]
path = "$HOME/.aws/**"

# Azure Credentials
[[scope.deny]]
path = "$HOME/.azure/**"

# Google Cloud Credentials
[[scope.deny]]
path = "$HOME/.config/gcloud/**"

# Docker Configuration (may contain registry credentials)
[[scope.deny]]
path = "$HOME/.docker/config.json"

# Git Credentials
[[scope.deny]]
path = "$HOME/.git-credentials"
[[scope.deny]]
path = "$HOME/.gitconfig"

# Browser Profiles (may contain cookies, passwords)
[[scope.deny]]
path = "$HOME/.mozilla/**"
[[scope.deny]]
path = "$HOME/.config/google-chrome/**"
[[scope.deny]]
path = "$HOME/.config/chromium/**"
[[scope.deny]]
path = "$APPLOCALDATA/Google/Chrome/User Data/**"
[[scope.deny]]
path = "$APPLOCALDATA/Microsoft/Edge/User Data/**"

# Password Managers
[[scope.deny]]
path = "$HOME/.password-store/**"

# Environment Files
[[scope.deny]]
path = "$HOME/.env"
[[scope.deny]]
path = "$HOME/.env.local"
```

**Platform-Specific Additions:**

```toml
# Windows Credential Manager
[[scope.deny]]
path = "$LOCALAPPDATA/Microsoft/Credentials/**"

# macOS Keychain (can't be read directly but include for completeness)
[[scope.deny]]
path = "$HOME/Library/Keychains/**"
```
```

## Glob Pattern Examples

**File**: `tauri/tauri-scope/SKILL.md`

**Add detailed examples section:**

```markdown
## Understanding Glob Patterns

Glob patterns control directory recursion and file matching. Understanding the difference is critical for security.

### Single Star (*) vs Double Star (**)

| Pattern | Matches | Doesn't Match |
|---------|---------|---------------|
| `$HOME/*` | `$HOME/file.txt` | `$HOME/subdir/file.txt` |
| `$HOME/**` | `$HOME/file.txt`, `$HOME/subdir/file.txt`, `$HOME/a/b/c/file.txt` | _(matches all)_ |

### Extension Matching

| Pattern | Matches | Doesn't Match |
|---------|---------|---------------|
| `$APPDATA/*.json` | `$APPDATA/config.json` | `$APPDATA/data/config.json` |
| `$APPDATA/**/*.json` | `$APPDATA/config.json`, `$APPDATA/data/config.json` | `$APPDATA/config.toml` |

### Common Mistakes

```toml
# ❌ WRONG: Overly permissive - allows ALL files in HOME
[[scope.allow]]
path = "$HOME/**"

# ✅ CORRECT: Specific to application data
[[scope.allow]]
path = "$APPDATA/MyApp/**"

# ❌ WRONG: Missing recursion - won't match subdirectories
[[scope.allow]]
path = "$DOCUMENTS/*"

# ✅ CORRECT: Explicit recursion
[[scope.allow]]
path = "$DOCUMENTS/MyAppData/**"
```

### Testing Your Patterns

Always test scope patterns with:
1. Allowed paths (should succeed)
2. Denied paths (should fail)
3. Edge cases (symlinks, hidden files, parent directory access)

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_scope_allows_app_data() {
        // Test that allowed paths work
    }

    #[test]
    fn test_scope_denies_ssh() {
        // Test that denied paths are blocked
    }

    #[test]
    fn test_scope_blocks_parent_traversal() {
        // Test that ../ doesn't escape scope
    }
}
```
```

## Wildcard Capabilities Warning

**File**: `tauri/tauri-capabilities/SKILL.md`

**Add warning section:**

```markdown
## ⚠️ Avoid Wildcard Window Selectors

Using `"windows": ["*"]` grants permissions to **ALL** windows, including:
- Dynamically created windows
- Future windows added later
- Windows you didn't anticipate

**Security Risk:**
If any window is compromised (e.g., loads untrusted content), it has full permissions.

**Bad Practice:**
```json
{
  "identifier": "overly-permissive",
  "windows": ["*"],
  "permissions": ["fs:allow-write-file"]
}
```

**Best Practice:**
```json
{
  "identifier": "main-window-only",
  "windows": ["main"],
  "permissions": ["fs:allow-write-file"]
}
```

**Exception**: Use wildcards only for read-only, low-risk permissions like:
```json
{
  "identifier": "safe-defaults",
  "windows": ["*"],
  "permissions": ["app:default", "window:default"]
}
```
```

## Dependency Audit Automation

**File**: `tauri/tauri-ecosystem-security/SKILL.md`

**Add CI/CD integration section:**

```markdown
## Automating Security Audits in CI/CD

### GitHub Actions Example

```yaml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      
      - name: Audit Rust Dependencies
        run: |
          cargo install cargo-audit
          cargo audit --deny warnings
      
      - name: Audit npm Dependencies
        run: |
          npm audit --audit-level=high
```

### Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Running security audit..."
cargo audit
if [ $? -ne 0 ]; then
    echo "❌ cargo audit found vulnerabilities"
    exit 1
fi

npm audit --audit-level=moderate
if [ $? -ne 0 ]; then
    echo "❌ npm audit found vulnerabilities"
    exit 1
fi

echo "✅ Security audit passed"
```

### Recommended Audit Cadence

| Frequency | Action | Tool |
|-----------|--------|------|
| Every commit | Pre-commit hook | `cargo audit`, `npm audit` |
| Every PR | CI pipeline | `cargo audit`, `npm audit` |
| Daily | Scheduled CI | `cargo audit`, `npm audit` |
| Weekly | Manual review | Security advisories |
| Monthly | Dependency updates | `cargo update`, `npm update` |
| Per release | Comprehensive audit | All tools + manual review |
```

## Remote Access Security

**File**: `tauri/tauri-permissions/SKILL.md`

**Add security warning:**

```markdown
## ⚠️ Security Implications of Remote Access

### Platform Limitations

**iOS and Android**: Cannot distinguish between iframes and windows for remote access permissions.

**Security Risk:**
```json
{
  "identifier": "remote-access",
  "windows": ["main"],
  "remote": {
    "urls": ["https://trusted-domain.com"]
  },
  "permissions": ["fs:allow-read-file"]
}
```

On iOS/Android, if `trusted-domain.com` embeds an iframe from `malicious-site.com`, the iframe may also receive the permissions.

### Best Practices for Remote Content

1. **Minimize Permissions**: Give remote content only essential permissions
2. **Separate Windows**: Use dedicated windows for remote content with isolated capabilities
3. **Monitor CSP**: Ensure CSP prevents unexpected iframe embedding
4. **Consider Alternatives**: Can the functionality be implemented without remote content?

### Safe Remote Access Example

```json
{
  "identifier": "remote-viewer",
  "description": "Read-only access for remote content viewer",
  "windows": ["viewer"],
  "remote": {
    "urls": ["https://trusted-cdn.com"]
  },
  "permissions": [
    "http:default",
    "core:window:default"
  ]
}
```

**Note**: No filesystem or system permissions granted to remote content.
```

## Update Security Section

**File**: Create new `tauri/tauri-update-security/SKILL.md`

**New skill document:**

```markdown
---
name: securing-tauri-updates
description: Security best practices for implementing and validating application updates in Tauri applications.
---

# Securing Tauri Application Updates

This skill covers security considerations for implementing application updates in Tauri.

## Update Security Threats

| Threat | Description | Impact |
|--------|-------------|--------|
| Manifest tampering | Attacker modifies update manifest to point to malicious binaries | Full compromise |
| Binary tampering | Attacker replaces legitimate update binary | Full compromise |
| Downgrade attack | Attacker forces installation of older vulnerable version | Known exploit |
| Man-in-the-middle | Attacker intercepts update download | Full compromise |

## Update Validation Requirements

### 1. HTTPS for Update Manifest

**Required**: Always serve update manifests over HTTPS.

```json
{
  "plugins": {
    "updater": {
      "endpoints": [
        "https://updates.myapp.com/{{target}}/{{current_version}}"
      ]
    }
  }
}
```

### 2. Signature Verification

**Critical**: Tauri automatically verifies update signatures using your public key.

```json
{
  "plugins": {
    "updater": {
      "pubkey": "YOUR_PUBLIC_KEY_HERE"
    }
  }
}
```

**Generate keypair:**
```bash
tauri signer generate
```

**Key Management:**
- Store private key in secure location (hardware key, secrets manager)
- Never commit private key to version control
- Rotate keys periodically
- Use different keys for different release channels

### 3. Version Validation

**Implement version checks:**

```rust
use tauri::updater::UpdaterBuilder;

#[tauri::command]
async fn check_update(app: tauri::AppHandle) -> Result<(), String> {
    let current = app.package_info().version.clone();
    
    let updater = UpdaterBuilder::new(&app)
        .build()
        .map_err(|e| e.to_string())?;
    
    if let Some(update) = updater.check().await.map_err(|e| e.to_string())? {
        // Verify version is newer, not older (prevent downgrade)
        if update.version <= current {
            return Err("Update version not newer than current".to_string());
        }
        
        // Proceed with update
        update.download_and_install().await.map_err(|e| e.to_string())?;
    }
    
    Ok(())
}
```

## Update Manifest Security

### Secure Manifest Example

```json
{
  "version": "1.2.3",
  "date": "2026-01-30T12:00:00Z",
  "platforms": {
    "windows-x86_64": {
      "url": "https://updates.myapp.com/MyApp-1.2.3-x64.msi",
      "signature": "BASE64_SIGNATURE_HERE"
    }
  },
  "notes": "Security update: Patches CVE-2024-XXXXX"
}
```

### Manifest Best Practices

1. **Include Release Date**: Helps detect replay attacks
2. **Detailed Release Notes**: Especially for security updates
3. **Per-platform Signatures**: Separate signature for each platform
4. **Checksum Verification**: Consider adding SHA256 hashes

## Update Distribution Security

### Hosting Requirements

| Requirement | Purpose |
|-------------|---------|
| HTTPS only | Prevent MITM |
| CDN with integrity checking | Detect tampering |
| Access logs | Monitor for attacks |
| Rate limiting | Prevent DoS |
| Geo-restrictions (optional) | Limit attack surface |

### Signing Workflow

```bash
# 1. Build release
npm run tauri build

# 2. Sign update binary
tauri signer sign target/release/bundle/msi/MyApp.msi

# 3. Generate update manifest
# (Include signature from step 2)

# 4. Upload to secure hosting
# - Binary: https://updates.myapp.com/MyApp-1.2.3.msi
# - Manifest: https://updates.myapp.com/latest.json
```

## Update Installation Security

### Verification Before Installation

```typescript
import { check } from '@tauri-apps/plugin-updater';

async function checkAndUpdate() {
  try {
    const update = await check();
    
    if (update?.available) {
      // Show update details to user
      const confirmed = await showUpdateDialog({
        version: update.version,
        date: update.date,
        notes: update.notes
      });
      
      if (confirmed) {
        // Download with progress
        await update.downloadAndInstall((progress) => {
          console.log(`Progress: ${progress.downloaded}/${progress.total}`);
        });
        
        // Restart application
        await relaunch();
      }
    }
  } catch (error) {
    console.error('Update failed:', error);
    // Don't leave app in broken state - handle gracefully
  }
}
```

## Security Checklist

```markdown
## Pre-Release
- [ ] Generate and secure signing keypair
- [ ] Configure public key in tauri.conf.json
- [ ] Test update on all target platforms
- [ ] Verify signature validation works
- [ ] Test downgrade attack prevention

## Release Process
- [ ] Build release binaries
- [ ] Sign all platform binaries
- [ ] Generate update manifest with signatures
- [ ] Upload to secure HTTPS hosting
- [ ] Verify manifest is accessible
- [ ] Test update installation

## Post-Release
- [ ] Monitor update server logs
- [ ] Track update adoption rate
- [ ] Watch for update failures
- [ ] Rotate keys periodically
```

## Emergency Update Procedure

For critical security updates:

1. **Immediate Communication**: Notify users through multiple channels
2. **Forced Update**: Consider requiring update before app usage
3. **Rollback Plan**: Keep previous version available if new version has issues
4. **Verification**: Extra testing for security updates

```rust
#[tauri::command]
async fn check_critical_update(app: tauri::AppHandle) -> Result<bool, String> {
    // Check if current version has known critical vulnerability
    let current = app.package_info().version.clone();
    
    if is_critically_vulnerable(&current) {
        // Force update check
        let update = updater::check().await?;
        if update.is_some() {
            return Ok(true); // Signal critical update available
        }
    }
    
    Ok(false)
}
```

## Summary

Update security requires:
1. **HTTPS** for all update traffic
2. **Signature verification** for all binaries
3. **Version validation** to prevent downgrades
4. **Secure key management**
5. **Monitoring and logging**

The weakest link in your update chain determines your security posture.
```
```

## Summary

These recommendations focus on making security guidance more explicit and actionable. Implementing them will:

1. **Reduce misconfiguration risks** by highlighting default insecure settings
2. **Improve developer awareness** of security implications
3. **Provide actionable templates** for secure configurations
4. **Fill documentation gaps** in critical security areas

**Priority Order:**
1. Add critical warnings (CSP, brownfield, scope validation)
2. Provide comprehensive deny lists and glob examples
3. Add CI/CD automation guidance
4. Create missing security documentation (updates, remote access)

---

**Status**: Ready for implementation  
**Estimated Effort**: 4-6 hours for high-priority items  
**Expected Impact**: Significant improvement in security posture of Tauri applications built using these skills
