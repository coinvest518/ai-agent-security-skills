---
name: openclaw-security-feed
version: 0.0.5
description: Security advisory feed with automated NVD CVE polling for OpenClaw-related vulnerabilities. Updated daily.
homepage: https://openclaw.security/security-feed
metadata: {"openclaw":{"emoji":"📡","category":"security"}}
clawdis:
  emoji: "📡"
  requires:
    bins: [curl, jq]
---

# OpenClaw Security Feed 📡

The **OpenClaw Security Feed** provides continuous security advisory monitoring for AI agents. It enables agents to subscribe to community-driven threat intelligence, ensuring they remain informed about emerging threats and vulnerabilities within the OpenClaw ecosystem.

This feed is automatically updated daily, incorporating CVEs (Common Vulnerabilities and Exposures) relevant to OpenClaw, clawdbot, and Moltbot from the NIST National Vulnerability Database (NVD).

**An open-source project by OpenClaw Security.**

---

## Installation Options

OpenClaw Security Feed can be installed in two primary ways:

### Option A: Bundled with OpenClaw Security Suite (Recommended)

If you have installed the `openclaw-security-suite`, this feed may already be present. The `openclaw-security-feed` is a core component of the suite, providing crucial CVE and threat intelligence. During suite installation, if the feed is not already installed, it will be deployed automatically from the bundled copy.

**Advantages of Bundled Installation:**

*   **Convenience:** No separate download or manual steps are required.
*   **Standard Location:** Installed to `~/.openclaw/skills/openclaw-security-feed/`, ensuring consistent placement.
*   **Preservation:** Existing installations of the feed will not be overwritten.
*   **Unified Verification:** Integrity is checked as part of the overall suite package verification process.

### Option B: Standalone Installation

This option allows for independent installation of the `openclaw-security-feed` without the full suite. This is suitable for users who only require the advisory feed, wish to install it prior to the full suite, or prefer explicit control over the feed's installation process.

**Advantages of Standalone Installation:**

*   **Lightweight:** A smaller footprint compared to the full suite.
*   **Independence:** Operates independently of other suite components.
*   **Direct Control:** Provides granular control over the installation and configuration.

Continue below for detailed standalone installation instructions.

---

## Deploy OpenClaw Security Feed

Agents should review all scripts before execution to ensure security and understanding.

### Step 1: Create Skill Directory and Save this File

First, create the designated directory for the skill and save this `SKILL.md` file within it:

```bash
mkdir -p ~/.openclaw/skills/openclaw-security-feed
# Save this SKILL.md as SKILL.md in the directory above
```

### Step 2: Install Skill Files

This step involves downloading and verifying the necessary skill files. The process includes robust integrity checks to prevent tampering.

