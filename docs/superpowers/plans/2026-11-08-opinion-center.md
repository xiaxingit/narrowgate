# Opinion Center Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Narrow Gate's local Opinion Center for managing OpenWatch sources, accounts, raw content, notification routes, and operating health without adding AI analysis.

**Architecture:** OpenWatch remains the source of truth for collection, deduplication, and sending. A localhost-only OpenWatch Admin Bridge exposes a normalized, secret-free control-plane API. Narrow Gate's API consumes that bridge and serves a web dashboard; its own store contains only UI preferences and management audit records.

**Tech Stack:** Python 3.14, FastAPI, SQLite, React, TypeScript, Vite, OpenWatch SQLite and local control services.

**Spec:** `docs/superpowers/specs/2026-11-08-opinion-center-design.md`

## Global Constraints

- Preserve OpenWatch as the sole collector and notification sender.
- Keep source-specific content fields and notification semantics isolated.
- Bind all management APIs to `127.0.0.1`; never return or log keys, Cookie, token, browser session, or pairing code.
- Global enabled channels are additive with account-specific channels; Bark remains the only registered delivery channel in the initial implementation.
- No AI-generated viewpoint, topic aggregation, individual-stock attribution, review, plan, holding, or strategy feature is in scope.

## Review Focus

- An old content row with only `sent` must display unknown historical channel detail rather than a fabricated Bark delivery.
- Disabling then enabling an account must preserve the account and all previously captured content.
- A source-specific field missing from one source must not break the unified timeline or borrow a field from another source.
- An unavailable OpenWatch bridge must produce a clear health state and must not permit a destructive fallback write to SQLite.
- A per-account extra channel equal to the global Bark channel must result in one effective Bark route, not duplicate delivery.

---

### Task 1: Bootstrap the local Narrow Gate application

**Files:**
- Create: `apps/api/pyproject.toml`
- Create: `apps/api/narrowgate/app.py`
- Create: `apps/api/narrowgate/settings.py`
- Create: `apps/web/package.json`
- Create: `apps/web/src/main.tsx`
- Create: `apps/web/src/App.tsx`
- Test: `apps/api/tests/test_health.py`

**Interfaces:**
- Produces `GET /health` returning `{"status":"ok"}`.
- Produces a local web shell with navigation entries for 观点、计划、持仓、战法、资讯; only 观点 is enabled.

- [ ] **Step 1: Write the failing health endpoint test**

```python
def test_health_returns_ok(client):
    assert client.get("/health").json() == {"status": "ok"}
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run: `pytest apps/api/tests/test_health.py -q`

Expected: FAIL because the Narrow Gate API does not exist.

- [ ] **Step 3: Implement the settings loader and FastAPI application**

Implement `create_app() -> FastAPI` in `apps/api/narrowgate/app.py`. The app reads its local configuration from `settings.py` and binds only to localhost in the start command.

- [ ] **Step 4: Implement the static web shell**

The shell labels non-viewpoint modules as coming soon and routes the active entry to the Opinion Center.

- [ ] **Step 5: Run focused API and web checks**

Run: `pytest apps/api/tests/test_health.py -q` and `npm run build --prefix apps/web`

Expected: both pass.

### Task 2: Add a secret-free OpenWatch Admin Bridge

**Files:**
- Create: `/Users/shawn/.claude/workspace/openwatch/openwatch_admin_bridge.py`
- Modify: `/Users/shawn/.claude/workspace/openwatch/openwatch.py`
- Create: `/Users/shawn/.claude/workspace/openwatch/tests/test_openwatch_admin_bridge.py`

**Interfaces:**
- Produces `GET /v1/sources`, `GET /v1/accounts`, `GET /v1/content`, and `GET /v1/health` on localhost.
- Produces account `PATCH /v1/accounts/{source}/{account_id}` for enabled and interval changes only.
- Produces no secret-bearing fields in any response.

- [ ] **Step 1: Write failing bridge tests**

```python
def test_accounts_hide_sensitive_configuration(client):
    payload = client.get("/v1/accounts").json()
    assert "cookie" not in json.dumps(payload).lower()
    assert "token" not in json.dumps(payload).lower()
