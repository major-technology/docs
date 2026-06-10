# Plan: Mounting Application MCPs on Agents with Tool Permissions

**Status:** Draft for review
**Scope note:** This plan was drafted against the public docs (gateway architecture, RBAC, environments, MCP support). File-level implementation details need to be mapped onto the gateway and agents codebases during grooming.

## Goal

Let builders attach ("mount") an application's MCP server — already exposed and supported by the Major API Gateway — onto an agent, and configure per-tool permissions for each mount. A mounted agent can then call the app's MCP tools at runtime, subject to both the mount's tool policy and the invoking user's RBAC.

## Current state (what we build on)

- **Application MCPs exist.** Apps expose MCP servers through the gateway; the gateway already does unified permission checking across app-level and org-level MCP access modes.
- **Agents exist** (web/Slack assistant sessions, AI coder, memory agent) but have no first-class way to consume an app's MCP tools.
- **RBAC exists** at org, app (Admin/Edit/View), and resource (Admin/Builder) levels, with groups and the All Users / All Builders managed groups.
- **Environments exist**, including a "default build environment" used by the AI coder, with production always pinned to the default environment.

## Concepts to introduce

### 1. Agent MCP mount

A configuration record linking an agent to one application MCP:

- `agent_id` — the agent the MCP is mounted on
- `app_id` — the application whose MCP is mounted
- `environment_id` — which environment the tools execute against (default: the org's default build environment for builder-facing agents; production for end-user-facing agents)
- `enabled` — kill switch for the whole mount
- `new_tool_policy` — what happens when an app deploy introduces tools that have no explicit policy yet (`disabled` recommended default, `enabled` opt-in)
- audit fields (`created_by`, timestamps)

One agent can have many mounts; one app MCP can be mounted on many agents.

### 2. Per-tool policy

For each mount, a policy per tool:

- `tool_name` (exact name as listed by the app MCP)
- `policy`: `enabled` | `disabled` (v1); `requires_approval` reserved for a later phase
- Tools not present in the policy table fall back to the mount's `new_tool_policy`

## Permission model (three layers)

1. **Config-time (who can mount):** Mounting app X on agent Y requires Admin on the app *and* permission to manage the agent. This mirrors the existing "deployer's permissions are checked when invoked" model — the mount creator is accountable for exposure.
2. **Mount-time tool policy:** The per-tool allow-list above, enforced by the gateway.
3. **Runtime user RBAC:** Agents act on behalf of a user. When the agent calls a mounted tool, the gateway checks the invoking user's JWT scopes against the app (View or higher on the app), exactly as it would if the user hit the app MCP directly. A mount never grants a user access they don't already have — it only makes tools reachable from the agent surface.

Per-user OAuth connectors keep their existing behavior: if a tool's underlying resource needs per-user OAuth and the invoking user hasn't connected, the call fails with the standard "complete OAuth at /connect" error.

## Gateway changes

1. **Session assembly:** When an agent session starts, resolve the agent's enabled mounts, fetch each app MCP's tool list, filter by tool policy, and merge into the agent's toolset.
2. **Namespacing:** Prefix tool names with the app slug (e.g. `inventory-app__lookup_sku`) to avoid collisions across mounts and with the agent's built-in tools. Keep a stable mapping so policies survive renames of the prefix scheme.
3. **Enforcement at call time, not just list time:** `tools/call` must re-check (a) mount enabled, (b) tool policy, (c) invoking user RBAC on the app, (d) environment resolution. Filtering only at `tools/list` is not a security boundary.
4. **Tool list caching + drift handling:** Cache the app MCP tool list per mount; invalidate on app deploy. New tools follow `new_tool_policy`; removed tools are pruned from the live toolset but their policy rows are retained (apps may restore tools in a later deploy).
5. **Environment resolution:** Tool execution uses the mount's pinned environment, falling back per the existing environment fallback rules. The org-MCP `set_environment` selection does not override a mount's pin (pin wins; document this).

## API surface

CRUD on mounts and policies, scoped under the agent:

- `GET /agents/:id/mcp-mounts` — list mounts with status
- `POST /agents/:id/mcp-mounts` — create (app, environment, new_tool_policy)
- `PATCH /agents/:id/mcp-mounts/:mountId` — enable/disable, change environment/new_tool_policy
- `DELETE /agents/:id/mcp-mounts/:mountId`
- `GET /agents/:id/mcp-mounts/:mountId/tools` — introspected tool list (name, description, current policy, "new since last deploy" flag)
- `PUT /agents/:id/mcp-mounts/:mountId/tools/:toolName` — set per-tool policy (bulk variant for the UI)

## UI

Agent settings gains a **Tools / MCP servers** tab:

- "Mount application MCP" picker (apps the user Admins that expose an MCP)
- Per-mount card: environment selector, enable toggle, new-tool default
- Tool table: name, description, policy toggle, badge for tools added since the mount was configured
- Empty/error states for apps whose MCP is unreachable or not yet deployed

## CLI

Follow the existing `major <noun>` pattern:

- `major agent mcp list|mount|unmount` — manage mounts
- `major agent mcp tools` — interactive per-tool enable/disable (mirrors `major resource manage` UX)

CLI can land in a later phase; the API + web UI are the critical path.

## Observability & audit

- Audit log entries for mount create/update/delete and tool-policy changes (actor, agent, app, diff)
- Per-tool invocation metrics feeding the existing agent activity dashboards (Integrations page already charts weekly agent activity)
- Request IDs propagated through the gateway as with resource invocations

## Security considerations

- **Call-time enforcement** (above) is the primary boundary; list-time filtering is UX.
- **Prompt injection:** tool *outputs* from app MCPs are untrusted content inside the agent context. At minimum, document this; consider output size limits and content tagging in the agent runtime.
- **Default-off for new tools** prevents an app deploy from silently widening an agent's capabilities.
- **No privilege escalation:** runtime checks always use the invoking user's identity; the mount creator's permissions are only checked at config time.

## Phasing

| Phase | Deliverable |
| --- | --- |
| 1 | Data model, gateway mount resolution + call-time enforcement, mounts/policies API, minimal web UI (mount + tool toggles) |
| 2 | Drift handling polish (new-tool badges, deploy-triggered cache invalidation), audit log, metrics, CLI commands |
| 3 | `requires_approval` tool policy with in-conversation approval UX; per-group tool policies if demanded |

## Docs updates (this repo, once shipped)

- New page: **Agents > Mounting application MCPs** (concept, permissions, environment behavior)
- `web/permissions.mdx`: add the config-time/runtime permission layers
- `cli` section: `major agent mcp` reference (Phase 2)
- `getting-started/core-concepts.mdx`: mention agents + tool mounts under the architecture section

## Open questions

1. Is there a first-class "agent" entity with its own settings surface today, or do mounts hang off the org-level assistant config? (Determines where the data model and UI attach.)
2. Should end-user-facing agents be allowed to mount non-production environments at all, or is environment pinning restricted to builder agents?
3. Tool-name matching: exact names only in v1, or support glob patterns (e.g. `read_*`) in policies?
4. Does the gateway already cache app MCP tool lists per deploy, or is introspection on-demand?
