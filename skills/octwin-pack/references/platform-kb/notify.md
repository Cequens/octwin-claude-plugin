# 18 — Platform `notify` (cross-user delivery)

**Status:** AUTHORITATIVE. Source files cited here are canonical truth.

**Pair with:** [12-pack-authoring.md](12-pack-authoring.md) (conventions) · [13a-dsl-reference.md](13a-dsl-reference.md) (step grammar) · [16-platform-stdlib-templates.md](16-platform-stdlib-templates.md) (other stdlib templates).

---

## §1 The problem

Pack code routinely needs to "send a card to user B from user A's turn":
- Real estate: buyer takes an offer → seller's WhatsApp gets a notification card.
- Real estate: buyer↔seller relay messages on a lead.
- Healthcare: doctor's reply lands in patient's thread.
- E-commerce: order-status update lands in customer's thread.

Pre-Phase 9, each pack reinvented the plumbing: resolve recipient → call `messaging.send*` directly to their channel handle → log to a pack-specific table → mark unread → on recipient's next turn, run a pack-specific hook that reads unread + injects as memory notes.

That ceremony moves into the platform. **`notify` is one DSL primitive that ANY flow can call**, channel-agnostic, with per-channel strategies that respect each channel's nature.

---

## §2 DSL: `do: notify`

`notify` is a platform-builtin primitive — every flow's primitives map gets it auto-injected at FlowTool construction ([pack/tools.ts](../src/platform/pack/tools.ts)). YAML calls it the same way as a pack primitive:

```yaml
- do: notify
  args:
    to:
      # Either form accepted:
      contact_id: '$lead.seller.contact_id'
      # OR
      channel:    whatsapp
      channel_contact_handle: '$lead.seller.channel_contact_handle'
    intent:
      render_intent: detail_card
      header:  '{$t("leads.notify.offer.header", { buyer: $buyer.name })}'
      body:    '{$t("leads.notify.offer.body",   { amount: $amount, ref: $ref })}'
      buttons:
        - { title: 'قبول',  on_select: { invoke: leads, with: { action: send_message, lead_id: '$lead_id', kind: accept  } } }
        - { title: 'رد',    on_select: { invoke: leads, with: { action: send_message, lead_id: '$lead_id'                 } } }
    memory_note:   'Pending offer from {$buyer.name} on {$ref}: {$amount}'   # injected into recipient's agent thread
    visibility:    permanent                                                  # permanent | ephemeral | diagnostic
    echo_delivery: true                                                       # fire follow-up notify back to source on WA delivery confirm
  outputs:
    delivered: [...]   # provider accepted; provider_message_id set when channel returns one
    pending:   [...]   # recipient unreachable now (web offline); drain replays on next inbound
    failed:    [...]   # contact_not_found / channel-without-strategy / adapter error
```

Omitting `outputs:` entirely makes notify fire-and-forget — the flow continues to the next step regardless of delivery state. Use this for "best-effort" sends where the source flow's UX shouldn't branch on result (the typical real-estate buyer→seller notification).

### §2.1 Recipient addressing — two forms

**Form A** (preferred when caller has a `contact_id`):
```yaml
to: { contact_id: '$some.contact_id' }
```

**Form B** (when caller only has a channel handle — phone number, web session id):
```yaml
to:
  channel:    whatsapp
  channel_contact_handle: '$user.channel_contact_handle'
```

The notify core ([notify.ts](../src/platform/notify/core/notify.ts)) resolves either form to a `Contact` row via `contact-store.getByChannelId` / `getById`. **Cross-pack tip:** when the pack stores its own `User` model (real-estate does — keyed by channel_contact_handle) and the user might be on any channel, look up the canonical channel via `contact-store.getByMetadata(tenantId, 'pack_user_id', user.id)` before calling notify. The pack's `toContact` mapper writes `metadata.pack_user_id = user.id` at first-touch, so this is a one-query lookup. The real-estate flow composes this cascade declaratively via the [`resolve-recipient-contact`](../src/packs/real-estate/templates/resolve-recipient-contact.template.yaml) pack template (cached `platform_contact_id` → `resolve_contact_by_metadata` → skip) before `do: notify`.

### §2.2 Visibility tiers