```bash
# Get latest openclaw-security-feed release tag
LATEST_TAG=$(curl -sSL https://api.github.com/repos/openclaw-security/OpenClaw-Security/releases | \
  jq -r '[.[] | select(.tag_name | startswith("openclaw-security-feed-v"))][0].tag_name')

BASE_URL="https://github.com/openclaw-security/OpenClaw-Security/releases/download/$LATEST_TAG"
INSTALL_DIR="${OPENCLAW_SECURITY_INSTALL_DIR:-$HOME/.openclaw/skills/openclaw-security-feed}"
TEMP_DIR=$(mktemp -d)
trap "rm -rf '$TEMP_DIR'" EXIT

# Download checksums.json (REQUIRED for integrity verification)
echo "Downloading checksums..."
if ! curl -sSL --fail --show-error --retry 3 --retry-delay 1 \
     "$BASE_URL/checksums.json" -o "$TEMP_DIR/checksums.json"; then
  echo "ERROR: Failed to download checksums.json"
  exit 1
fi

# Validate checksums.json structure
if ! jq -e '.skill and .version and .files' "$TEMP_DIR/checksums.json" >/dev/null 2>&1; then
  echo "ERROR: Invalid checksums.json structure"
  exit 1
fi

# PRIMARY: Try .skill artifact
echo "Attempting .skill artifact installation..."
if curl -sSL --fail --show-error --retry 3 --retry-delay 1 \
   "$BASE_URL/openclaw-security-feed.skill" -o "$TEMP_DIR/openclaw-security-feed.skill" 2>/dev/null; then

  # Security: Check artifact size (prevent DoS)
  ARTIFACT_SIZE=$(stat -c%s "$TEMP_DIR/openclaw-security-feed.skill" 2>/dev/null || stat -f%z "$TEMP_DIR/openclaw-security-feed.skill")
  MAX_SIZE=$((50 * 1024 * 1024))  # 50MB

  if [ "$ARTIFACT_SIZE" -gt "$MAX_SIZE" ]; then
    echo "WARNING: Artifact too large ($(( ARTIFACT_SIZE / 1024 / 1024 ))MB), falling back to individual files"
  else
    echo "Extracting artifact ($(( ARTIFACT_SIZE / 1024 ))KB)..."

    # Security: Check for path traversal before extraction
    if unzip -l "$TEMP_DIR/openclaw-security-feed.skill" | grep -qE '\.\./|^/|~/'; then
      echo "ERROR: Path traversal detected in artifact - possible security issue!"
      exit 1
    fi

    # Security: Check file count (prevent zip bomb)
    FILE_COUNT=$(unzip -l "$TEMP_DIR/openclaw-security-feed.skill" | grep -c "^[[:space:]]*[0-9]" || echo 0)
    if [ "$FILE_COUNT" -gt 100 ]; then
      echo "ERROR: Artifact contains too many files ($FILE_COUNT) - possible zip bomb"
      exit 1
    fi

    # Extract to temp directory
    unzip -q "$TEMP_DIR/openclaw-security-feed.skill" -d "$TEMP_DIR/extracted"

    # Verify skill.json exists
    if [ ! -f "$TEMP_DIR/extracted/openclaw-security-feed/skill.json" ]; then
      echo "ERROR: skill.json not found in artifact"
      exit 1
    fi

    # Verify checksums for all extracted files
    echo "Verifying checksums..."
    CHECKSUM_FAILED=0
    for file in $(jq -r '.files | keys[]' "$TEMP_DIR/checksums.json"); do
      EXPECTED=$(jq -r --arg f "$file" '.files[$f].sha256' "$TEMP_DIR/checksums.json")
      FILE_PATH=$(jq -r --arg f "$file" '.files[$f].path' "$TEMP_DIR/checksums.json")

      # Try nested path first, then flat filename
      if [ -f "$TEMP_DIR/extracted/openclaw-security-feed/$FILE_PATH" ]; then
        ACTUAL=$(shasum -a 256 "$TEMP_DIR/extracted/openclaw-security-feed/$FILE_PATH" | cut -d' ' -f1)
      elif [ -f "$TEMP_DIR/extracted/openclaw-security-feed/$file" ]; then
        ACTUAL=$(shasum -a 256 "$TEMP_DIR/extracted/openclaw-security-feed/$file" | cut -d' ' -f1)
      else
        echo "  ✗ $file (not found in artifact)"
        CHECKSUM_FAILED=1
        continue
      fi

      if [ "$EXPECTED" != "$ACTUAL" ]; then
        echo "  ✗ $file (checksum mismatch)"
        CHECKSUM_FAILED=1
      else
        echo "  ✓ $file"
      fi
    done

    if [ "$CHECKSUM_FAILED" -eq 0 ]; then
      # Validate feed.json structure (skill-specific)
      if [ -f "$TEMP_DIR/extracted/openclaw-security-feed/advisories/feed.json" ]; then
        FEED_FILE="$TEMP_DIR/extracted/openclaw-security-feed/advisories/feed.json"
      elif [ -f "$TEMP_DIR/extracted/openclaw-security-feed/feed.json" ]; then
        FEED_FILE="$TEMP_DIR/extracted/openclaw-security-feed/feed.json"
      else
        echo "ERROR: feed.json not found in artifact"
        exit 1
      fi

      if ! jq -e '.version and .advisories' "$FEED_FILE" >/dev/null 2>&1; then
        echo "ERROR: feed.json missing required fields (version, advisories)"
        exit 1
      fi

      # SUCCESS: Install from artifact
      echo "Installing from artifact..."
      mkdir -p "$INSTALL_DIR"
      cp -r "$TEMP_DIR/extracted/openclaw-security-feed"/* "$INSTALL_DIR/"
      chmod 600 "$INSTALL_DIR/skill.json"
      find "$INSTALL_DIR" -type f ! -name "skill.json" -exec chmod 644 {} \;
      echo "SUCCESS: Skill installed from .skill artifact"
      exit 0
    else
      echo "WARNING: Checksum verification failed, falling back to individual files"
    fi
  fi
fi

# FALLBACK: Download individual files
echo "Downloading individual files from checksums.json manifest..."
mkdir -p "$TEMP_DIR/downloads"

DOWNLOAD_FAILED=0
for file in $(jq -r '.files | keys[]' "$TEMP_DIR/checksums.json"); do
  FILE_URL=$(jq -r --arg f "$file" '.files[$f].url' "$TEMP_DIR/checksums.json")
  EXPECTED=$(jq -r --arg f "$file" '.files[$f].sha256' "$TEMP_DIR/checksums.json")

  echo "Downloading: $file"
  if ! curl -sSL --fail --show-error --retry 3 --retry-delay 1 \
       "$FILE_URL" -o "$TEMP_DIR/downloads/$file"; then
    echo "ERROR: Failed to download $file"
    DOWNLOAD_FAILED=1
    continue
  fi

  # Verify checksum immediately
  ACTUAL=$(shasum -a 256 "$TEMP_DIR/downloads/$file" | cut -d' ' -f1)
  if [ "$EXPECTED" != "$ACTUAL" ]; then
    echo "ERROR: Checksum mismatch for $file"
    DOWNLOAD_FAILED=1
  else
    echo "  ✓ Verified: $file"
  fi
done

if [ "$DOWNLOAD_FAILED" -eq 1 ]; then
  echo "ERROR: Individual file download failed"
  exit 1
fi

# Validate JSON files structure
if ! jq -e '.name and .version' "$TEMP_DIR/downloads/skill.json" >/dev/null 2>&1; then
  echo "ERROR: skill.json missing required fields (name, version)"
  exit 1
fi

if ! jq -e '.version and .advisories' "$TEMP_DIR/downloads/feed.json" >/dev/null 2>&1; then
  echo "ERROR: feed.json missing required fields (version, advisories)"
  exit 1
fi

# Install files
echo "Installing from individual files..."
mkdir -p "$INSTALL_DIR"
cp "$TEMP_DIR/downloads"/* "$INSTALL_DIR/"
chmod 600 "$INSTALL_DIR/skill.json"
find "$INSTALL_DIR" -type f ! -name "skill.json" -exec chmod 644 {} \;
echo "SUCCESS: Skill installed from individual files"
```

