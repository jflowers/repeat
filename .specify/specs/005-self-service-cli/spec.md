# Feature Specification: Self-Service CLI

**Feature Branch**: `005-self-service-cli`  
**Created**: 2026-02-08  
**Status**: Implemented  
**Input**: Make the CLI self-bootstrapping with actionable errors, eliminating documentation as a dependency for setup

## User Scenarios & Testing *(mandatory)*

### User Story 1 - First Run Setup (Priority: P0)

As a new user who just downloaded the binary, I want `repeat init` to guide me through setup so I don't need to read docs to get started.

**Why this priority**: Eliminates the #1 onboarding friction — reading SETUP.md before doing anything

**Independent Test**: Download a fresh binary, run `repeat init`, follow prompts, then `repeat run --dry-run` succeeds

**Acceptance Scenarios**:

1. **Given** `~/.repeat/` does not exist, **When** I run `repeat init`, **Then** it creates the directory, generates `.env` template, and prompts for `GEMINI_API_KEY`
2. **Given** I provide a valid API key, **When** init continues, **Then** it auto-detects my Chrome profile path and writes it to `.env`
3. **Given** init completes, **When** I run `repeat run --dry-run`, **Then** config loads correctly from `~/.repeat/.env`
4. **Given** `~/.repeat/` already exists, **When** I run `repeat init`, **Then** it skips existing files and only creates missing ones
5. **Given** I pass `--non-interactive`, **When** I run `repeat init`, **Then** it uses defaults without prompting

---

### User Story 2 - Diagnostics (Priority: P0)

As a user encountering errors, I want `repeat doctor` to tell me exactly what's wrong and how to fix it.

**Why this priority**: Most support burden comes from misconfiguration — this eliminates it

**Independent Test**: Remove credentials.json and run `repeat doctor` — verify it reports the missing file with exact fix instructions

**Acceptance Scenarios**:

1. **Given** all prerequisites are met, **When** I run `repeat doctor`, **Then** I see all green checkmarks
2. **Given** credentials.json is missing, **When** I run `repeat doctor`, **Then** it reports the issue with a link to the setup guide
3. **Given** GEMINI_API_KEY is not set, **When** I run `repeat doctor`, **Then** it tells me to set it in `~/.repeat/.env`
4. **Given** Node.js is not installed, **When** I run `repeat doctor`, **Then** it warns that Step 3 (task assignment) will be unavailable
5. **Given** the OAuth token is expired, **When** I run `repeat doctor`, **Then** it tells me to run `repeat auth login`

---

### User Story 3 - Service Install via CLI (Priority: P1)

As a user, I want `repeat install` to set up the hourly service without needing Make.

**Why this priority**: Makefile is a developer tool; end users should use the binary

**Independent Test**: Run `repeat install`, verify the service is active with `repeat doctor`

**Acceptance Scenarios**:

1. **Given** I'm on macOS, **When** I run `repeat install`, **Then** it installs the LaunchAgent and starts the service
2. **Given** I'm on Linux, **When** I run `repeat install`, **Then** it installs the systemd user timer
3. **Given** the service is already installed, **When** I run `repeat install`, **Then** it is idempotent (reinstalls cleanly)
4. **Given** I want to remove it, **When** I run `repeat uninstall`, **Then** it stops and removes all service files

---

### User Story 4 - Actionable Errors (Priority: P0)

As a user hitting an error, I want the error message to tell me exactly what to do so I can fix it myself.

**Why this priority**: Every confusing error is a potential user abandonment

**Acceptance Scenarios**:

1. **Given** credentials.json is missing, **When** any command runs, **Then** the error includes the fix: `Run 'repeat init' or download from https://console.cloud.google.com/apis/credentials`
2. **Given** GEMINI_API_KEY is empty, **When** any command runs, **Then** the error includes: `Set GEMINI_API_KEY in ~/.repeat/.env or run 'repeat init'`
3. **Given** Node.js is not found, **When** assign-tasks runs, **Then** the error includes: `Install Node.js 18+ from https://nodejs.org`
4. **Given** OAuth token is expired, **When** any API call fails, **Then** the error includes: `Run 'repeat auth login' to re-authenticate`
5. **Given** Chrome profile is not found, **When** assign-tasks runs, **Then** the error includes: `Set CHROME_PROFILE_PATH. Find yours at chrome://version`

---

### Edge Cases

- What if `init` is interrupted mid-way?
  → Partial files are left in place; re-running `init` fills in the gaps (idempotent)
- What if the user runs `install` before `init`?
  → `install` checks prerequisites and runs `init` first if needed
- What if Node.js is installed after initial setup?
  → `doctor` will detect it; no reconfig needed since Steps 1-2 work without it
- What if the config directory has wrong permissions?
  → `doctor` checks and reports with fix: `chmod 700 ~/.repeat`

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `repeat init` MUST create `~/.repeat/` and generate a `.env` file
- **FR-002**: `repeat init` MUST prompt for GEMINI_API_KEY interactively (or accept `--api-key` flag)
- **FR-003**: `repeat init` MUST set `CHROME_PROFILE_PATH` to a dedicated `repeat` Chrome profile directory for the current OS
- **FR-004**: `repeat init` MUST support `--non-interactive` flag for scripted use
- **FR-005**: `repeat doctor` MUST check: config dir, credentials, token validity, API key, Node.js, Chrome profile, service status
- **FR-006**: `repeat doctor` MUST output actionable fix commands for each issue
- **FR-007**: `repeat install` MUST detect OS and install the appropriate service (launchd/systemd)
- **FR-008**: `repeat install` MUST be idempotent
- **FR-009**: `repeat uninstall` MUST stop and remove all service files
- **FR-010**: All error messages MUST include a `Fix:` line with exact resolution steps
- **FR-011**: `repeat init` MUST be idempotent — skip existing files, fill in missing ones
- **FR-012**: Self-service commands SHOULD use `charmbracelet/huh` for interactive prompts (API key, Chrome profile selection)
- **FR-013**: Self-service commands SHOULD use `charmbracelet/lipgloss` for styled terminal output
- **FR-014**: User-facing error messages MUST include a reference to `repeat doctor` for diagnostics
- **FR-015**: `repeat setup-browser` MUST create/use a dedicated `repeat` Chrome profile, launch Chrome with `--remote-debugging-port=9222`, guide first-run Google sign-in, and verify CDP connectivity
- **FR-016**: `doctor` and `init` MUST use `charmbracelet/lipgloss` styled summary boxes for polished output

### Non-Functional Requirements

- **NFR-001**: `init` completes in under 2 seconds (excluding OAuth flow)
- **NFR-002**: `doctor` completes in under 3 seconds
- **NFR-003**: Error messages use consistent formatting with emoji indicators (✅ ⚠️ ❌)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can go from binary download to first successful `run --dry-run` using only CLI commands (no docs)
- **SC-002**: `repeat doctor` identifies 100% of common misconfiguration issues
- **SC-003**: Every error reachable by end users includes an actionable fix
- **SC-004**: Service install/uninstall works without Makefile or build tools
