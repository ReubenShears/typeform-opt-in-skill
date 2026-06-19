# typeform-opt-in

A Claude Code skill that builds a calibrated **client opt-in Typeform** in the Optimally Typeform
account, end to end:

- 3 lead-qualification questions (current situation → main blocker → revenue)
- revenue brackets calibrated to **half the client's ICP** (only the worst leads get disqualified)
- disqualification routing → `/confirmed` (qualified) vs `/uq-confirmed` (unqualified)
- hidden URL params (`full_name`, `email`), correct theme/branding
- the `lead-survey` webhook applied automatically
- CSV-safe copy (no commas, no em dashes)

It runs on a **single input — the client's `Client ID`** (from the Baserow `Client Data` table) —
and reads everything else (company, ICP, funnel domain) from that row, then writes the new form URL
back to Baserow. That single-input contract makes it **composable**: other skills (e.g.
`framer-vsl-funnel` or a client-onboarding flow) can invoke it as one step.

## Install

Copy the `typeform-opt-in/` directory into your Claude skills directory.

## Contents

- `typeform-opt-in/SKILL.md` — the workflow
- `typeform-opt-in/references/build-notes.md` — exact Composio Typeform API payloads + the
  non-obvious API gotchas (piping rejection, create-vs-update ref handling, `url_redirect` endings,
  `PATCH_FORM` limits)

## Requirements

- A Composio Typeform connection (the skill assumes the `optimally-internal` account alias)
- Baserow access to the `Client Data` table
- The `lead-survey` n8n webhook endpoint is replaced with a placeholder in this public copy —
  set it to your own n8n webhook.

Built by Optimally.