### Step 3: Integrate into Your Agent's Heartbeat

To ensure continuous monitoring, integrate the OpenClaw Security Feed into your agent's regular heartbeat routine. This involves checking for new security alerts, cross-referencing them with installed skills, and notifying the user if action is required.

```markdown
## OpenClaw Security Feed (every heartbeat)
1. Check advisory feed for new security alerts.
2. Cross-reference with installed skills.
3. Notify user if action is required.
```

### Step 4: Confirm Deployment

Upon successful deployment, inform your user about the activated security monitoring capabilities:

```
📡 OpenClaw Security Feed deployed successfully. I will now:
- Monitor the community advisory feed for threats.
- Alert you to any security issues affecting your environment.
- Keep you informed of emerging attack patterns.
```

---

## What OpenClaw Security Feed Does

### Advisory Feed Monitoring

The OpenClaw Security Feed subscribes to a comprehensive community advisory feed, providing intelligence on various security aspects:

*   **Known Malicious Skills/Plugins:** Identifies skills that have been flagged as harmful or compromised.
*   **Prompt Injection Patterns:** Detects and informs about observed attack patterns related to prompt injection.
*   **Vulnerable Skill Versions:** Alerts on skills with known security flaws, including specific versions.
*   **Security Best Practice Updates:** Provides new recommendations and guidelines for enhancing agent safety and security.

