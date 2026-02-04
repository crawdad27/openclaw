# Adding a Tool to OpenClaw: Apple Notes Example

## Decision Tree: Which Approach?

Before implementing, ask:

```
┌─────────────────────────────────────────────────────────────────┐
│ What kind of tool is Apple Notes?                               │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Is it a native platform API (requires Swift/Kotlin)?            │
│ (camera, location, notifications, native UI)                    │
└─────────────────────────────────────────────────────────────────┘
                            │
                     NO ────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ Is there an existing CLI tool we can wrap?                      │
│ (memo, things, remindctl, osascript)                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                     YES ───┘  ← Apple Notes case!
                            │
                            ▼
                    ┌───────────────┐
                    │ CREATE SKILL  │
                    └───────────────┘
```

**For Apple Notes:** We have a CLI tool called `memo` that wraps the AppleScript API.
**Decision:** Use the **Skills system**.

---

## Workflow: Adding Apple Notes as a Skill

### Step 1: Create the Skill Directory

```bash
mkdir -p skills/apple-notes
```

### Step 2: Create the SKILL.md File

**File:** `skills/apple-notes/SKILL.md`

````markdown
---
name: apple-notes
description: Manage Apple Notes via the `memo` CLI on macOS
homepage: https://github.com/antoniorodr/memo
metadata:
  {
    "openclaw":
      {
        "emoji": "📝",
        "os": ["darwin"],
        "requires": { "bins": ["memo"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "antoniorodr/memo/memo",
              "bins": ["memo"],
              "label": "Install memo via Homebrew",
            },
          ],
      },
  }
---

# Apple Notes CLI

Use the `memo` command to manage Apple Notes on macOS.

## Commands

### List all notes

```bash
memo notes
```
````

### Create a new note

```bash
memo add "Title" --body "Note content here"
```

### Search notes

```bash
memo search "query"
```

### Read a specific note

```bash
memo show <note-id>
```

### Update a note

```bash
memo update <note-id> --body "Updated content"
```

### Delete a note

```bash
memo delete <note-id>
```

## Examples

Create a meeting note:

```bash
memo add "Team Standup 2024-01-15" --body "Discussed roadmap priorities"
```

Search for notes about a project:

```bash
memo search "project alpha"
```

## Notes

- Notes are stored in the default Notes.app folder
- Requires macOS with Notes.app installed
- The `memo` tool uses AppleScript under the hood

````

### Step 3: Metadata Explanation

The frontmatter YAML contains critical metadata:

```yaml
metadata: {
  "openclaw": {
    # Emoji for UI display
    "emoji": "📝",

    # Platform restriction - only enable on macOS
    "os": ["darwin"],

    # Required binaries - skill only loads if `memo` exists
    "requires": {
      "bins": ["memo"]
    },

    # Installation instructions shown in CLI/UI
    "install": [{
      "id": "brew",
      "kind": "brew",  # brew | npm | pip | apt | manual
      "formula": "antoniorodr/memo/memo",
      "bins": ["memo"],
      "label": "Install memo via Homebrew"
    }]
  }
}
````

**Metadata fields:**

- `os`: Array of platforms (`["darwin"]`, `["linux"]`, `["win32"]`, or omit for all)
- `requires.bins`: Required binaries (e.g., `["memo"]`)
- `requires.anyBins`: Optional - at least one must exist (e.g., `["docker", "podman"]`)
- `requires.env`: Required environment variables (e.g., `["GITHUB_TOKEN"]`)
- `install`: Installation specs for missing dependencies

### Step 4: How Skills Are Loaded

**Loading pipeline:** `src/agents/skills/load.ts`

```typescript
// 1. Scan skills/ directory
const skillDirs = await fs.readdir("skills/");

// 2. For each skill, read SKILL.md
for (const dir of skillDirs) {
  const content = await fs.readFile(`skills/${dir}/SKILL.md`, "utf-8");

  // 3. Parse frontmatter
  const { data, content: body } = matter(content);
  const metadata = resolveOpenClawMetadata(data);

  // 4. Check platform compatibility
  if (metadata.os && !metadata.os.includes(process.platform)) {
    continue; // Skip - wrong platform
  }

  // 5. Check binary requirements
  for (const bin of metadata.requires.bins) {
    if (!hasBinary(bin)) {
      // Skill not available, but show installation instructions
      missingSkills.push({ name, metadata, installSpecs: metadata.install });
      continue;
    }
  }

  // 6. Skill is available - include in system prompt
  availableSkills.push({ name, body, metadata });
}
```

**Binary check:** `src/agents/skills/has-binary.ts`

```typescript
export function hasBinary(binName: string): boolean {
  try {
    // Check if binary exists in PATH
    const result = execSync(`command -v ${binName}`, {
      encoding: "utf-8",
      stdio: ["pipe", "pipe", "ignore"],
    });
    return result.trim().length > 0;
  } catch {
    return false;
  }
}
```

### Step 5: How Skills Appear to the Agent

Skills are **NOT** separate tools in the TypeBox schema. Instead, they are **injected into the system prompt** as Bash usage documentation.

**System prompt injection:** `src/agents/skills/render.ts`

```typescript
export function renderSkillsForPrompt(skills: LoadedSkill[]): string {
  let prompt = "\n\n# Available Skills\n\n";

  for (const skill of skills) {
    prompt += `## ${skill.metadata.emoji} ${skill.name}\n\n`;
    prompt += skill.body; // The markdown content
    prompt += "\n\n---\n\n";
  }

  return prompt;
}
```

**Agent sees:**

````
# Available Skills

## 📝 apple-notes

Use the `memo` command to manage Apple Notes on macOS.

### Commands

#### List all notes
```bash
memo notes
````

