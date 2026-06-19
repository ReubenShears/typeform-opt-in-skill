---
name: typeform-opt-in
description: >-
  Build a calibrated client opt-in Typeform in the Optimally Typeform account end to end
  (3 qualification questions, ICP-calibrated revenue brackets, disqualification routing,
  /confirmed vs /uq-confirmed redirects, hidden URL params, and the lead-survey webhook),
  driven by a single client identifier. Use this WHENEVER the user wants to create, build,
  clone, or calibrate a client opt-in form / opt-in survey / lead-qualification Typeform for
  a funnel — e.g. "build the opt-in for Disruptors", "make ExitEngine's Typeform", "clone the
  opt-in template for <client>", "set up the lead survey for <client>" — and ALSO when a
  parent funnel-build skill (like framer-vsl-funnel) or a client-onboarding flow needs the
  opt-in created as a step. Composable: it takes one input (the client's Client ID) and reads
  everything else from Baserow, so other skills can invoke it with a single line.
---

# Typeform Opt-In Builder

Builds the standard Optimally client opt-in form via the Composio Typeform tools. The opt-in is
a 3-question lead-qualification survey that pixels/redirects qualified leads to `/confirmed` and
quietly routes the worst leads to `/uq-confirmed` (so you don't waste pixel/ad signal on them).
Every client form lives in the **Optimally** Typeform account — never the client's own.

## Input contract (so other skills can call this)

The only required input is the client's **Client ID** — the `UPPERCASE_SNAKE` value in the
Baserow `Client Data` table (e.g. `DISRUPTORS_MEDIA`). Everything else is read from that row.

A parent skill invokes this by passing the Client ID and (optionally) reading back the
new form's public URL, which this skill writes to the client's `Typeform URL` Baserow field.

## Prerequisites

- The Composio Typeform connection must be active under alias **`optimally-internal`**.
  **Every** `TYPEFORM_*` tool call needs `account: "optimally-internal"` (explicit selection is on).
- Baserow MCP access to `Client Data` (table id `1000911`).

If you need the exact API payloads, refer to `references/build-notes.md` — it has copy-paste
JSON for the create + update calls and the non-obvious Typeform API gotchas. Read it before
your first build; the API has several traps that will waste a build if you don't know them.

## Workflow

### Step 1 — Read the client from Baserow

List `Client Data` (table `1000911`), find the row whose `Client ID` matches the input. Pull:
- **Company** → the form title becomes `<short brand name> | Opt-In` (match the existing
  naming, e.g. `Tradeflow | Opt-In`, not the legal name).
- **Offer** (free text) → parse the **ICP minimum monthly revenue** from it
  (e.g. "...need a solid base (over 20k/mo)..." → `$20k/mo`). This drives Q3's brackets.
- **Base Funnel Domain** → the root for the redirects, e.g. `https://www.client.com`.
  If it's blank, ask the user for it before continuing (the redirects can't be built without it).

Use the rest of the row (Best Product/Service, Pain Points, Consistent Client Persona, Industry)
as *context* to write answer options that actually fit this client's audience.

### Step 2 — Calibrate the 3 questions

Keep the three-part structure; tailor the wording and answers to the client:

1. **Q1 — current situation** (identifies the lead's bucket). 3–4 options spanning maturity
   stages of the client's target customer.
2. **Q2 — main blocker** (what's stopping them). 3–4 options matching the client's known
   pains / objections.
3. **Q3 — financial qualifier** (revenue brackets). **Calibrate the lowest bracket to roughly
   HALF the ICP minimum** — that boundary is the disqualification line, so only genuinely
   terrible leads get filtered. Example: ICP min `$20k/mo` → lowest bracket `Under $10k/mo`,
   then `$10k to $20k/mo`, `$20k to $50k/mo`, `$50k+/mo`.

The reason for the ½-ICP line: a lead at, say, $15k against a $20k ICP is still worth pixeling
and nurturing; only the leads at less than half the floor are clearly unworkable.

### Step 3 — Copy rules (non-negotiable)

Survey results get exported as **comma-separated values**, and the brand voice avoids em dashes.
So in every title, answer, and description:
- **No commas** (`,`) — they break the CSV.
- **No em dashes or en dashes** (`—` `–`) — rewrite with "and", "so", "to", a full stop, or a colon.
- Apostrophes (`'`) are fine.

Rephrase rather than punctuate. "Growing, but inconsistent" → "Growing but inconsistent".
"$10k–$20k/mo" → "$10k to $20k/mo".

### Step 4 — Build the form (create then update)

Build in the Optimally account: workspace `https://api.typeform.com/workspaces/JCx6nx`,
theme `https://api.typeform.com/themes/M2ZyLjyp`, `show_typeform_branding: false`,
`hide_navigation: true`, progress bar on (`proportion`), language `en`.

- **Hidden fields (URL params):** always exactly `["full_name", "email"]`.
- Greeting: use a plain `"Hey there 👋 ..."` — do **not** try to pipe `{{hidden:full_name}}`
  into a title; the API rejects piping (see build-notes). Tell the user they can add the
  `{{full_name}}` mention in the Typeform editor afterwards if they want it.
- Do **not** add a "Webhook Link" statement slide. The template carries one only as a human
  marker; the webhook is applied via the API (Step 6), so the slide is unnecessary.

**Use the create-then-update pattern** (this is the reliable path — see build-notes for why):
1. `TYPEFORM_CREATE_FORM` with the fields + a single `default_tys` ending. Capture the new
   `form_id`.
2. `TYPEFORM_UPDATE_FORM` with the full final structure using **your own stable refs** for the
   endings (`end_confirmed`, `end_uq`) and for the Q3 disqualifying choice (`rev_dq`). On update,
   custom refs are preserved, so the logic can reference them reliably.

### Step 5 — Endings + disqualification routing

Build two `url_redirect` endings from `Base Funnel Domain` (note the consistent `/` convention):
- `end_confirmed` → `<domain>/confirmed`, titled by action e.g. `Redirect to /confirmed (qualified)`
- `end_uq` → `<domain>/uq-confirmed`, titled e.g. `Redirect to /uq-confirmed (unqualified)`

(Keep a `default_tys` screen too; Typeform requires a default.)

Logic on Q3: first action routes the **lowest** bracket (`rev_dq`) to `end_uq`; a fallback
`always` action routes everyone else to `end_confirmed`. Exact JSON is in build-notes.

### Step 6 — Apply the webhook (get the URL from the template, never hardcode)

The live n8n `lead-survey` endpoint is stored as a **marker inside the template form** — that is the
single source of truth (which is why this skill never hardcodes it, and why this public copy is safe
to share). Retrieve it at build time:

1. `TYPEFORM_GET_FORM` the `DUPLICATE FOR OPT-IN` template, form id **`ErAr5UtG`** (account
   `optimally-internal`).
2. Find the field with `type: "statement"` titled `Webhook Link 👇` and read its
   `properties.description` — that string is the live webhook URL.
3. `TYPEFORM_CREATE_OR_UPDATE_WEBHOOK` with `form_id` = the new form, `tag` = `lead-survey`,
   `url` = the URL you just read, `enabled` = true.

Reading it from the template (rather than a constant) means: the public copy works without secrets,
and if Optimally ever changes the n8n endpoint, they update the template slide once and every future
build picks it up. The same endpoint is correct for every client (it keys off the form/hidden fields
downstream). Do NOT add a "Webhook Link" slide to the new form — it's a template marker only.

### Step 7 — Write the URL back to Baserow (composability)

Update the client's `Typeform URL` field in `Client Data` with the new form's public URL
(`https://optimally.typeform.com/to/<form_id>`) so parent skills and humans can find it.

### Step 8 — Report what's done and what's manual

Confirm: questions, brackets (and the DQ line), both redirects, hidden params, webhook. Flag the
one manual step: the `{{full_name}}` greeting personalisation (editor only). If `Base Funnel Domain`
was missing or on the wrong row, say so.

## Verify before declaring done

`TYPEFORM_GET_FORM` the new form and check: `hidden` contains full_name + email; Q3 logic has the
DQ jump to `end_uq` and the fallback to `end_confirmed`; both endings are `type: url_redirect` with
the right URLs; no stray "Webhook Link" slide; no commas or dashes in any label. Then
`TYPEFORM_LIST_WEBHOOKS` to confirm `lead-survey` is enabled.