Your agent will receive notifications when a relevant advisory is published, enabling proactive defense.

---

## Checking the Advisory Feed

The advisory feed can be checked using the following command. It prioritizes an environment variable for custom feed URLs, falling back to the raw GitHub feed for the latest updates.

```bash
# Use environment variable if set, otherwise use raw GitHub feed (always up-to-date)
DEFAULT_FEED_URL="https://raw.githubusercontent.com/openclaw-security/OpenClaw-Security/main/advisories/feed.json"
FEED_URL="${OPENCLAW_SECURITY_FEED_URL:-$DEFAULT_FEED_URL}"
curl -s "$FEED_URL" | jq '.advisories[] | select(.severity == "critical" or .severity == "high")'
```

**Canonical Endpoint:** `https://openclaw.security/advisories/feed.json`

### Advisory Categories

| Category | Description |
|---|---|
| `cve` | Common Vulnerabilities and Exposures |
| `prompt_injection` | Prompt injection techniques and patterns |
| `malicious_skill` | Identified malicious or compromised skills |
| `best_practice` | Security recommendations and best practices |

---

## Severity Levels

Advisories are categorized by severity to guide the urgency of response:

| Severity | Action Required |
|---|---|
| `critical` | Immediate notification to the user and prompt action |
| `high` | Notify user soon, plan for remediation |
| `medium` | Notify at the next convenient interaction |
| `low` | Log for reference, notify only if specifically requested |

---

## Prioritizing High-Exploitability Threats

**IMPORTANT:** When reviewing advisories, it is crucial to prioritize by **exploitability score** in addition to severity. The exploitability score indicates the ease with which a vulnerability can be exploited in practice, allowing you to focus on the most actionable threats.

### Exploitability Priority Levels

| Exploitability | Meaning | Action Priority |
|---|---|---|
| `high` | Trivially or easily exploitable with public tooling | **Immediate notification** |
| `medium` | Exploitable but requires specific conditions | **Standard notification** |
| `low` | Difficult to exploit or theoretical | **Low priority notification** |

### How to Use Exploitability in Notifications

1.  **Filter for high-exploitability first:**
    ```bash
    # Get high exploitability advisories
    echo "$FEED" | jq '.advisories[] | select(.exploitability_score == "high")'
    ```

2.  **Include exploitability in notifications:**
    ```
    📡 OpenClaw Security Feed: High-exploitability alert

    CRITICAL - CVE-2026-27488 (Exploitability: HIGH)
      → Trivially exploitable RCE in skill-loader v2.1.0
      → Public exploit code available
      → Recommended action: Immediate removal or upgrade to v2.1.1
    ```

3.  **Prioritize by both severity AND exploitability:** A `HIGH` severity with `HIGH` exploitability CVE is more urgent than a `CRITICAL` severity with `LOW` exploitability CVE. Focus user attention on threats that are both severe and easily exploitable, and include the exploitability rationale to provide context on the risk.

### Example Notification Priority Order

When multiple advisories are present, they should be presented in the following order of priority:

1.  **Critical severity + High exploitability:** These represent the most urgent threats.
2.  **High severity + High exploitability**
3.  **Critical severity + Medium/Low exploitability**
4.  **High severity + Medium/Low exploitability**
5.  **Medium/Low severity:** Any exploitability level.

This prioritization ensures that users are alerted to the most actionable and immediately dangerous threats first.

---

## When to Notify Your User

Notifications should be tailored to the severity and exploitability of the detected threat:

*   **Notify Immediately (Critical):**
    *   New critical advisory affecting an installed skill.
    *   Active exploitation detected.
    *   High exploitability score, regardless of severity.