...

````

**Agent uses the Bash tool** to execute skill commands:

```typescript
// Agent decides to create a note
{
  "tool": "bash",
  "command": "memo add 'Meeting Notes' --body 'Discussed Q1 roadmap'"
}
````

### Step 6: Installation Flow (User Experience)

When a user tries to use a skill but the binary is missing:

1. **Agent detects skill is unavailable** (not in system prompt)
2. **User runs:** `openclaw skills list`

```bash
$ openclaw skills list

Available Skills (3):
  ✓ apple-notes        Manage Apple Notes via memo CLI
  ✓ peekaboo          macOS UI automation
  ✓ things-mac        Things 3 task management

Missing Dependencies (1):
  ✗ apple-reminders   Missing binary: remindctl
    Install: brew install remindctl
```

3. **User installs:**

```bash
$ brew install antoniorodr/memo/memo
```

4. **Next agent run:** Skill appears in system prompt automatically

### Step 7: Testing the Skill

```bash
# 1. Ensure memo is installed
brew install antoniorodr/memo/memo

# 2. Verify binary exists
command -v memo

# 3. Test skill manually
memo notes

# 4. Start OpenClaw agent and test
openclaw agent

> "Create a note in Apple Notes with title 'Test' and body 'Hello World'"

# Agent should use: bash tool with `memo add "Test" --body "Hello World"`
```

---

## Alternative Approaches (Not Used for Apple Notes)

### Approach 2: Node Delegation (Native Swift Implementation)

If we wanted **native Notes.app API access** instead of CLI:

**Step 1:** Add Swift code to macOS app

```swift
// apps/macos/Sources/OpenClaw/NodeMode/MacNodeNotesService.swift

import EventKit

final class MacNodeNotesService {
    func createNote(title: String, body: String) async throws -> String {
        // Use EventKit to create note in Notes.app
        let store = EKEventStore()
        try await store.requestAccess(to: .reminder)

        let note = EKReminder(eventStore: store)
        note.title = title
        note.notes = body

        try store.save(note, commit: true)
        return note.calendarItemIdentifier
    }

    func listNotes() async throws -> [Note] {
        // Fetch notes from EventKit
    }
}
```

**Step 2:** Register node command in gateway session

```swift
// apps/macos/Sources/OpenClaw/NodeMode/MacNodeSession.swift

func handleNodeInvoke(request: NodeInvokeRequest) async -> NodeInvokeResponse {
    switch request.command {
    case "notes.create":
        let params = request.params as! NotesCreateParams
        let noteId = try await notesService.createNote(
            title: params.title,
            body: params.body
        )
        return .success(payload: ["noteId": noteId])

    case "notes.list":
        let notes = try await notesService.listNotes()
        return .success(payload: notes)

    default:
        return .error(message: "Unknown command")
    }
}
```

**Step 3:** Add to nodes tool

```typescript
// src/agents/tools/nodes-tool.ts

case "notes_create": {
  const nodeId = await resolveNodeId(gatewayOpts, node);
  const result = await callGatewayTool("node.invoke", gatewayOpts, {
    nodeId,
    command: "notes.create",
    params: { title, body },
  });
  return jsonResult(result.payload);
}
```

**When to use:** Camera, location, notifications, screen recording, native UI automation

### Approach 3: Plugin Tool (TypeScript Implementation)

If Apple Notes had a JavaScript API or we wanted to wrap AppleScript directly:

**Step 1:** Create plugin directory

```bash
mkdir -p extensions/apple-notes/src
```

**Step 2:** Create tool implementation

```typescript
// extensions/apple-notes/src/notes-tool.ts

import { Type } from "@sinclair/typebox";
import type { AgentTool } from "@mariozechner/pi-agent-core";
import { execSync } from "node:child_process";

