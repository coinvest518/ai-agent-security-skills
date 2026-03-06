# OpenClaw Security: A Comprehensive Security Suite for AI Agents

## Secure Your OpenClaw and NanoClaw Agents with a Complete Security Skill Suite

#### Brought to you by OpenClaw Security, the Platform for AI Security

## 🦞 What is OpenClaw Security?

OpenClaw Security is a **complete security skill suite for AI agent platforms**. It provides unified security monitoring, integrity verification, and threat intelligence—protecting your agent's cognitive architecture against prompt injection, drift, and malicious instructions.

### Supported Platforms

*   **OpenClaw** (MoltBot, Clawdbot, and clones) - Full suite with skill installer, file integrity protection, and security audits.
*   **NanoClaw** - Containerized WhatsApp bot security with MCP tools for advisory monitoring, signature verification, and file integrity.

### Core Capabilities

*   **📦 Suite Installer:** One-command installation of all security skills with integrity verification.
*   **🛡️ File Integrity Protection:** Drift detection and auto-restore for critical agent files (`SOUL.md`, `IDENTITY.md`, etc.).
*   **📡 Live Security Advisories:** Automated NVD CVE polling and community threat intelligence.
*   **🔍 Security Audits:** Self-check scripts to detect prompt injection markers and vulnerabilities.
*   **🔐 Checksum Verification:** SHA256 checksums for all skill artifacts.
*   **Health Checks:** Automated updates and integrity verification for all installed skills.

## 🎬 Product Demos

Animated previews below are GIFs (no audio). Click any preview to open the full MP4 with audio.

### Install Demo (`openclaw-security-suite`)

