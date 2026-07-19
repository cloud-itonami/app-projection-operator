# etzhayyim-project-projection-operator

Unified project runtime — projection-manager for org/project governance + projection-operator for project execution. 1 Project = 1 Org model (authn.etzhayyim.com DID-based).

**URL**: `https://po.etzhayyim.com`

## Components

| Component | Nanoid | Endpoint | Purpose |
|-----------|--------|----------|---------|
| `projection-manager-mcp` | `pm7k3x9n` | `https://pm7k3x9n.etzhayyim.com/api/mcp` | Org/Project management, contracts, auctions, member invitations |
| `projection-operator-mcp` | `po1x9k2m` | `https://po1x9k2m.etzhayyim.com/api/mcp` | Project execution, inbox routing, run gates |
| `po-ui` | `po1x9k2m` | `https://po.etzhayyim.com` | Projection operator UI |

## Directory Structure

```
wasm/
├── projection-manager-mcp-component/   # Core governance actor
│   ├── src/app.ts                         # ~1400 lines, 34 MCP tools
│   ├── genkv/                          # Generated KV bindings
│   ├── wit/world.wit
│   └── App manifest
├── projection-operator-mcp-component/  # Execution actor
│   ├── src/app.ts
│   └── ...
├── po-ui-po1x9k2m/                    # SvelteKit UI (TS Native)
│   ├── svelte/                         # SvelteKit app
│   └── static/                         # Embedded SSG output (mage syncstatic)
```

## projection-manager-mcp (pm7k3x9n)

### 統合管理レイヤー連携

統合管理レイヤーとして以下の Actor を連携:

| Actor | Nanoid | MCP Endpoint | 役割 |
|---|---|---|---|
| **projection-manager-mcp** | `pm7k3x9n` | `https://pm7k3x9n.etzhayyim.com/api/mcp` | Org リソース割当、プロジェクトライフサイクル、契約ガバナンス、auction |
| **matrix-mcp** | `br8bojxp` | `https://br8bojxp.etzhayyim.com/api/mcp` | Matrix room 管理 (SWE 対話) |
| **scheduler-mcp** | `5dcfvsbd` | `https://5dcfvsbd.etzhayyim.com/api/mcp` | Agent Orchestrator、タスクスケジューリング |
| **projection-operator-mcp** | `po1x9k2m` | `https://po1x9k2m.etzhayyim.com/api/mcp` | Project 実行基盤 (inbox routing, runs, approval gates) |
| **hub-mcp** | `qk6cjn0l` | `https://qk6cjn0l.etzhayyim.com/api/mcp` | Git 互換 project hub (repos, PRs, commits) |

**MCP tool namespace**: `manager.*` (projection-manager)

| Tool | 説明 |
|---|---|
| `manager.create_org` | Org 作成 (リソースクォータ付き) |
| `manager.add_member` | Org に member 追加 |
| `manager.get_resource_usage` | Org リソース使用状況 |
| `manager.create_project` | Project 作成 (Org プールからリソース割当) |
| `manager.allocate_resources` | Project へのリソース再割当 |
| `manager.create_contract` | 契約作成 (service/social/legal) |
| `manager.create_activity` | Activity 登録 (Actor/Capability 紐付き) |
| `manager.create_auction` | Capability 充足のための auction 開始 |
| `manager.place_bid` | Auction に GCC で入札 |
| `manager.evaluate_project` | Project 進捗評価 |
| `manager.delegate_to_actor` | 統合 Actor への tool 委譲 |

### Data Model (JSON-LD / DoDAF DM2)

| Type | Fields | Notes |
|------|--------|-------|
| `org` | ID, name, members[], quotas, plan, gccBalance | Resource boundary |
| `member` | ID, name, email, role | Org membership |
| `project` | ID, name, orgId, members[], resources, contracts[], activities[], blockers[] | Main work unit |
| `contract` | ID, type (service/social/legal), parties[], jurisdiction | Governance |
| `activity` | ID, type, actorId, capability, responsibility, accountability | DoDAF OV-5b |
| `blocker` | ID, description, severity, status, resolution | Issue tracking |
| `auction` | ID, projectId, capability, bids[], winnerId | Actor selection via GCC |
| `invitation` | ID, orgId, projectId?, email, role, status, invitedBy, token, expiresAt | Member invitations |

### State Persistence

- `managerState` struct holds all data in memory
- Persisted via kotodama WIT bindings (SQL graph)
- KV bucket: `projection-manager`

### Auth Context

- `authContext(r)` extracts `claims{OrgID, UserID}` from HTTP headers
- Reads `X-etzhayyim-ORG-ID`, `X-etzhayyim-USER-ID` headers (set by frontend)
- Falls back to JWT `sub`/`org_id` claims from `Authorization: Bearer` header
- Backward compatible: returns `UserID: "default"` when no auth headers present

### MCP Tools (34 total)

**Org Management (4)**:
`manager.get_org`, `manager.create_org`, `manager.add_member`, `manager.get_resource_usage`

**Project Lifecycle (5)**:
`manager.list_projects`, `manager.create_project`, `manager.get_project`, `manager.update_membership`, `manager.allocate_resources`

**Contract Governance (2)**:
`manager.create_contract`, `manager.list_contracts`

**Activity Tracking (2)**:
`manager.create_activity`, `manager.list_activities`

**Blocker Management (2)**:
`manager.create_blocker`, `manager.resolve_blocker`

**Auction (3)**:
`manager.create_auction`, `manager.place_bid`, `manager.close_auction`

**Evaluate (1)**:
`manager.evaluate_project`

**Actor Integration (2)**:
`manager.delegate_to_actor`, `manager.list_actors`

**Org Member Management (5)**:
`manager.list_members`, `manager.remove_member`, `manager.invite_member`, `manager.list_invitations`, `manager.revoke_invitation`

**Project Member Management (3)**:
`manager.list_project_members`, `manager.invite_project_member`, `manager.remove_project_member`

### Integrated Actors (delegation targets)

| Actor | Nanoid | Project |
|-------|--------|---------|
| matrix-mcp | br8bojxp | etzhayyim-project-matrix |
| scheduler-mcp | 5dcfvsbd | etzhayyim-project-scheduler |
| scheduler-cron-mcp | 2w9k6q1m | etzhayyim-project-scheduler |
| projection-operator-mcp | po1x9k2m | etzhayyim-project-projection-operator |
| hub-mcp | qk6cjn0l | etzhayyim-project-hub |
| credits-mcp | z8l65qxz | etzhayyim-project-credits |

## Build

```bash
cd wasm/projection-manager-mcp-component
etzhayyim build
go vet ./...        # Verify Go compilation locally

cd wasm/projection-operator-mcp-component
etzhayyim build
```

**Note**: Use `go vet ./...` to verify Go code compiles correctly if `etzhayyim build` is unavailable locally.

## CORS

Required headers: `Content-Type,Authorization,X-etzhayyim-ORG-ID,X-etzhayyim-USER-ID`

Allowed origins: `*.etzhayyim.com`, `https://etzhayyim.com`, `http://localhost*`

## Conventions

- `callTool(cl claims, name string, args map[string]any)` — all tool handlers receive auth claims
- Invitation status flow: `pending` → `accepted` | `revoked`
- Invitation tokens expire after 7 days
- Org quota enforcement: project resource allocation checked against org limits
- Auction winner selection: lowest GCC bid wins
