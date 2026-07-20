# Product-grade pack UX — the home hub, rich interaction, and confirmations

A pack is not "a bag of tools the agent picks from at random" — it's a **guided product with a front
door**. Three patterns turn a set of flows into that product. Every shipped reference pack uses all three;
your pack should too. (`octwin init` scaffolds the home hub for you — extend it.)

---

## 1. The home flow — your pack's front door

**Every pack ships a `home` flow: the menu / navigation hub.** It's the human-facing table of contents for
your tools, and the FIRST thing the agent calls on a greeting, an unclear/off-topic message, or "what can you
do?". Wire it **first** in `manifest.flows` and in the agent's `tools`, as a normal agent-callable tool (NOT a
pre/post-turn hook). Its `description` is the trigger the agent reads.

Minimal skeleton — a pure-render `list_picker` with one row per capability, each routing a tap straight to
that tool via `on_select.invoke` (no LLM turn):

```yaml
# flows/tools/home.flow.yaml
flow_id:     home
version:     1
description: Main menu / welcome. Call on a greeting, a vague or off-topic message, or "what can you do?".
input:
  message: { type: string, optional: true, describe: 'ONE short greeting line YOU compose (optional).' }
entry:
  - else: true
    goto: show
steps:
  show:
    - render:
        render_intent: list_picker            # the universal chooser
        header: '{$t("home.header")}'
        body:   '$coalesce($input.message, $t("home.greeting"))'   # agent's line if given, else default
        cta:    '{$t("home.cta")}'             # WhatsApp list-button label (required for list_picker)
        items:                                 # STATIC inline rows — one per capability
          - { title: '{$t("home.row_a")}', on_select: { invoke: browse } }
          - { title: '{$t("home.row_b")}', on_select: { invoke: book, with: { mode: quick } } }
      memory_note: 'Showed home menu.'         # English breadcrumb
    - end: applied
```

**Which rows?** Mirror the pack's PRIMARY journeys — one row each. EXCLUDE: home itself; drill-down / detail
tools reached *from* a journey (a product/record detail, a reply-to-case); and pure free-text Q&A tools.

**Personalization layering** (add only what your domain needs):

- **Static** (ecommerce, clinic) — just the menu, copy from locale. Start here.
- **State-aware** (kaiian) — run a `do: case_list` / `record_list` + `assign` a count BEFORE the render, and
  interpolate it into the body (`'🎫 Your open tickets: {count}'`). Live, no identity needed.
- **Full** (real-estate) — a `pre_turn_flows` identity flow publishes `$pre_turn.user`; home reads it for a
  `db_rpc` counts summary + a 3-tier greeting fallback (agent's `$input.message` → name greeting → generic).

**Home checklist:** first in `flows` + `tools` · single optional `message` input · `entry: else→goto` ·
`list_picker` with `on_select.invoke` rows · `memory_note` on the render · all copy in `home.locale.<lang>.yaml`
· rows = a 1:1 mirror of the primary journeys.

---

## 2. Rich interaction — icons + images

A menu of bare words feels like a form; a couple of icons and a photo make it feel like a product.

- **Icons (emoji).** Lead row titles, headers, and button labels with a relevant emoji — it makes options
  scannable at a glance. Every reference pack does this: `🏠` `📍` `🗓️` `🎫` `📦`, and `✅` / `❌` on
  confirm/cancel buttons. Put the emoji in the locale string (`row_browse: '📦 تصفّح'`), not the YAML.
- **Images.** When rows carry a photo, render a **`carousel`** (image cards) instead of a text `list_picker`;
  map the record's image field into `item_template.image_url`. A stored `MEDIA-…` ref (or a raw asset UUID)
  **auto-resolves to a URL at render** — pass the ref directly, never a URL. See
  [data-and-render.md](data-and-render.md) (carousel + `image_url`) and the platform-kb `media.md`.
  Litmus: photos → `carousel`; text choices → `list_picker`.

---

## 3. Confirmations — preview before a committing action

Before anything that **commits** (books a slot, places an order, submits a lead, publishes a listing), show a
preview and get an explicit yes. Don't rely on the agent to "ask first" in prose — use the platform's
**`approve_apply`** template: it renders a `detail_card` with confirm / cancel buttons, pauses, and resumes
into one of three steps based on the choice.

```yaml
  confirm:
    - use: approve_apply
      with:
        preview:       '{$t("book.preview", { doctor: $state.doctor_name, slot: $state.slot_label })}'
        apply:         do_book        # step id to goto on ✅
        abort:         cancelled      # step id to goto on ❌ (or `abort_render:` for an inline card)
        on_other:      collect_fields # user typed instead of tapping → re-enter collect (optional)
        confirm_title: '{$t("book.confirm")}'   # e.g. 'تأكيد ✅'  (defaults are Arabic)
        cancel_title:  '{$t("book.cancel")}'    # e.g. 'إلغاء ❌'
        memory_note:   'Asked to confirm the booking.'
  do_book:
    - do: booking_reserve
      args: { resource_record_id: '$state.doctor_id', slot_start: '$state.slot_start' }
      outputs: { ok: [{ goto: booked }], full: [{ goto: confirm }], invalid_slot: [{ goto: confirm }], failed: [{ end: failed }] }
```

The approve/cancel buttons carry their decision on the `on_select.control` channel (`$control.approved`), so
the flag never leaks into `$input` / the collect bag. Shipped users: clinic `book-appointment`, ecommerce
`cart`/`orders`, real-estate `lead-action`/`publish-listing`. The full param list is in the template
(`approve_apply`) — surfaced in the platform-kb templates catalog.

---

**Where `on_select` goes** (the one shape that's easy to get wrong) is in [flows.md](flows.md) — a per-intent
table: static picker rows carry `on_select` at the ROW level; card intents (`carousel`/`detail_card`) carry it
inside `buttons: [ … ]`.