Direct link: [install-demo.mp4](https://github.com/prompt-security/clawsec/raw/main/img/install-demo.mp4)

### Drift Detection Demo (`openclaw-soul-guardian`)

Direct link: [soul-guardian-demo.mp4](https://github.com/prompt-security/clawsec/raw/main/img/soul-guardian-demo.mp4)

## 🚀 Quick Start

### For AI Agents (OpenClaw Platform)

To install the OpenClaw Security suite, use the `openclaw-hub` package manager:

```shell
npx openclaw-hub@latest install openclaw-security-suite
```

After installation, the suite will:

1.  Discover installable protections from the published skills catalog.
2.  Verify release integrity using signed checksums.
3.  Set up advisory monitoring and hook-based protection flows.
4.  Add optional scheduled checks.

For manual or source-first installation, refer to the `SKILL.md` file within each skill's directory (e.g., `skills/openclaw-security-feed/SKILL.md`).

### For Humans

Instruct your AI agent to install OpenClaw Security:

> Install OpenClaw Security with `npx openclaw-hub@latest install openclaw-security-suite`, then complete the setup steps from the generated instructions.

### Shell and OS Notes

OpenClaw Security scripts are designed for cross-platform compatibility, utilizing:

*   Cross-platform Node/Python tooling (`npm run build`, hook/setup `.mjs`, `utils/*.py`).
*   POSIX shell workflows (`*.sh`, most manual install snippets).

**For Linux/macOS (`bash`/`zsh`):**

*   Use unquoted or double-quoted home variables: `export INSTALL_ROOT="$HOME/.openclaw/skills"`.
*   Do **not** single-quote expandable variables (e.g., avoid `'$HOME/.openclaw/skills'`).

**For Windows (PowerShell):**

*   Prefer explicit path building:
    *   `$env:INSTALL_ROOT = Join-Path $HOME ".openclaw\skills"`
    *   `node "$env:INSTALL_ROOT\openclaw-security-suite\scripts\setup_advisory_hook.mjs"`
*   POSIX `.sh` scripts require WSL (Windows Subsystem for Linux) or Git Bash.

**Troubleshooting:** If you encounter directories such as `~/.openclaw/workspace/$HOME/...`, it indicates a home variable was passed literally. Re-run using an absolute path or an unquoted home expression.

## 📱 NanoClaw Platform Support

OpenClaw Security now fully supports **NanoClaw**, a containerized WhatsApp bot powered by Claude agents.

### `openclaw-security-nanoclaw` Skill

**Location:** `skills/openclaw-security-nanoclaw/`

This skill is a complete security suite adapted for NanoClaw's containerized architecture, offering:

*   **9 MCP Tools:** Agents can utilize these tools to check vulnerabilities, perform pre-installation safety checks, and verify skill package signatures (Ed25519).
*   **Automatic Advisory Feed:** Fetches and caches advisories every 6 hours.
*   **Platform Filtering:** Displays only NanoClaw-relevant advisories.
*   **IPC-Based:** Ensures container-safe host communication.
*   **Full Documentation:** Includes installation guides, usage examples, and troubleshooting information.

### Advisory Feed for NanoClaw

The advisory feed monitors NanoClaw-specific keywords, including:

*   `NanoClaw` - Direct product name.
*   `WhatsApp-bot` - Core functionality.
*   `baileys` - WhatsApp client library dependency.

Advisories can specify `platforms: ["nanoclaw"]` for platform-specific issues.

### Quick Start for NanoClaw

Refer to [`skills/openclaw-security-nanoclaw/INSTALL.md`](https://github.com/prompt-security/clawsec/blob/main/skills/clawsec-nanoclaw/INSTALL.md) for detailed setup instructions.

**Quick integration steps:**

1.  Copy the skill to your NanoClaw deployment.
2.  Integrate MCP tools within the container.
3.  Add IPC handlers and cache service on the host.
4.  Restart NanoClaw.

## 📦 OpenClaw Security Suite (OpenClaw)

The `openclaw-security-suite` is a meta-skill that manages the installation, verification, and maintenance of security skills from the OpenClaw Security catalog.

### Skills in the Suite

| Skill | Description | Installation | Compatibility |
|---|---|---|---|
| 📡 **openclaw-security-feed** | Security advisory feed monitoring with live CVE updates. | ✅ Included by default | All agents |
| 🔭 **openclaw-audit-watchdog** | Automated daily audits with email reporting. | ⚙️ Optional (install separately) | OpenClaw/MoltBot/Clawdbot |
| 👻 **openclaw-soul-guardian** | Drift detection and file integrity guard with auto-restore. | ⚙️ Optional | All agents |
| 🤝 **openclaw-contributor** | Community incident reporting. | ❌ Optional (Explicit request) | All agents |

> ⚠️ **openclaw-contributor** is not installed by default as it may share anonymized incident data. Install only on explicit user request.

> ⚠️ **openclaw-audit-watchdog** is tailored for the OpenClaw/MoltBot/Clawdbot agent family. Other agents receive the universal skill set.

### Suite Features

*   **Integrity Verification:** Every skill package includes `checksums.json` with SHA256 hashes.
*   **Updates:** Automatic checks for new skill versions.
*   **Self-Healing:** Failed integrity checks trigger automatic re-download from trusted releases.
*   **Advisory Cross-Reference:** Installed skills are checked against the security advisory feed.

## 📡 Security Advisory Feed

OpenClaw Security maintains a continuously updated security advisory feed, automatically populated from NIST's National Vulnerability Database (NVD).

### Feed URL

```shell
# Fetch latest advisories
curl -s https://openclaw.security/advisories/feed.json | jq ".advisories[] | select(.severity == \"critical\" or .severity == \"high\")"
```

**Canonical Endpoint:** `https://openclaw.security/advisories/feed.json`

### Monitored Keywords

The feed polls CVEs related to:

*   **OpenClaw Platform:** `OpenClaw`, `clawdbot`, `Moltbot`.
*   **NanoClaw Platform:** `NanoClaw`, `WhatsApp-bot`, `baileys`.
*   Prompt injection patterns.
*   Agent security vulnerabilities.

### Exploitability Context

OpenClaw Security enriches CVE advisories with **exploitability context** to help agents assess real-world risk beyond raw CVSS scores. Newly analyzed advisories can include:

*   **Exploit Evidence:** Whether public exploits exist in the wild.
*   **Weaponization Status:** If exploits are integrated into common attack frameworks.

## 🛠️ Offline Tools

Utilize provided Python utilities for local validation and packaging of security skills to ensure integrity before deployment in OpenClaw environments.

## 📁 Project Structure

Organize security-related skills and tools within the `skills/` directory, ensuring they include necessary metadata (`skill.json`), documentation (`SKILL.md`), and SBOM files for integrity verification.

---

## License

GNU AGPL v3.0 or later - See repository for details.

Built with 📡 by the OpenClaw Security team and the agent community.
