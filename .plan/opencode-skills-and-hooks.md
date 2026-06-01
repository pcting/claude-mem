# Plan: OpenCode Skills & Hooks Integration

## Why

claude-mem has a rich set of 15 skills and a working OpenCode plugin that captures tool executions and chat messages. However, these are not yet surfaced inside OpenCode — the plugin only exposes `claude_mem_search` as a tool, and the skills live exclusively in the Claude Code ecosystem. OpenCode users get no access to claude-mem's capabilities (memory search, knowledge-agent, pathfinder, make-plan, do, etc.) despite having the hooks installed.

## What Changes

- **Register claude-mem skills in OpenCode**: Make all 15 `plugin/skills/*` SKILL.md entries discoverable and loadable by OpenCode
- **Expose skill-triggering tools**: Add OpenCode tool bindings for each skill (e.g. `claude_mem_make_plan`, `claude_mem_do`, `claude_mem_pathfinder`, etc.)
- **Integrate context injection for skills**: Ensure AGENTS.md context includes skill-relevant memory data
- **Add skill lifecycle hooks**: New OpenCode plugin hooks for skill activation, deactivation, and observation filtering
- **Distribute skills during install**: `npx claude-mem install opencode` copies skills into OpenCode's skill directory

## Capabilities

| Capability | Purpose |
|-----------|---------|
| `opencode-skills-registration` | Register and expose all 15 claude-mem skills to OpenCode's discovery system |
| `opencode-skill-tools` | Expose skill-specific tools bridging OpenCode tool calls to worker API |
| `opencode-skill-lifecycle` | Hook-based skill lifecycle — track active skills per session, filter observations |

**Modified**: Existing plugin hooks (`tool.execute.after`, `chat.message`, `event`, `experimental.session.compacting`) continue working; new observations include skill metadata.

---

## Design Decisions

### D1: Tool-based skill invocation over direct SKILL.md loading

**Choice:** Register each skill as an OpenCode tool (`claude_mem_<skill-name>`) rather than loading SKILL.md directly.

**Rationale:** OpenCode's plugin API exposes tools, not skills. SKILL.md files become reference material in AGENTS.md context; actual invocation happens via tool calls. This matches the existing `claude_mem_search` pattern.

**Alternatives considered:**
- Direct SKILL.md loading — may be supported in future OpenCode versions
- MCP-style skill protocol — requires new transport layer; over-engineered

### D2: Skill tools use existing worker endpoints with new ones

**Choice:** Reuse existing HTTP API for known skills; add new POST endpoints for knowledge-agent corpus operations.

**Rationale:** Minimizes worker changes. `mem-search`, `babysit`, `timeline-report` reuse existing endpoints. `knowledge-agent` needs `/api/corpus/build`, `/api/corpus/prime`, `/api/corpus/query`.

**Alternatives considered:**
- Single skill gateway endpoint — new routing layer; adds latency
- WebSocket for skill streaming — too complex for initial implementation

### D3: Skill metadata embedded in observation schema

**Choice:** Include optional `skill_context` field in observations via `tool.execute.after` hook.

**Rationale:** Enables skill-specific filtering and retrospective analysis. Additive, backward compatible, no schema migration.

**Alternatives considered:**
- Separate skill event stream — doubles observation load; unnecessary
- Post-hoc skill detection via tool names — fragile; loses intent signals

### D4: Skills distributed as part of plugin bundle, not separately

**Choice:** Copy all `plugin/skills/*` into `~/.config/opencode/skills/` during install, keeping SKILL.md content unchanged.

**Rationale:** Canonical format preserved. OpenCode discovers from skills directory. No transformation needed.

**Alternatives considered:**
- Convert SKILL.md to OpenCode-native format — maintenance burden (two formats)
- Inline skills into AGENTS.md — too large; loses discoverability

---

## Requirements & Scenarios

### Requirement: Each skill has a corresponding OpenCode tool

Every registered claude-mem skill SHALL have a tool named `claude_mem_<skill-name>`. Each tool SHALL:
- Accept arguments appropriate to the skill's purpose
- Forward the call to the claude-mem worker via HTTP POST
- Return the worker's response formatted for the AI

| Scenario | Behavior |
|----------|----------|
| `claude_mem_search` forwards to worker | WHEN `query="authentication"`, THEN POST `/api/search?query=authentication&limit=10`, return parsed text |
| `claude_mem_make_plan` forwards to worker | WHEN `plan="refactor auth module"`, THEN POST `/api/skills/make-plan`, return generated plan |
| Worker down | Return `"claude-mem worker is not running. Start it with: npx claude-mem start"` |
| `claude_mem_do` orchestrates subagents | WHEN `plan-id="abc123"`, THEN POST `/api/skills/do`, return execution steps |

### Requirement: Knowledge agent tools have dedicated endpoints

| Tool | Endpoint | Purpose |
|------|----------|---------|
| `claude_mem_build_corpus` | `POST /api/corpus/build` | Create filtered corpus from observations |
| `claude_mem_query_corpus` | `POST /api/corpus/query` | Ask a question against a corpus |
| `claude_mem_list_corpora` | `GET /api/corpus/list` | List all corpora with stats |

### Requirement: Tool arguments follow consistent schema

All skill tools SHALL use Zod-specified argument schemas validated before forwarding.

| Scenario | Behavior |
|----------|----------|
| Invalid args rejected | `claude_mem_search` without query → error explaining missing parameter |
| Optional params default correctly | `claude_mem_search` with only `query="test"` → `limit=10` used |

### Requirement: Tool execution is captured as observations

Every skill tool execution SHALL be captured as an observation via `tool.execute.after` hook.