export function createAppleNotesTool(): AgentTool<unknown, unknown> {
  return {
    name: "apple_notes",
    description: "Manage Apple Notes on macOS",
    parameters: Type.Object({
      action: Type.Unsafe<"create" | "list" | "search">({
        type: "string",
        enum: ["create", "list", "search"],
      }),
      title: Type.Optional(Type.String()),
      body: Type.Optional(Type.String()),
      query: Type.Optional(Type.String()),
    }),
    execute: async (_toolCallId, args) => {
      const { action, title, body, query } = args;

      switch (action) {
        case "create": {
          const script = `
            tell application "Notes"
              make new note at folder "Notes" with properties {name:"${title}", body:"${body}"}
            end tell
          `;
          execSync(`osascript -e '${script}'`);
          return { kind: "success", result: "Note created" };
        }

        case "list": {
          const script = `
            tell application "Notes"
              get name of every note
            end tell
          `;
          const output = execSync(`osascript -e '${script}'`, { encoding: "utf-8" });
          return { kind: "success", result: output };
        }

        case "search": {
          // Implement search logic
        }
      }
    },
  };
}
```

**Step 3:** Register in plugin

```typescript
// extensions/apple-notes/index.ts

import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { createAppleNotesTool } from "./src/notes-tool.js";

export default function register(api: OpenClawPluginApi) {
  api.registerTool(
    (ctx) => {
      // Only load on macOS
      if (process.platform !== "darwin") return null;
      if (ctx.sandboxed) return null;

      return createAppleNotesTool();
    },
    { optional: true }, // Require explicit enablement
  );
}
```

**Step 4:** User enables plugin

```json
// ~/.openclaw/openclaw.json
{
  "tools": {
    "allow": ["apple_notes"]
  }
}
```

**When to use:** Optional/experimental tools, third-party integrations, complex logic

### Approach 4: Core Tool (Built-in TypeScript)

If Apple Notes was universally needed:

**Step 1:** Create core tool

```typescript
// src/agents/tools/apple-notes-tool.ts

export function createAppleNotesTool(): AgentTool<unknown, unknown> {
  // Same implementation as plugin
}
```

**Step 2:** Register in core tools

```typescript
// src/agents/openclaw-tools.ts

export function createOpenClawTools(options?: { ... }): AnyAgentTool[] {
  return [
    createBrowserTool({ ... }),
    createNodesTool({ ... }),
    createAppleNotesTool(), // ← Add here
    // ...
  ];
}
```

**When to use:** Universally needed, no platform restrictions, core functionality

---

## Summary: Chosen Approach for Apple Notes

**Selected:** Skills System (External CLI wrapper)

**Rationale:**

1. ✅ `memo` CLI already exists and is well-maintained
2. ✅ No need to maintain AppleScript glue code
3. ✅ Platform check handled automatically (`os: ["darwin"]`)
4. ✅ Installation flow is simple (`brew install`)
5. ✅ Agent can use Bash tool naturally
6. ✅ No additional TypeScript/Swift code to maintain

**Trade-offs:**

- ❌ Requires external binary installation
- ❌ Less type-safe than TypeScript tool (just Bash strings)
- ❌ Harder to provide structured error messages
- ✅ But simpler, more maintainable, and flexible

---

## Complete File Locations

| File                               | Purpose                        |
| ---------------------------------- | ------------------------------ |
| `skills/apple-notes/SKILL.md`      | Skill definition and docs      |
| `src/agents/skills/load.ts`        | Skill loading logic            |
| `src/agents/skills/frontmatter.ts` | Metadata parsing               |
| `src/agents/skills/has-binary.ts`  | Binary availability check      |
| `src/agents/skills/render.ts`      | System prompt injection        |
| `src/cli/skills-cli/index.ts`      | `openclaw skills list` command |

---

## Decision Matrix: Which Approach?

| Criteria                   | Skill            | Node Delegation | Plugin Tool      | Core Tool       |
| -------------------------- | ---------------- | --------------- | ---------------- | --------------- |
| External binary available  | ✅ Best          | ❌              | ⚠️ Can wrap      | ⚠️ Can wrap     |
| Native platform API needed | ❌               | ✅ Best         | ❌               | ❌              |
| Optional/experimental      | ✅ Good          | N/A             | ✅ Best          | ❌              |
| Universal requirement      | ⚠️ Needs install | N/A             | ⚠️ Gated         | ✅ Best         |
| Platform-specific          | ✅ Auto-filtered | ✅ Native       | ✅ Factory check | ⚠️ Manual check |
| Maintenance burden         | ✅ Low           | ⚠️ Medium       | ⚠️ Medium        | ⚠️ High         |
| Type safety                | ❌ Bash strings  | ✅ TypeBox      | ✅ TypeBox       | ✅ TypeBox      |
| Installation complexity    | ⚠️ User installs | ✅ Built-in     | ✅ Built-in      | ✅ Built-in     |
