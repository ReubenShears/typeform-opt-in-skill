---
name: typeform-opt-in
description: >-
  Build a calibrated client opt-in Typeform in the Optimally Typeform account end to end
  (3 qualification questions, ICP-calibrated revenue brackets, disqualification routing,
  Qualified/Unqualified Calendar redirects, hidden URL params, and the lead-survey webhook),
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
a 3-question lead-qualification survey that redirects qualified leads to the **Qualified Calendar**
(Calendly) and routes the worst leads to the **Unqualified Calendar**, so booking energy and ad/pixel
signal go to the right place. Funnel flow: opt-in → Typeform → Calendly → confirmation page. Every client
form lives in the **Optimally** Typeform account — never the client's own.

## Input contract (so other skills can call this)

The only **required** input is the client's **Client ID** — the `UPPERCASE_SNAKE` value in the
Baserow `Client Data` table (e.g. `DISRUPTORS_MEDIA`). Everything else can be read from that row.

Everything this skill needs lives in that one row: `Company`, the qualification context (`Offer`,
`Pain Points`, etc.), and the **`Qualified Calendar URL`** / **`Unqualified Calendar URL`** the form
redirects to. It writes the new form's public URL back to the row's **`Typeform URL`** field. So a parent
skill (e.g. `framer-vsl-funnel`) invokes it with just the Client ID and reads `Typeform URL` back — no
other inputs needed.

## Prerequisites

- The Composio Typeform connection must be active under alias **`optimally-internal`** (the Optimally
  Typeform account), reachable via the `u=optimally` operator connector.
- **How the calls run:** the `TYPEFORM_*` tools execute through Composio's
  **`COMPOSIO_MULTI_EXECUTE_TOOL`** — each call passes the tool slug (e.g. `TYPEFORM_CREATE_FORM`) **and**
  `account: "optimally-internal"`. Explicit account selection is ON, so a call without the account errors
  rather than guessing. Wherever this skill says "call `TYPEFORM_X`", that means a
  `COMPOSIO_MULTI_EXECUTE_TOOL` run of `TYPEFORM_X` with that account.
- Baserow MCP access to `Client Data` (table id `1000911`).

If you need the exact API payloads, refer to `references/build-notes.md` — it has copy-paste
JSON for the create + update calls and the non-obvious Typeform API gotchas. Read it before
your first build; the API has several traps that will waste a build if you don't know them.

## Workflow

### Step 1 — Read the client from Baserow

List `Client Data` (table `1000911`), find the row whose `Client ID` matches the input. Pull:
- **Company** → the form title becomes `<short brand name> | Opt-In` (match the existing
  naming, e.g. `Tradeflow | Opt-In`, not the legal name).
- **ICP minimum monthly revenue** (drives Q3's brackets) → **estimate** it from the client's ICP signals.
  The **`Offer`** field often states it outright (e.g. "...need a solid base (over 20k/mo)..." → `$20k/mo`);
  otherwise infer from `Consistent Client Persona`, `ICP Examples`, `Current Monthly Revenue`,
  `Desired Monthly Revenue`, `Industry`, and `Best Product/Service`. **Do NOT use `Floor Amount`** — that
  is Optimally's retainer floor with the client (agency pricing), not the lead's revenue. If nothing in the
  row implies an ICP, make a sensible industry-appropriate estimate and flag it in the report.
- **`Qualified Calendar URL`** and **`Unqualified Calendar URL`** → the Calendly links the form redirects
  to (Step 5). Usually empty at build time (the human sets Calendly up during the manual finish), so treat
  them as optional: empty just means use the clearly-labelled `https://www.calendly.com` placeholder.

Use the rest of the row (Best Product/Service, Pain Points, Frequent Objections, Main Bottleneck,
Consistent Client Persona, Industry) as *context* to write answer options that fit this client's audience.

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
and nurturing; only the leads at less than half that ICP line are clearly unworkable.

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
   endings (`end_qualified`, `end_unqualified`) and for the Q3 disqualifying choice (`rev_dq`). On update,
   custom refs are preserved, so the logic can reference them reliably.

### Step 5 — Endings + disqualification routing (redirect to Calendly)

The funnel flow is opt-in → Typeform → **Calendly** → confirmation page. So the two `url_redirect`
endings send leads to the client's Calendly calendars (Calendly itself then redirects to `/confirmed` /
`/uq-confirmed` after booking — that redirect is set in the Calendly UI, not here). Build:
- `end_qualified` → titled **`Qualified Calendar`**; `redirect_url` = the row's `Qualified Calendar URL`
  if set, else the placeholder `https://www.calendly.com`.
- `end_unqualified` → titled **`Unqualified Calendar`**; `redirect_url` = the row's
  `Unqualified Calendar URL` if set, else `https://www.calendly.com`.

The clear ending titles tell the human exactly which Calendly link to paste if the placeholder is still
there. (Keep a `default_tys` screen too; Typeform requires a default.)

Logic on Q3: first action routes the **lowest** bracket (`rev_dq`) to `end_unqualified`; a fallback
`always` action routes everyone else to `end_qualified`. Exact JSON is in build-notes.

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

Confirm: questions, brackets (and the DQ line), both Calendar redirects (real URL or the labelled
`calendly.com` placeholder), hidden params, webhook. Flag the manual steps: the `{{full_name}}` greeting
personalisation (editor only); and if either Calendar redirect is still the placeholder, that the human
pastes the real Calendly link and sets that calendar's post-booking redirect to the confirmation page.

## Verify before declaring done

`TYPEFORM_GET_FORM` the new form and check: `hidden` contains full_name + email; Q3 logic has the
DQ jump to `end_unqualified` and the fallback to `end_qualified`; both endings are `type: url_redirect`
titled `Qualified Calendar` / `Unqualified Calendar`; no stray "Webhook Link" slide; no commas or dashes
in any label. Then `TYPEFORM_LIST_WEBHOOKS` to confirm `lead-survey` is enabled.