| Scenario | Behavior |
|----------|----------|
| Skill tool execution produces observation | `claude_mem_pathfinder` call → observation with `tool_name="claude_mem_pathfinder"` |

### Requirement: All 15 claude-mem skills are discoverable

Skills distributed to `~/.config/opencode/skills/<skill-name>/` during install, each containing its canonical `SKILL.md` from `plugin/skills/<skill-name>/`.

**The 15 skills:** `babysit`, `design-is`, `do`, `how-it-works`, `knowledge-agent`, `learn-codebase`, `make-plan`, `mem-search`, `oh-my-issues`, `pathfinder`, `smart-explore`, `timeline-report`, `version-bump`, `weekly-digests`, `wowerpoint`

| Scenario | Behavior |
|----------|----------|
| Skill files copied during install | `npx claude-mem install opencode` → all 15 directories copied with SKILL.md intact |
| Skills discoverable at runtime | OpenCode loads plugin → all 15 skill definitions registered and visible |
| Skills survive plugin update | Re-running install → skill files overwritten with latest version |

### Requirement: Skill-to-tool mapping is configurable

| Scenario | Behavior |
|----------|----------|
| Disable via config | `"skills": ["mem-search", "make-plan"]` → only 2 tools registered, 13 absent |
| Default is all enabled | No filter in config → all 15 tools registered |

### Requirement: Skill activation is tracked per session

| Scenario | Behavior |
|----------|----------|
| Activation recorded | `claude_mem_make_plan` called → recorded as active for session |
| Multiple skills active | `make-plan` then `do` → both recorded as active |
| Persists across tool calls | `make-plan` + 3 subsequent tool calls → `make-plan` context included in all observations |

### Requirement: Observations include skill context metadata

| Scenario | Behavior |
|----------|----------|
| Active skill tags | `claude_mem_pathfinder` + Read tool → observation includes `skill_context: ["pathfinder"]` |
| Multiple active skills | `make-plan` + `do` + Bash tool → `skill_context: ["make-plan", "do"]` |

### Requirement: Skill lifecycle hooks

| Hook | Payload | Trigger |
|------|---------|---------|
| `skill.active` | `{ skillName, sessionId, timestamp }` | Skill tool invoked |
| `skill.inactive` | `{ skillName, sessionId }` | Skill effect expires (10 tool calls or session end) |

| Scenario | Behavior |
|----------|----------|
| `skill.active` fires | `claude_mem_smart_explore` → `skill.active` event with `{ skillName: "smart-explore", sessionId, timestamp }` |
| `skill.inactive` fires | 10 tool calls since `claude_mem_pathfinder` → `skill.inactive` event |

### Requirement: Skill filtering on observation queries

| Scenario | Behavior |
|----------|----------|
| Filter by single skill | `skill="make-plan"` → only observations with matching `skill_context` |
| Filter by multiple skills | `skill=["make-plan", "do"]` → observations with either context |

### Requirement: Skill context injection in AGENTS.md

| Scenario | Behavior |
|----------|----------|
| Active skills listed | `syncContextToAgentsMd` → injects "Active Skills" section |
| Skill descriptions included | Each listed skill includes SKILL.md frontmatter description |

---

## Worker API Endpoints

### Existing (reused)

| Endpoint | Method | Used By |
|----------|--------|---------|
| `/api/search?query=&limit=` | POST | `claude_mem_search` |
| `/api/skills/make-plan` | POST | `claude_mem_make_plan` |
| `/api/skills/do` | POST | `claude_mem_do` |

### New

| Endpoint | Method | Used By |
|----------|--------|---------|
| `/api/corpus/build` | POST | `claude_mem_build_corpus` |
| `/api/corpus/prime` | POST | knowledge-agent priming |
| `/api/corpus/query` | POST | `claude_mem_query_corpus` |
| `/api/corpus/list` | GET | `claude_mem_list_corpora` |

---

## Migration Plan

| Phase | Scope |
|-------|-------|
| **1** | Add skill tools to OpenCode plugin + distribute skills on install. Existing `claude_mem_search` continues working. |
| **2** | Add skill lifecycle hooks (`skill.active`, `skill.inactive`) and `skill_context` to observations. |
| **3** | Add corpus endpoints for knowledge-agent skill. |

**Rollback:** Removing skill tool definitions is a single-file revert of `src/integrations/opencode-plugin/index.ts`. Skills removed by uninstalling the plugin.

---

## Risks & Trade-offs

| Risk | Impact | Mitigation |
|------|--------|------------|
| Too many tools clutter OpenCode's tool palette | High — AI context window waste | Only expose when contextually relevant; use tool descriptions for conditional discovery |
| New corpus endpoints add worker complexity | Medium | Phase into implementation; start with search + key skills, expand iteratively |
| Skill metadata increases observation payload | Low | `skill_context` is optional, only sent when skill tool actively invoked |
| OpenCode plugin API changes break skill hooks | Medium | Contract test (exists for hooks); add new test for tool registration |
| Duplicate skills between Claude Code and OpenCode | Low | Skills are canonical in `plugin/skills/`; both platforms read same source |

---

## Open Questions

1. Does OpenCode support dynamic tool registration at runtime, or must all tools be registered at plugin load? (Impacts lazy-loading 15 tools vs registering upfront.)
2. Should skills be project-scoped (only active for specific projects) or global?
3. Should skill tools accept the full SKILL.md as a parameter, or should they reference skills by name?

---

## Impact

- **Affected code:** `src/integrations/opencode-plugin/index.ts`, `src/services/integrations/OpenCodeInstaller.ts`, `plugin/skills/`
- **New APIs:** POST endpoints on worker for corpus operations
- **Dependencies:** No new external dependencies; builds on existing worker HTTP API
- **Systems:** OpenCode plugin, npx installer, AGENTS.md context injection