The `visibility` arg rides into the recipient's `conversation_events` row metadata:
- `permanent` (default) — agent reads on every subsequent turn (normal message-history entry).
- `ephemeral` — agent reads ONCE on next turn (the platform's "tool action" tier — surfaces a one-shot context note without polluting permanent history).
- `diagnostic` — never agent-visible; admin console only. Use for delivery echoes, system noise.

---

## §3 Per-channel strategies — "consider the channel's nature"

Each `Channel` registers a `NotifyStrategy` ([strategies.ts](../src/platform/notify/strategies.ts)). The notify core selects the strategy by `contact.channel` and delegates wire delivery. The two built-ins:

### §3.1 WhatsApp (the WA strategy in [strategies.ts](../src/platform/notify/strategies.ts))

Push-native. Hands the channel-agnostic UiIntent straight to `MessagingAdapter.sendIntent({ to, intent })` — the adapter already runs `renderIntent + compileOnSelectsInTree` for arbitrary intents. Returns `delivered` + `provider_message_id` (Meta's wamid) on success, `failed` + reason on adapter error.

Delivery confirmation comes from the WA status webhook (see §5).

### §3.2 Web (the web strategy in [strategies.ts](../src/platform/notify/strategies.ts))

Pull-native. Tries `pushEnvelope(projectId, channel_contact_handle, hint)` against the live SSE stream:
- **Active stream** → render lands in the widget immediately → `delivered`.
- **No active stream** (tab closed) → persist row as `pending` → drain hook replays it next time the user connects + sends any inbound.

The web channel has no out-of-band push — this asymmetry is THE "channel nature" the notify abstraction encodes.

### §3.3 Registering a new channel

```typescript
// at boot
import { registerStrategy } from 'src/platform/notify'

registerStrategy({
  channel: 'slack',
  deliver: async ({ contact, intent }) => {
    // your slack send; return { state: 'delivered', provider_message_id: <slack ts> }
    // or { state: 'pending' } for offline + queued
    // or { state: 'failed', failure_reason: '<msg>' }
  },
})
```

Future channels (slack, sms, voice) plug in here; no notify-core change required.

---

## §4 Persistence — `notifications` table

[migration 016_notifications.sql](../src/platform/db/migrations/016_notifications.sql). Every notify call writes exactly one row.

| Column                    | Type        | Notes |
|---|---|---|
| `id`                      | uuid        | PK |
| `tenant_id`, `project_id` | uuid        | Scope |
| `recipient_contact_id`    | uuid        | FK → contacts |
| `recipient_conversation_id` | uuid      | FK → conversations (open conversation for the recipient in this project) |
| `channel`                 | text        | enum: whatsapp / slack / web / voice / sms |
| `intent_json`             | jsonb       | The full UiIntent — replayed on drain |
| `memory_note`             | text        | Optional; injected into recipient's Mastra thread |
| `visibility`              | text        | permanent / ephemeral / diagnostic |
| `state`                   | text        | pending / delivered / seen / failed |
| `provider_message_id`     | text        | Channel-native delivery id (WhatsApp → wamid; web → SSE event id) |
| `delivered_at`, `seen_at` | timestamptz | Lifecycle stamps |
| `failure_reason`          | text        | Populated only when `state = failed` |
| `source_contact_id`       | uuid        | Audit: who triggered the notify |
| `source_flow_id`          | text        | Audit: which flow called notify |
| `echo_delivery`           | boolean     | When true, fire follow-up notify back to source on WA delivery confirm |

**State lifecycle:**
- `pending` → strategy attempted; either web-offline-queued OR no provider_message_id received yet.
- `delivered` → strategy confirmed (WA wamid back OR SSE push OK).
- `seen` → drain hook replayed on recipient's next inbound; row no longer surfaces.
- `failed` → unrecoverable; `failure_reason` set; the YAML's `failed:` port handles it.

---

## §5 Inbound integrations

### §5.1 WA delivery webhook → `delivered` + echo

[dispatcher.ts](../src/platform/pack/dispatcher.ts) hooks the inbound `status === 'delivered'` event:
- Calls `handleDeliveryStatus({ provider_message_id, delivered_at })` ([delivery-webhook.ts](../src/platform/notify/delivery/delivery-webhook.ts)).
- Flips the matching `notifications` row to `state = delivered`.
- When `echo_delivery = true` AND the row has a `source_contact_id`, fires a follow-up notify back to the source — a "✅" diagnostic-tier card so the original sender learns their message reached the other party. Real-estate's relay flow uses this.

### §5.2 Pre-turn drain hook

[pre-turn-drain.ts](../src/platform/notify/delivery/pre-turn-drain.ts) — invoked by [agent/runs/run-agent-turn.ts](../src/platform/agent/runs/run-agent-turn.ts) BEFORE every recipient inbound. For each pending notification on the recipient's open conversation:

1. **Memory note injection** — write `row.memory_note` to the recipient's Mastra thread via `createMastraThreadSink`. Next `agent.generate()` sees it as a `[TOOL_ACTION: …]`-style pre-turn briefing.
2. **Web replay** — if `channel = web`, push the persisted `intent_json` to the live SSE stream so the recipient sees the card now that they're connected.
3. **Mark `seen`** — row transitions `pending → seen`, won't fire again.

This replaces packs' bespoke "collect_pre_turn_notes" hooks. Real-estate's `relay-briefings.ts` retired in Phase 9.

---

## §6 Programmatic exposure (non-flow, platform-internal callers)

```typescript
import { notify } from '#platform/notify/boundaries.js'

await notify({
  to:     { contact_id: '...' },
  intent: { render_intent: 'text_card', body: 'Hello' },
  memory_note: 'Background job sent reminder',
})
```

The function shares the same code path as the DSL primitive — strategies, store, drain, all identical. Use this form from platform-internal admin routes / background jobs (it's NOT on the pack `#sdk` surface — pack code uses the `do: notify` primitive). Inside flows, prefer `do: notify` so the YAML stays declarative.

---

## §7 Real-estate worked example (Phase 9 migration)

The pack retired three pre-existing code paths and replaced them with `do: notify`:

| Before (Phase 8) | After (Phase 9) |
|---|---|
| `notify_seller_async` fire-and-forget IIFE in `lead-action.primitives.ts` (post-mutation) | The `apply` step mutates via `db_update`, then `apply_notify` composes the seller-notify body in YAML (`$t` + `$unit_ref`) and runs `do: notify` before the buyer's success render. (The interim `apply_lead_action` primitive was retired in Phase 4d.2 — its data access is now explicit YAML.) |
| `relay-message` flow (full validate + sanitize + preview + send + delivery-echo chain) | Folded into `leads` flow's `action: send_message`. The pure `build_relay_intent` primitive (over a `db_rpc(lead_detail)`-fetched lead) builds the recipient intent + sender preview; the `send` step calls `do: notify`. (Replaced the retired `sanitize_message_intent` primitive in Phase 4d.2.) `relay-message.flow.yaml` deleted. |
| `relay-delivery.ts` WA-status echo handler | Platform `handleDeliveryStatus` + `echo_delivery: true` on the notify call |
| `relay-briefings.ts` + `collect_pre_turn_notes` hook | Platform `pre-turn-drain.ts` runs for every recipient inbound, generically |
| `relay_messages` pack table | Retired; data lives in `notifications` + `conversation_events`. Migration `007_relay_messages_deprecated.sql` renames the table |

Net LOC retired from the pack: ~300. The pack stays focused on domain logic (lead status mutations, offer analysis, sanitization rules); delivery + ledger + thread mirroring are platform concerns.

---

## §8 Migrations

### §8.1 New tenants (provisioning)

Automatic via `applyPackMigrations(...)` ([provision-tenant.ts](../src/platform/provisioning/provision-tenant.ts)). No action required.

### §8.2 Existing tenants

The platform-side `notifications` table goes on the application DB (`DATABASE_URL`):
```bash
npm run platform:db:migrate
```

The pack-side `007_relay_messages_deprecated.sql` goes per-project via the runner:
```bash
npx tsx scripts/apply-pack-migrations.ts real-estate
```

For tenants whose tracking table is empty (pre-tracking-table provisioning), backfill first:
```bash
npx tsx scripts/apply-pack-migrations.ts real-estate --backfill-up-to 006_lead_unread.sql
```

The runner is idempotent — `_pack_migrations` tracks applied files per `(pack_id, filename)`; re-runs are safe.

---

## §9 Out of scope (today)

- **Cross-project / cross-tenant notify** — notify operates within the current project context only. Future `notify_cross_project` would require explicit tenant-scoped permissions.
- **Bulk / broadcast notify** — single-recipient only. Bulk-sends need rate limits, batching, opt-out lists.
- **Scheduling** — "send now (or queue for next reconnect)" only. No "deliver in 30 minutes" cron-style.
- **Notification-specific render intents** — reuses the existing 8 canonical intents.
- **Slack / SMS / Voice strategies** — registry is open; no implementations shipped in Phase 9.
- **Opt-out / quiet-hours / preference center** — recipient consent management is a separate effort.