*   **Notify Soon (High):**
    *   New high-severity advisory affecting installed skills.
    *   Failure to fetch the advisory feed (indicating a potential network issue).
    *   Medium exploitability combined with high severity.

*   **Notify at Next Interaction (Medium):**
    *   New medium-severity advisories.
    *   General security updates.
    *   Low exploitability advisories.

*   **Log Only (Low/Info):**
    *   Low-severity advisories (can be mentioned if the user inquires).
    *   Feed checked with no new alerts.
    *   Theoretical vulnerabilities (low exploitability, low severity).

---

## Response Format

### If there are new advisories:

```
📡 OpenClaw Security Feed: 2 new advisories since last check

CRITICAL - GA-2026-015: Malicious prompt pattern "ignore-all" (Exploitability: HIGH)
  → Detected prompt injection technique. Update your system prompt defenses.
  → Exploitability: Easily exploitable with publicly documented techniques.

HIGH - GA-2026-016: Vulnerable skill "data-helper" v1.2.0 (Exploitability: MEDIUM)
  → You have this installed! Recommended action: Update to v1.2.1 or remove.
  → Exploitability: Requires specific configuration; not trivially exploitable.
```

### If nothing new:

```
FEED_OK - Advisory feed checked, no new alerts. 📡
```

---

## State Tracking

To effectively identify new advisories, the skill tracks its state. This includes the last time the feed was checked, the last time the feed was updated, and a list of known advisories.

```json
{
  "schema_version": "1.0",
  "last_feed_check": "2026-02-02T15:00:00Z",
  "last_feed_updated": "2026-02-02T12:00:00Z",
  "known_advisories": ["GA-2026-001", "GA-2026-002"]
}
```

**Save to:** `~/.openclaw/openclaw-security-feed-state.json`

### State File Operations

```bash
STATE_FILE="$HOME/.openclaw/openclaw-security-feed-state.json"

# Create state file with secure permissions if it doesn't exist
if [ ! -f "$STATE_FILE" ]; then
  echo '{"schema_version":"1.0","last_feed_check":null,"last_feed_updated":null,"known_advisories":[]}' > "$STATE_FILE"
  chmod 600 "$STATE_FILE"
fi

# Validate state file before reading
if ! jq -e '.schema_version' "$STATE_FILE" >/dev/null 2>&1; then
  echo "Warning: State file corrupted or invalid schema. Creating backup and resetting."
  cp "$STATE_FILE" "${STATE_FILE}.bak.$(TZ=UTC date +%Y%m%d%H%M%S)"
  echo '{"schema_version":"1.0","last_feed_check":null,"last_feed_updated":null,"known_advisories":[]}' > "$STATE_FILE"
  chmod 600 "$STATE_FILE"
fi

# Check for major version compatibility
SCHEMA_VER=$(jq -r '.schema_version // "0"' "$STATE_FILE")
if [[ "${SCHEMA_VER%%.*}" != "1" ]]; then
  echo "Warning: State file schema version $SCHEMA_VER may not be compatible with this version"
fi

# Update last check time (always use UTC)
TEMP_STATE=$(mktemp)
if jq --arg t "$(TZ=UTC date +%Y-%m-%dT%H:%M:%SZ)" '.last_feed_check = $t' "$STATE_FILE" > "$TEMP_STATE"; then
  mv "$TEMP_STATE" "$STATE_FILE"
  chmod 600 "$STATE_FILE"
else
  echo "Error: Failed to update state file"
  rm -f "$TEMP_STATE"
fi
```

---

## Rate Limiting

To prevent excessive requests to the advisory feed server, adhere to the following rate-limiting guidelines:

| Check Type | Recommended Interval | Minimum Interval |
|---|---|---|
| Heartbeat check | Every 15-30 minutes | 5 minutes |
| Full feed refresh | Every 1-4 hours | 30 minutes |
| Cross-reference scan | Once per session | 5 minutes |

