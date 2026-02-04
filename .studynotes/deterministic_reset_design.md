# Deterministic Environment Reset - Design Document

**Purpose**: Enable reproducible RL training by providing deterministic environment initialization and reset across episodes.

**Critical Requirement**: Given the same task and seed, environment reset must produce **identical** initial states, leading to **identical** behavior for identical action sequences.

---

## Table of Contents

1. [Why Determinism Matters](#why-determinism-matters)
2. [Scope of Determinism](#scope-of-determinism)
3. [Architecture Overview](#architecture-overview)
4. [State Components](#state-components)
5. [Tiered Reset Strategy](#tiered-reset-strategy)
6. [Account Management](#account-management)
7. [Implementation Guide](#implementation-guide)
8. [Verification & Testing](#verification-testing)
9. [Troubleshooting](#troubleshooting)

---

## Why Determinism Matters

### For RL Training

**Reproducible Experiments**:

- Same initial state → same reward for same actions
- Can replay episodes exactly
- Debug agent failures reliably
- Compare algorithm performance fairly

**Debugging**:

- If episode fails, can replay with logging
- Identify exact step where agent diverges
- Test fixes deterministically

**Offline Training**:

- Recorded trajectories remain valid
- Can re-evaluate old trajectories with new reward functions
- Enable counterfactual analysis

### For Development

**Testing**:

- Unit tests produce consistent results
- Can assert exact outcomes
- Catch regressions reliably

**Benchmarking**:

- Fair comparison across runs
- Measure true performance improvements
- Eliminate noise from environment variance

---

## Scope of Determinism

### What MUST Be Deterministic

1. **Initial State**
   - Browser: cookies, localStorage, session storage
   - File system: files, directories, permissions
   - Databases: all rows, schemas, indexes
   - Environment variables
   - System time (for time-dependent tasks)

2. **Tool Execution**
   - File operations (read/write order, timestamps)
   - Database queries (if using LIMIT without ORDER BY, must add ORDER BY)
   - Random number generation (seeded)

3. **Observations**
   - Browser snapshots (if DOM order matters, sort by stable criteria)
   - File listings (sorted consistently)
   - Tool results (consistent formatting)

### What CAN Be Non-Deterministic (With Mitigation)

1. **External Websites**
   - Library website may change content
   - **Mitigation**:
     - Snapshot expected pages locally
     - Or accept non-determinism, focus on agent robustness
     - Use test environments with controlled content

2. **Real-Time Services**
   - Telegram/Slack message IDs change
   - **Mitigation**:
     - Use test workspaces, not production
     - Normalize message IDs in observations
     - Focus on content, not IDs

3. **Timestamps**
   - System time advances
   - **Mitigation**:
     - Mock time for time-sensitive tasks
     - Or normalize timestamps in observations

### Design Principle

> **"Deterministic where it matters, robust to non-determinism where it doesn't"**

For example:

- Browser element order (deterministic) matters for snapshots
- Exact timestamp (non-deterministic) doesn't matter for most tasks

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Episode N                                              │
│  - Execute actions                                      │
│  - Observe outcomes                                     │
│  - Accumulate rewards                                   │
└─────────────────┬───────────────────────────────────────┘
                  │ Episode ends
                  ▼
         ┌────────────────┐
         │ Reset Required │
         └────────┬───────┘
                  │
        ┌─────────▼──────────┐
        │ Choose Reset Level │
        │ (1, 2, or 3)       │
        └─────────┬──────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌───────┐    ┌─────────┐   ┌────────┐
│Level 1│    │Level 2  │   │Level 3 │
│Memory │    │Snapshot │   │Full    │
│< 1s   │    │< 5s     │   │< 30s   │
└───┬───┘    └────┬────┘   └───┬────┘
    │             │            │
    └─────────────┼────────────┘
                  │
                  ▼
         ┌────────────────┐
         │ Verify State   │
         │ (Determinism   │
         │  check)        │
         └────────┬───────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│  Episode N+1                                            │
│  - Same initial state as Episode N (or previous)       │
└─────────────────────────────────────────────────────────┘
```

---

## State Components

### 1. Browser State

**What Needs to Be Deterministic**:

```yaml
browser_state:
  profile_name: rl-saratoga-library

  # Session state
  cookies:
    - name: session_id
      value: abc123
      domain: sccld.org
      path: /
      secure: true
      httpOnly: true
      sameSite: Lax
      expirationDate: 1735689600 # Fixed, not relative

  localStorage:
    theme: dark
    language: en

  sessionStorage:
    currentView: dashboard

  # Navigation state
  current_url: https://sccld.org/dashboard

  # User agent (some sites behave differently)
  user_agent: "Mozilla/5.0 ... (fixed)"

  # Viewport size (affects rendering)
  viewport:
    width: 1280
    height: 720
```

**Implementation**:

```typescript
interface BrowserState {
  profileName: string;
  cookies: Cookie[];
  localStorage: Record<string, string>;
  sessionStorage: Record<string, string>;
  currentUrl: string;
  userAgent: string;
  viewport: { width: number; height: number };
}

async function captureBrowserState(profile: string): Promise<BrowserState> {
  await browserStart({ profile });

  return {
    profileName: profile,
    cookies: await browserGetCookies(),
    localStorage: await browserGetLocalStorage(),
    sessionStorage: await browserGetSessionStorage(),
    currentUrl: await browserGetUrl(),
    userAgent: await browserGetUserAgent(),
    viewport: await browserGetViewport(),
  };
}

async function restoreBrowserState(state: BrowserState): Promise<void> {
  // 1. Set user agent BEFORE starting browser
  await browserSetUserAgent(state.userAgent);

  // 2. Start with clean profile
  await browserStart({ profile: state.profileName, clean: true });

  // 3. Set viewport
  await browserSetViewport(state.viewport);

  // 4. Restore cookies (must do before navigation)
  for (const cookie of state.cookies) {
    await browserSetCookie(cookie);
  }

  // 5. Navigate to starting URL
  await browserNavigate(state.currentUrl);

  // 6. Restore storage (after navigation)
  await browserSetLocalStorage(state.localStorage);
  await browserSetSessionStorage(state.sessionStorage);
}
```

**Snapshot Storage**:

```typescript
async function saveBrowserSnapshot(profileName: string, snapshotId: string): Promise<void> {
  const state = await captureBrowserState(profileName);

  // Save as JSON for human readability and git-friendliness
  const snapshotPath = path.join(config.rl.snapshotsDir, "browser", `${snapshotId}.json`);

  await fs.writeFile(snapshotPath, JSON.stringify(state, null, 2));

  // Also save browser profile directory (more complete)
  const profilePath = getBrowserProfilePath(profileName);
  const profileSnapshotPath = path.join(
    config.rl.snapshotsDir,
    "browser-profiles",
    `${snapshotId}.tar.gz`,
  );

  await exec(`tar -czf "${profileSnapshotPath}" -C "${profilePath}" .`);
}
```

---

### 2. File System State

**What Needs to Be Deterministic**:

```yaml
filesystem_state:
  workspace_dir: /tmp/rl-env-workspace-{episode_id}

  files:
    - path: data/users.csv
      content: "id,name,email\n1,Alice,alice@example.com\n"
      permissions: 0644
      mtime: 1704067200 # Fixed timestamp

  directories:
    - path: reports/
      permissions: 0755
      mtime: 1704067200

  # Symlinks (if any)
  symlinks:
    - source: data/latest.csv
      target: data/users.csv
```

**Implementation**:

```typescript
interface FileSystemState {
  workspaceDir: string;
  files: FileEntry[];
  directories: DirectoryEntry[];
  symlinks: SymlinkEntry[];
}

async function initializeFileSystem(state: FileSystemState): Promise<void> {
  // 1. Create clean workspace
  await fs.rm(state.workspaceDir, { recursive: true, force: true });
  await fs.mkdir(state.workspaceDir, { recursive: true });

  // 2. Create directories (sorted by depth to handle nested dirs)
  const sortedDirs = state.directories.sort(
    (a, b) => a.path.split("/").length - b.path.split("/").length,
  );

  for (const dir of sortedDirs) {
    const fullPath = path.join(state.workspaceDir, dir.path);
    await fs.mkdir(fullPath, { mode: dir.permissions });

    // Set mtime (for determinism)
    await fs.utimes(fullPath, dir.mtime, dir.mtime);
  }

  // 3. Create files
  for (const file of state.files) {
    const fullPath = path.join(state.workspaceDir, file.path);
    await fs.writeFile(fullPath, file.content, { mode: file.permissions });
    await fs.utimes(fullPath, file.mtime, file.mtime);
  }

  // 4. Create symlinks
  for (const link of state.symlinks) {
    const sourcePath = path.join(state.workspaceDir, link.source);
    await fs.symlink(link.target, sourcePath);
  }
}
```

**Filesystem Overlay (Alternative for Fast Reset)**:

```typescript
// Use OverlayFS for copy-on-write filesystem
// Changes are ephemeral, reset just unmounts overlay

async function setupFilesystemOverlay(baseDir: string): Promise<string> {
  const overlayDir = `/tmp/rl-overlay-${randomId()}`;
  const upperDir = path.join(overlayDir, "upper");
  const workDir = path.join(overlayDir, "work");
  const mergedDir = path.join(overlayDir, "merged");

  await fs.mkdir(upperDir, { recursive: true });
  await fs.mkdir(workDir, { recursive: true });
  await fs.mkdir(mergedDir, { recursive: true });

  // Mount overlay
  await exec(`
    sudo mount -t overlay overlay \
      -o lowerdir=${baseDir},upperdir=${upperDir},workdir=${workDir} \
      ${mergedDir}
  `);

  return mergedDir; // Use this as workspace
}

async function resetFilesystemOverlay(overlayDir: string): Promise<void> {
  // Just unmount and remount (very fast)
  const mergedDir = path.join(overlayDir, "merged");
  await exec(`sudo umount ${mergedDir}`);

  // Clear upper and work dirs
  const upperDir = path.join(overlayDir, "upper");
  const workDir = path.join(overlayDir, "work");
  await fs.rm(upperDir, { recursive: true, force: true });
  await fs.rm(workDir, { recursive: true, force: true });
  await fs.mkdir(upperDir);
  await fs.mkdir(workDir);

  // Remount
  await exec(`
    sudo mount -t overlay overlay \
      -o lowerdir=${baseDir},upperdir=${upperDir},workdir=${workDir} \
      ${mergedDir}
  `);
}
```

---

### 3. Database State

**What Needs to Be Deterministic**:

```yaml
database_state:
  type: sqlite
  name: library_catalog

  schema:
    - table: books
      columns:
        - name: id
          type: INTEGER PRIMARY KEY
        - name: title
          type: TEXT NOT NULL
        - name: author
          type: TEXT
        - name: available
          type: BOOLEAN DEFAULT 1

  data:
    - table: books
      rows:
        - { id: 1, title: "The Pragmatic Programmer", author: "Hunt & Thomas", available: 1 }
        - { id: 2, title: "Clean Code", author: "Robert Martin", available: 0 }

  # Important: Row order matters for determinism
  # Always use ORDER BY in queries, or store with explicit order
```

**Implementation**:

```typescript
interface DatabaseState {
  type: "sqlite" | "postgres" | "mongodb";
  name: string;
  seedFile: string; // SQL file with INSERT statements
}

async function seedDatabase(state: DatabaseState): Promise<void> {
  const dbPath = getDatabasePath(state.name);

  // 1. Delete existing database
  await fs.rm(dbPath, { force: true });

  // 2. Create fresh database
  const db = await openDatabase(dbPath);

  // 3. Execute seed file
  const seedSQL = await fs.readFile(state.seedFile, "utf-8");
  await db.exec(seedSQL);

  // 4. Verify determinism: compute checksum
  const checksum = await computeDatabaseChecksum(db);
  console.log(`Database seeded: ${state.name}, checksum: ${checksum}`);

  await db.close();
}

async function computeDatabaseChecksum(db: Database): Promise<string> {
  // Get all table data in deterministic order
  const tables = await db.all("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name");

  let combined = "";
  for (const table of tables) {
    const rows = await db.all(`SELECT * FROM ${table.name} ORDER BY rowid`);
    combined += JSON.stringify(rows);
  }

  return crypto.createHash("sha256").update(combined).digest("hex");
}
```

**Database Snapshot (Fast Reset)**:

```typescript
async function snapshotDatabase(name: string): Promise<void> {
  const dbPath = getDatabasePath(name);
  const snapshotPath = path.join(config.rl.snapshotsDir, "databases", `${name}-initial.db`);

  // SQLite: simple file copy
  await fs.copyFile(dbPath, snapshotPath);

  // Also store checksum for verification
  const db = await openDatabase(snapshotPath);
  const checksum = await computeDatabaseChecksum(db);
  await db.close();

  await fs.writeFile(`${snapshotPath}.checksum`, checksum);
}

async function restoreDatabaseSnapshot(name: string): Promise<void> {
  const dbPath = getDatabasePath(name);
  const snapshotPath = path.join(config.rl.snapshotsDir, "databases", `${name}-initial.db`);

  // Copy snapshot to active location
  await fs.copyFile(snapshotPath, dbPath);

  // Verify checksum
  const db = await openDatabase(dbPath);
  const actualChecksum = await computeDatabaseChecksum(db);
  await db.close();

  const expectedChecksum = await fs.readFile(`${snapshotPath}.checksum`, "utf-8");

  if (actualChecksum !== expectedChecksum) {
    throw new Error(
      `Database checksum mismatch! Expected ${expectedChecksum}, got ${actualChecksum}`,
    );
  }
}
```

---

### 4. Random Seed Management

**Why Random Seeds Matter**:

Some tasks may involve randomness:

- Selecting random sample from dataset
- Generating test data
- Simulating user behavior

**Implementation**:

```typescript
interface EpisodeConfig {
  taskId: string;
  seed: number; // Deterministic seed for this episode
}

class SeededRandom {
  private rng: () => number;

  constructor(seed: number) {
    // Use seedrandom library for deterministic PRNG
    this.rng = seedrandom(seed.toString());
  }

  random(): number {
    return this.rng();
  }

  randomInt(min: number, max: number): number {
    return Math.floor(this.random() * (max - min + 1)) + min;
  }

  choice<T>(array: T[]): T {
    return array[this.randomInt(0, array.length - 1)];
  }

  shuffle<T>(array: T[]): T[] {
    const result = [...array];
    for (let i = result.length - 1; i > 0; i--) {
      const j = this.randomInt(0, i);
      [result[i], result[j]] = [result[j], result[i]];
    }
    return result;
  }
}

// Usage in episode
const episodeRandom = new SeededRandom(episode.seed);
const randomItem = episodeRandom.choice(["apple", "banana", "cherry"]);
```

---

## Tiered Reset Strategy

### Level 1: Memory Reset (< 1s)

**What It Resets**:

- Agent session state (conversation history)
- Episode tracking variables (step count, actions, observations)
- In-memory caches

**What It Doesn't Reset**:

- Browser state (assumes session still valid)
- File system (assumes no changes)
- Databases (assumes no changes)

**When to Use**:

- Agent didn't modify any persistent state
- Read-only tasks
- Failed early (before any tool calls)

**Implementation**:

```typescript
async function resetLevel1(): Promise<void> {
  // 1. Clear agent session
  await agentSession.reset();

  // 2. Reset episode state
  episodeState = {
    taskId: episode.taskId,
    seed: episode.seed,
    step: 0,
    startTime: Date.now(),
    actions: [],
    observations: [],
    rewards: [],
  };

  // 3. Re-initialize random seed
  episodeRandom = new SeededRandom(episode.seed);

  // 4. Clear any in-memory caches
  toolResultCache.clear();
}
```

---

### Level 2: Snapshot Restore (< 5s)

**What It Resets**:

- Everything from Level 1, plus:
- Browser state (from snapshot)
- File system (remount overlay or restore from snapshot)
- Databases (copy from snapshot)

**When to Use**:

- Agent made changes to browser/files/DB
- Default reset level for most episodes
- Session still valid (not expired)

**Implementation**:

```typescript
async function resetLevel2(task: Task): Promise<void> {
  // 1. Memory reset
  await resetLevel1();

  // 2. Restore browser state
  for (const account of task.accountsRequired) {
    if (account.type === "browser_login") {
      const snapshotId = `${account.service}-logged-in`;

      try {
        await restoreBrowserSnapshot(snapshotId);
      } catch (error) {
        if (error instanceof SnapshotExpiredError) {
          // Session expired, need full reset
          throw new NeedFullResetError(`Browser session expired for ${account.service}`);
        }
        throw error;
      }
    }
  }

  // 3. Restore file system
  if (config.rl.filesystem.useOverlay) {
    // Fast: just remount overlay
    await resetFilesystemOverlay(task.workspaceDir);
  } else {
    // Slower: restore from snapshot
    await restoreFilesystemSnapshot(task.id);
  }

  // 4. Restore databases
  for (const db of task.environment.databases || []) {
    await restoreDatabaseSnapshot(db.name);
  }

  // 5. Verify state (checksum validation)
  await verifyEnvironmentState(task);
}
```

---

### Level 3: Full Reset (< 30s)

**What It Resets**:

- Everything from Level 2, plus:
- Re-perform browser logins (fresh sessions)
- Re-seed databases from source
- Re-create file system from spec

**When to Use**:

- Browser sessions expired
- Snapshot corrupted
- Periodic cleanup (every N episodes)
- First reset for a new task

**Implementation**:

```typescript
async function resetLevel3(task: Task): Promise<void> {
  // 1. Memory reset
  await resetLevel1();

  // 2. Re-perform browser logins
  for (const account of task.accountsRequired) {
    if (account.type === "browser_login") {
      const credential = await getCredential(account.id);
      const loginFlow = loginFlows[account.service];

      await performBrowserLogin({
        profileName: account.service,
        url: loginFlow.url,
        credential,
        steps: loginFlow.steps,
      });

      // Save new snapshot for future Level 2 resets
      await saveBrowserSnapshot(account.service, `${account.service}-logged-in`);
    }
  }

  // 3. Re-initialize file system
  await initializeFileSystem(task.environment.filesystem);

  // 4. Re-seed databases
  for (const db of task.environment.databases || []) {
    await seedDatabase(db);

    // Save snapshot for future Level 2 resets
    await snapshotDatabase(db.name);
  }

  // 5. Verify state
  await verifyEnvironmentState(task);
}
```

---

### Reset Level Selection Logic

```typescript
class ResetController {
  private consecutiveLevel2Resets = 0;
  private readonly FULL_RESET_INTERVAL = 100; // Every 100 episodes

  async reset(task: Task, episodeNumber: number): Promise<void> {
    let level: ResetLevel;

    // Decide reset level
    if (episodeNumber % this.FULL_RESET_INTERVAL === 0) {
      // Periodic full reset for cleanup
      level = ResetLevel.LEVEL3;
      this.consecutiveLevel2Resets = 0;
    } else if (this.consecutiveLevel2Resets === 0) {
      // First reset for this task
      level = ResetLevel.LEVEL3;
    } else {
      // Try Level 2 (snapshot restore)
      level = ResetLevel.LEVEL2;
    }

    try {
      switch (level) {
        case ResetLevel.LEVEL1:
          await resetLevel1();
          break;

        case ResetLevel.LEVEL2:
          await resetLevel2(task);
          this.consecutiveLevel2Resets++;
          break;

        case ResetLevel.LEVEL3:
          await resetLevel3(task);
          this.consecutiveLevel2Resets = 1; // Next can be Level 2
          break;
      }

      await logResetMetrics(level, task.id);
    } catch (error) {
      if (error instanceof NeedFullResetError) {
        // Level 2 failed, fallback to Level 3
        console.warn(`Level 2 reset failed: ${error.message}, falling back to Level 3`);
        await resetLevel3(task);
        this.consecutiveLevel2Resets = 1;
      } else {
        throw error;
      }
    }
  }
}
```

---

## Account Management

### Credential Storage

**Recommended: 1Password CLI**

```yaml
# config/rl-accounts.yaml
accounts:
  sccld-library:
    id: sccld-library
    service: saratoga_library
    type: browser_login
    credential_source: 1password
    vault: Private
    item: "SCCLD Library - RL Test"
    fields:
      username: username
      password: password
```

**Credential Retrieval**:

```typescript
async function getCredential(accountId: string): Promise<Credential> {
  const accountConfig = config.rlAccounts[accountId];

  if (accountConfig.credential_source === "1password") {
    // Fetch from 1Password CLI
    const vault = accountConfig.vault;
    const item = accountConfig.item;

    const username = await execSync(
      `op read "op://${vault}/${item}/${accountConfig.fields.username}"`,
      { encoding: "utf-8" },
    ).trim();

    const password = await execSync(
      `op read "op://${vault}/${item}/${accountConfig.fields.password}"`,
      { encoding: "utf-8" },
    ).trim();

    return { username, password };
  }

  throw new Error(`Unknown credential source: ${accountConfig.credential_source}`);
}
```

---

### Login Flow Automation

**Login Flow Definition**:

```yaml
# config/rl-login-flows.yaml
login_flows:
  saratoga_library:
    service: saratoga_library
    url: https://sccld.org

    steps:
      - type: navigate
        url: https://sccld.org/user/login
        wait_for: networkidle

      - type: fill
        selector: input[name="username"]
        field: username

      - type: fill
        selector: input[name="password"]
        field: password

      - type: click
        selector: button[type="submit"]

      - type: wait
        condition:
          type: url_pattern
          pattern: "**/dashboard"
        timeout_ms: 10000

    verification:
      type: browser_element
      selector: .user-profile
      text_contains: "Logged in"

    snapshot_after_login: true
```

**Login Execution**:

```typescript
interface LoginStep {
  type: "navigate" | "fill" | "click" | "wait";
  // ... specific fields per type
}

async function performBrowserLogin(params: {
  profileName: string;
  url: string;
  credential: Credential;
  steps: LoginStep[];
}): Promise<void> {
  const { profileName, credential, steps } = params;

  // Start browser with fresh profile
  await browserStart({
    profile: profileName,
    headless: config.rl.browser.headless,
  });

  // Execute login steps
  for (const step of steps) {
    switch (step.type) {
      case "navigate":
        await browserNavigate(step.url);
        if (step.wait_for === "networkidle") {
          await browserWait({ load: "networkidle" });
        }
        break;

      case "fill":
        const value = step.field === "username" ? credential.username : credential.password;
        await browserType(step.selector, value);
        break;

      case "click":
        await browserClick(step.selector);
        break;

      case "wait":
        if (step.condition.type === "url_pattern") {
          await browserWait({
            url: step.condition.pattern,
            timeout_ms: step.timeout_ms,
          });
        }
        break;
    }
  }

  // Verify login succeeded
  const loginSuccess = await verifyLogin(params.verification);

  if (!loginSuccess) {
    throw new LoginFailedError(`Login verification failed for ${profileName}`);
  }
}

async function verifyLogin(verification: {
  type: "browser_element";
  selector: string;
  text_contains?: string;
}): Promise<boolean> {
  try {
    const snapshot = await browserSnapshot({ interactive: true });

    // Check if element exists and contains expected text
    const element = findElementInSnapshot(snapshot, verification.selector);

    if (!element) {
      return false;
    }

    if (verification.text_contains) {
      return element.text.includes(verification.text_contains);
    }

    return true;
  } catch {
    return false;
  }
}
```

---

### Session Expiration Handling

```typescript
async function checkBrowserSessionValid(profileName: string, verification: any): Promise<boolean> {
  try {
    await browserStart({ profile: profileName });
    const isValid = await verifyLogin(verification);
    await browserStop();
    return isValid;
  } catch {
    return false;
  }
}

async function restoreBrowserSnapshotWithValidation(
  profileName: string,
  loginFlow: LoginFlow,
): Promise<void> {
  const snapshotId = `${profileName}-logged-in`;

  // Restore snapshot
  await restoreBrowserSnapshot(snapshotId);

  // Verify session still valid
  const isValid = await checkBrowserSessionValid(profileName, loginFlow.verification);

  if (!isValid) {
    // Session expired, re-login needed
    throw new SnapshotExpiredError(`Browser session expired for ${profileName}, re-login required`);
  }
}
```

---

## Implementation Guide

### Week 1: Foundation

**Day 1-2: Credential Management**

```typescript
// src/rl/accounts/credential-loader.ts

export interface Credential {
  username: string;
  password: string;
}

export async function getCredential(accountId: string): Promise<Credential> {
  // TODO: Implement credential retrieval
  // - Support 1Password CLI
  // - Fallback to environment variables
  // - Cache credentials in memory (security: only for duration of setup)
}
```

**Day 3-4: Browser Login Automation**

```typescript
// src/rl/accounts/browser-login-manager.ts

export interface LoginFlow {
  service: string;
  url: string;
  steps: LoginStep[];
  verification: VerificationRule;
}

export async function performBrowserLogin(
  profileName: string,
  loginFlow: LoginFlow,
  credential: Credential,
): Promise<void> {
  // TODO: Execute login steps
  // TODO: Verify login success
  // TODO: Handle errors (wrong password, CAPTCHA, etc.)
}
```

**Day 5: One-Time Setup Script**

```typescript
// scripts/setup-rl-accounts.ts

async function setupAccount(accountId: string): Promise<void> {
  console.log(`Setting up account: ${accountId}...`);

  // 1. Load config
  const accountConfig = config.rlAccounts[accountId];
  const loginFlow = config.rlLoginFlows[accountConfig.service];

  // 2. Get credential
  const credential = await getCredential(accountId);

  // 3. Perform login
  await performBrowserLogin(accountConfig.service, loginFlow, credential);

  // 4. Save snapshot
  await saveBrowserSnapshot(accountConfig.service, `${accountConfig.service}-logged-in`);

  console.log(`✓ Account ${accountId} set up successfully`);
}

async function main() {
  const accountIds = Object.keys(config.rlAccounts);

  for (const accountId of accountIds) {
    await setupAccount(accountId);
  }

  console.log("\n✓ All accounts set up!");
}

main();
```

---

### Week 2: State Snapshots

**Day 1-2: Browser Snapshots**

```typescript
// src/rl/state/browser-snapshots.ts

export async function saveBrowserSnapshot(profileName: string, snapshotId: string): Promise<void> {
  // TODO: Capture browser state (cookies, storage, etc.)
  // TODO: Save as JSON
  // TODO: Also save profile directory as tar.gz
}

export async function restoreBrowserSnapshot(snapshotId: string): Promise<void> {
  // TODO: Load snapshot JSON
  // TODO: Restore cookies, storage
  // TODO: Or restore from tar.gz
  // TODO: Verify restoration success
}
```

**Day 3: Database Snapshots**

```typescript
// src/rl/state/database-snapshots.ts

export async function snapshotDatabase(name: string): Promise<void> {
  // TODO: Copy database file
  // TODO: Compute checksum
  // TODO: Store metadata
}

export async function restoreDatabaseSnapshot(name: string): Promise<void> {
  // TODO: Copy snapshot to active location
  // TODO: Verify checksum
}
```

**Day 4-5: File System Snapshots**

```typescript
// src/rl/state/filesystem-snapshots.ts

export async function initializeFileSystem(spec: FileSystemSpec): Promise<void> {
  // TODO: Create directories
  // TODO: Create files with fixed timestamps
  // TODO: Create symlinks
}

export async function setupFilesystemOverlay(baseDir: string): Promise<string> {
  // TODO: Create overlay directories
  // TODO: Mount overlayfs
  // TODO: Return merged directory path
}

export async function resetFilesystemOverlay(overlayDir: string): Promise<void> {
  // TODO: Unmount
  // TODO: Clear upper/work dirs
  // TODO: Remount
}
```

---

### Week 3: Reset Controller

**Day 1-3: Implement Reset Levels**

```typescript
// src/rl/state/reset-controller.ts

export class ResetController {
  async reset(task: Task, episodeNumber: number): Promise<void> {
    // TODO: Implement reset level selection logic
    // TODO: Implement Level 1 reset (memory)
    // TODO: Implement Level 2 reset (snapshots)
    // TODO: Implement Level 3 reset (full)
    // TODO: Fallback logic (Level 2 → Level 3)
  }
}
```

**Day 4-5: Verification & Testing**

```typescript
// src/rl/state/verification.ts

export async function verifyEnvironmentState(task: Task): Promise<void> {
  // TODO: Verify browser state
  // TODO: Verify file system state
  // TODO: Verify database state
  // TODO: Throw error if verification fails
}
```

---

## Verification & Testing

### Determinism Test

**Test Procedure**:

```typescript
// tests/rl/determinism.test.ts

describe("Environment Determinism", () => {
  test("identical action sequences produce identical rewards", async () => {
    const task = await loadTask("library-room-booking");
    const seed = 12345;

    // Run episode 1
    await resetController.reset(task, seed);
    const actions1 = [
      { tool: "browser", params: { action: "open", url: "..." } },
      { tool: "browser", params: { action: "snapshot" } },
      // ... more actions
    ];
    const rewards1 = await runEpisode(task, actions1);

    // Run episode 2 (same seed, same actions)
    await resetController.reset(task, seed);
    const rewards2 = await runEpisode(task, actions2);

    // Verify identical rewards
    expect(rewards2).toEqual(rewards1);
  });

  test("browser snapshot restore produces identical state", async () => {
    const profileName = "test-profile";

    // Capture initial state
    const state1 = await captureBrowserState(profileName);

    // Make some changes
    await browserNavigate("https://example.com");
    await browserSetCookie({ name: "test", value: "modified" });

    // Restore snapshot
    await restoreBrowserState(state1);

    // Capture state again
    const state2 = await captureBrowserState(profileName);

    // Verify identical
    expect(state2).toEqual(state1);
  });

  test("database snapshot restore produces identical checksum", async () => {
    const dbName = "test-db";

    // Seed database
    await seedDatabase({ name: dbName, seedFile: "test-seed.sql" });
    const checksum1 = await computeDatabaseChecksum(dbName);

    // Modify database
    const db = await openDatabase(getDatabasePath(dbName));
    await db.run('INSERT INTO books VALUES (999, "Test Book", "Test Author", 1)');
    await db.close();

    // Restore snapshot
    await restoreDatabaseSnapshot(dbName);
    const checksum2 = await computeDatabaseChecksum(dbName);

    // Verify identical
    expect(checksum2).toEqual(checksum1);
  });
});
```

### Reset Performance Test

```typescript
describe("Reset Performance", () => {
  test("Level 1 reset completes in < 1 second", async () => {
    const start = Date.now();
    await resetLevel1();
    const duration = Date.now() - start;

    expect(duration).toBeLessThan(1000);
  });

  test("Level 2 reset completes in < 5 seconds", async () => {
    const task = await loadTask("library-room-booking");
    const start = Date.now();
    await resetLevel2(task);
    const duration = Date.now() - start;

    expect(duration).toBeLessThan(5000);
  });

  test("Level 3 reset completes in < 30 seconds", async () => {
    const task = await loadTask("library-room-booking");
    const start = Date.now();
    await resetLevel3(task);
    const duration = Date.now() - start;

    expect(duration).toBeLessThan(30000);
  });
});
```

---

## Troubleshooting

### Issue: Non-Deterministic Observations

**Symptom**: Same actions produce different observations across episodes

**Causes**:

1. Timestamps in observations (e.g., browser snapshot includes time)
2. Unordered data structures (e.g., file listings without sort)
3. Random IDs in UI (e.g., element IDs generated randomly)

**Solutions**:

1. Normalize timestamps (replace with fixed value or relative time)
2. Always sort data structures before observations
3. Use stable selectors (CSS classes, not random IDs)

### Issue: Browser Session Expires

**Symptom**: Level 2 reset fails, "Login required" message appears

**Causes**:

1. Session cookie expired
2. Server-side session timeout
3. IP address changed (if using proxy/VPN)

**Solutions**:

1. Reduce session expiration time in snapshot metadata
2. Fallback to Level 3 reset when session invalid
3. Implement periodic re-login (every N episodes)

### Issue: Database Checksum Mismatch

**Symptom**: Checksum verification fails after restore

**Causes**:

1. Auto-incrementing IDs (non-deterministic)
2. Timestamp columns with DEFAULT CURRENT_TIMESTAMP
3. Concurrent access (if multiple environments share DB)

**Solutions**:

1. Use fixed IDs in seed data
2. Disable auto-timestamp columns in test DB
3. Ensure DB isolation (one DB per environment)

---

## Summary

### Key Principles

1. **Deterministic Initialization**: Same task + seed → identical initial state
2. **Tiered Reset**: Balance speed (Level 1) vs. thoroughness (Level 3)
3. **Snapshot Management**: Cache logged-in states for fast restore
4. **Verification**: Always verify state after reset (checksums, login checks)

### Implementation Checklist

- [ ] Credential storage (1Password or env vars)
- [ ] Login flow automation (browser, messaging)
- [ ] Browser state snapshots (cookies, storage, profile)
- [ ] Database snapshots (with checksums)
- [ ] File system initialization (fixed timestamps, overlay support)
- [ ] Random seed management
- [ ] Reset controller (Level 1, 2, 3)
- [ ] Verification logic (checksums, login checks)
- [ ] Determinism tests
- [ ] Performance tests (< 1s, < 5s, < 30s)

### Success Criteria

After implementation:

- ✅ Same actions → same rewards (100% reproducible)
- ✅ Level 2 reset < 5 seconds (90%+ of episodes)
- ✅ Level 3 reset < 30 seconds (fallback cases)
- ✅ Session expiration handled gracefully
- ✅ All state components verified after reset

---

## Next Steps

1. **Week 1**: Implement credential management + browser login automation
2. **Week 2**: Implement state snapshots (browser, DB, files)
3. **Week 3**: Implement reset controller + verification
4. **Week 4**: Testing, optimization, documentation

Ready to start with Week 1?