```

- [ ] **Step 2: Run bridge tests to verify they fail**

Run: `python3 -m unittest tests/test_openwatch_admin_bridge.py -v`

Expected: FAIL because the bridge module and routes do not exist.

- [ ] **Step 3: Implement read adapters for all current sources**

Implement source adapters that normalize the existing account and content tables into bridge DTOs while retaining `source_type`, `content_type`, raw body, timestamps, images, original URL, and delivery state.

- [ ] **Step 4: Implement controlled account and source mutations**

Route enable/disable and interval changes through existing OpenWatch control methods. Reject unsupported fields and never delete accounts or content.

- [ ] **Step 5: Run bridge tests and the existing OpenWatch suite**

Run: `python3 -m unittest tests/test_openwatch_admin_bridge.py -v` and `python3 -m unittest discover -s tests -q`

Expected: bridge tests pass; all pre-existing OpenWatch tests remain green.

### Task 3: Build the Opinion Center read model and API

**Files:**
- Create: `apps/api/narrowgate/openwatch_client.py`
- Create: `apps/api/narrowgate/opinions.py`
- Create: `apps/api/tests/test_opinions.py`
- Modify: `apps/api/narrowgate/app.py`

**Interfaces:**
- Consumes the bridge DTOs from Task 2.
- Produces `GET /api/opinions/accounts`, `GET /api/opinions/content`, and `GET /api/opinions/health`.
- Produces account state changes through `PATCH /api/opinions/accounts/{source}/{account_id}`.

- [ ] **Step 1: Write failing API tests for timeline isolation and health failures**

```python
def test_timeline_keeps_source_specific_content_fields(client):
    item = client.get("/api/opinions/content").json()["items"][0]
    assert item["source_type"] == "weibo"
    assert item["original_url"].startswith("https://")
```

- [ ] **Step 2: Run focused tests to verify they fail**

Run: `pytest apps/api/tests/test_opinions.py -q`

Expected: FAIL because the OpenWatch client and Opinion Center API do not exist.

- [ ] **Step 3: Implement the OpenWatch client and read model**

Implement typed DTO validation, pagination, source/account/time filters, and explicit `historical_channel_detail: "unknown"` for legacy delivery rows without channel-level data.

- [ ] **Step 4: Implement account controls with optimistic-refresh-safe errors**

Forward only enabled and interval fields to the bridge and return the bridge's canonical account representation.

- [ ] **Step 5: Run focused tests**

Run: `pytest apps/api/tests/test_opinions.py -q`

Expected: PASS.

### Task 4: Implement the Opinion Center dashboard

**Files:**
- Create: `apps/web/src/features/opinions/OpinionCenter.tsx`
- Create: `apps/web/src/features/opinions/AccountsTable.tsx`
- Create: `apps/web/src/features/opinions/ContentTimeline.tsx`
- Create: `apps/web/src/features/opinions/HealthPanel.tsx`
- Create: `apps/web/src/features/opinions/api.ts`
- Create: `apps/web/src/features/opinions/OpinionCenter.test.tsx`
- Modify: `apps/web/src/App.tsx`

**Interfaces:**
- Consumes Task 3 APIs.
- Produces dashboard views for health, account management, raw-content timeline, and delivery status.

- [ ] **Step 1: Write failing component tests**

```tsx
it("shows unknown legacy delivery detail instead of Bark success", async () => {
  render(<OpinionCenter />)
  expect(await screen.findByText("历史渠道明细未知")).toBeVisible()
})
```

- [ ] **Step 2: Run the focused component test to verify it fails**

Run: `npm test --prefix apps/web -- OpinionCenter.test.tsx`

Expected: FAIL because the dashboard has not been implemented.

- [ ] **Step 3: Implement account, timeline, and health views**

Show source-specific content without automatic semantic classification. Keep original links as explicit user actions; they must not turn a dashboard row into an unintended navigation.

- [ ] **Step 4: Implement accessible controls and error states**

Disable unsupported channel controls, show bridge unavailability clearly, and refresh the affected account after a successful mutation.

- [ ] **Step 5: Run web tests and production build**

Run: `npm test --prefix apps/web -- OpinionCenter.test.tsx` and `npm run build --prefix apps/web`

Expected: both pass.

### Task 5: Add channel route configuration without a second sender

**Files:**
- Modify: `/Users/shawn/.claude/workspace/openwatch/database.py`
- Modify: `/Users/shawn/.claude/workspace/openwatch/notification.py`
- Modify: `/Users/shawn/.claude/workspace/openwatch/openwatch_admin_bridge.py`
- Create: `/Users/shawn/.claude/workspace/openwatch/tests/test_notification_routes.py`
- Modify: `apps/api/narrowgate/opinions.py`
- Modify: `apps/web/src/features/opinions/AccountsTable.tsx`

**Interfaces:**
- Produces `effective_channels = global_channels ∪ account_extra_channels` with stable deduplication.
- Produces read and update routes for account extra channels.
- Supports Bark only until another sender is registered.

- [ ] **Step 1: Write failing route-resolution tests**

```python
def test_account_bark_extra_does_not_duplicate_global_bark():
    assert effective_channels(["bark"], ["bark"]) == ["bark"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m unittest tests/test_notification_routes.py -v`

Expected: FAIL because account-level channel routes do not exist.

- [ ] **Step 3: Implement the route store and effective-channel resolver**

Persist only channel names and account mapping. Keep keys in existing channel configuration and reject a requested extra channel that has no registered sender.

- [ ] **Step 4: Use the resolver at notification dispatch**

Preserve immediate raw-content delivery and record one channel-level outcome per effective channel.

- [ ] **Step 5: Verify both systems**

Run: `python3 -m unittest discover -s tests -q`, `pytest apps/api/tests -q`, and `npm run build --prefix apps/web`

Expected: all pass; a repeated Bark route sends only once.