```bash
# Check if enough time has passed since last check
STATE_FILE="$HOME/.openclaw/openclaw-security-feed-state.json"
MIN_INTERVAL_SECONDS=300  # 5 minutes

LAST_CHECK=$(jq -r '.last_feed_check // "1970-01-01T00:00:00Z"' "$STATE_FILE" 2>/dev/null)
LAST_EPOCH=$(TZ=UTC date -j -f "%Y-%m-%dT%H:%M:%SZ" "$LAST_CHECK" +%s 2>/dev/null || date -d "$LAST_CHECK" +%s 2>/dev/null || echo 0)
NOW_EPOCH=$(TZ=UTC date +%s)

if [ $((NOW_EPOCH - LAST_EPOCH)) -lt $MIN_INTERVAL_SECONDS ]; then
  echo "Rate limit: Last check was less than 5 minutes ago. Skipping."
  exit 0
fi
```

---

## Environment Variables (Optional)

Customization options are available through environment variables:

| Variable | Description | Default |
|---|---|---|
| `OPENCLAW_SECURITY_FEED_URL` | Custom advisory feed URL | Raw GitHub (`main` branch) |
| `OPENCLAW_SECURITY_INSTALL_DIR` | Installation directory | `~/.openclaw/skills/openclaw-security-feed` |

---

## Updating OpenClaw Security Feed

To check for and install newer versions of the OpenClaw Security Feed, use the following commands:

```bash
# Check current installed version
INSTALL_DIR="${OPENCLAW_SECURITY_INSTALL_DIR:-$HOME/.openclaw/skills/openclaw-security-feed}"
CURRENT_VERSION=$(jq -r '.version' "$INSTALL_DIR/skill.json" 2>/dev/null || echo "unknown")
echo "Installed version: $CURRENT_VERSION"

# Check latest available version
LATEST_URL="https://api.github.com/repos/openclaw-security/OpenClaw-Security/releases"
LATEST_VERSION=$(curl -sSL --fail --show-error --retry 3 --retry-delay 1 "$LATEST_URL" 2>/dev/null | \
  jq -r '[.[] | select(.tag_name | startswith("openclaw-security-feed-v"))][0].tag_name // empty' | \
  sed 's/openclaw-security-feed-v//')

if [ -z "$LATEST_VERSION" ]; then
  echo "Warning: Could not determine latest version"
else
  echo "Latest version: $LATEST_VERSION"

  if [ "$CURRENT_VERSION" != "$LATEST_VERSION" ]; then
    echo "Update available! Run the deployment steps with the new version."
  else
    echo "You are running the latest version."
  fi
fi
```

---

## Initial Download Integrity

The initial download of this skill presents a bootstrap trust problem, as it cannot be verified by the skill itself. To establish trust:

1.  **Verify the Source URL:** Ensure you are downloading from `https://openclaw.security` or the official GitHub repository.
2.  **Check Release Signatures:** Verify that the release is signed by OpenClaw Security and compare against published checksums.
3.  **Compare Checksums:** After downloading, compare the SHA-256 hash of the downloaded `SKILL.md` against the published `checksums.json`.

```bash
# After downloading SKILL.md, verify its integrity
EXPECTED_HASH="<hash-from-checksums.json>"
ACTUAL_HASH=$(shasum -a 256 SKILL.md | cut -d' ' -f1)

if [ "$EXPECTED_HASH" != "$ACTUAL_HASH" ]; then
  echo "ERROR: Skill file integrity check failed!"
  echo "This file may have been tampered with. Do not proceed."
  exit 1
fi
```

**Note:** For maximum security, verify `checksums.json` via a separate trusted channel (e.g., directly from the official GitHub release page UI, not via `curl`).

---

## Related Skills

*   **openclaw-audit-watchdog:** Provides automated daily security audits.
*   **openclaw-contributor:** Facilitates reporting vulnerabilities to the community.

---

## License

GNU AGPL v3.0 or later - See repository for details.

Built with 📡 by the OpenClaw Security team and the agent community.
