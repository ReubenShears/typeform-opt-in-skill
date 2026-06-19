# Typeform build notes (exact payloads + API gotchas)

Reference implementation: a client opt-in built end to end. All calls go through the Composio
Typeform tools with `account: "optimally-internal"`.

## The non-obvious Typeform API traps (read these first)

1. **No piping on write.** `TYPEFORM_CREATE_FORM` / `UPDATE_FORM` reject `{{hidden:full_name}}`
   in a title with `INVALID_PIPING`. Use a plain greeting. Name-personalisation is editor-only.

2. **Custom ending refs: create reassigns, update preserves.** On `CREATE_FORM`, the only
   thank-you ref it accepts is the reserved `default_tys`; any custom ending ref you pass gets
   silently reassigned, so logic that points at it fails with `UNKNOWN_THANKYOU_REFERENCE`.
   On `UPDATE_FORM`, custom refs you provide **are** kept. → Always create minimal, then update
   with the real structure. (Field refs like `q1_situation` and choice refs like `rev_dq` are
   preserved on update too.)

3. **Redirect endings must be `type: "url_redirect"`** with `properties.redirect_url`. A normal
   `thankyou_screen` carrying a `redirect_url` becomes a hidden-button screen
   (`button_mode: default_redirect`, `show_button: false`) that does **not** auto-redirect — it
   just dead-ends on the "All done" text. `url_redirect` auto-redirects on completion.

4. **`PATCH_FORM` cannot touch logic or fields.** Allowed paths are only
   `/settings/*`, `/theme`, `/title`, `/workspace`, `/field_reference`. For logic/fields/endings,
   use `UPDATE_FORM` (full PUT — include the whole definition or fields get deleted).

5. **Explicit account every call.** Multi-account selection is on, so omit `account` and the call
   errors. Always pass `account: "optimally-internal"`.

## Step 1 — CREATE_FORM (minimal, just to get a form_id)

You can create the full field set here; just keep ONE ending (`default_tys`) and simple logic,
then fix routing/endings in the update. Minimal viable create:

```json
{
  "title": "<Brand> | Opt-In",
  "type": "quiz",
  "hidden": ["full_name", "email"],
  "theme": {"href": "https://api.typeform.com/themes/M2ZyLjyp"},
  "workspace": {"href": "https://api.typeform.com/workspaces/JCx6nx"},
  "settings": {"language": "en", "is_public": true, "progress_bar": "proportion",
    "show_progress_bar": true, "hide_navigation": true, "show_typeform_branding": false,
    "show_question_number": false, "show_time_to_complete": true, "autosave_progress": true,
    "free_form_navigation": false},
  "fields": [ /* q1_situation, q2_blocker, q3_revenue (see below) */ ],
  "thankyou_screens": [
    {"ref": "default_tys", "title": "All done! Thanks for your time.",
     "type": "thankyou_screen", "properties": {"show_button": false, "share_icons": false}}
  ]
}
```

## Step 2 — UPDATE_FORM (the real structure, with stable refs + routing)

This is the call that does the work. Note: `fields` use stable `ref`s; the Q3 disqualifying
choice gets `ref: "rev_dq"`; the two redirect endings get `ref`s `end_confirmed` / `end_uq`; the
logic references those.

```json
{
  "form_id": "<new form id>",
  "title": "<Brand> | Opt-In",
  "type": "quiz",
  "hidden": ["full_name", "email"],
  "theme": {"href": "https://api.typeform.com/themes/M2ZyLjyp"},
  "workspace": {"href": "https://api.typeform.com/workspaces/JCx6nx"},
  "settings": {"language": "en", "is_public": true, "progress_bar": "proportion",
    "show_progress_bar": true, "hide_navigation": true, "show_typeform_branding": false,
    "show_question_number": false, "show_time_to_complete": true, "autosave_progress": true,
    "free_form_navigation": false},
  "fields": [
    {"ref": "q1_situation", "type": "multiple_choice", "validations": {"required": true},
     "title": "Hey there 👋 what best describes *your situation right now?*",
     "properties": {"description": "We'd love to get to know you a little better!",
       "allow_multiple_selection": false, "allow_other_choice": false, "vertical_alignment": true,
       "choices": [{"label": "<bucket option 1>"}, {"label": "<bucket option 2>"},
                   {"label": "<bucket option 3>"}, {"label": "<bucket option 4>"}]}},
    {"ref": "q2_blocker", "type": "multiple_choice", "validations": {"required": true},
     "title": "And *what's the main thing holding you back* right now?",
     "properties": {"allow_multiple_selection": false, "allow_other_choice": false,
       "vertical_alignment": true,
       "choices": [{"label": "<blocker 1>"}, {"label": "<blocker 2>"},
                   {"label": "<blocker 3>"}, {"label": "<blocker 4>"}]}},
    {"ref": "q3_revenue", "type": "multiple_choice", "validations": {"required": true},
     "title": "Finally *what's your current monthly revenue?*",
     "properties": {"allow_multiple_selection": false, "allow_other_choice": false,
       "vertical_alignment": true,
       "choices": [{"label": "Under $<half-ICP>k/mo", "ref": "rev_dq"},
                   {"label": "$<half-ICP>k to $<ICP>k/mo", "ref": "rev_b2"},
                   {"label": "$<ICP>k to $<hi>k/mo", "ref": "rev_b3"},
                   {"label": "$<hi>k+/mo", "ref": "rev_b4"}]}}
  ],
  "thankyou_screens": [
    {"ref": "end_confirmed", "title": "Redirect to /confirmed (qualified)",
     "type": "url_redirect", "properties": {"redirect_url": "<domain>/confirmed"}},
    {"ref": "end_uq", "title": "Redirect to /uq-confirmed (unqualified)",
     "type": "url_redirect", "properties": {"redirect_url": "<domain>/uq-confirmed"}},
    {"ref": "default_tys", "title": "All done! Thanks for your time.",
     "type": "thankyou_screen", "properties": {"show_button": false, "share_icons": false}}
  ],
  "logic": [
    {"type": "field", "ref": "q3_revenue", "actions": [
      {"action": "jump",
       "condition": {"op": "is", "vars": [
         {"type": "field", "value": "q3_revenue"}, {"type": "choice", "value": "rev_dq"}]},
       "details": {"to": {"type": "thankyou", "value": "end_uq"}}},
      {"action": "jump", "condition": {"op": "always", "vars": []},
       "details": {"to": {"type": "thankyou", "value": "end_confirmed"}}}
    ]}
  ]
}
```

## Step 3 — Webhook (url read from the template slide)

First read the URL from the template (`TYPEFORM_GET_FORM` form `ErAr5UtG`, then the `Webhook Link 👇`
statement field's `properties.description`). Then:

```json
{"form_id": "<new form id>", "tag": "lead-survey",
 "url": "<url read from the DUPLICATE FOR OPT-IN 'Webhook Link' slide>", "enabled": true}
```

Why read it from the template instead of a constant: the public copy of this skill has the real URL
redacted, and the template carries the live value, so this works with no extra config and survives the
endpoint changing. Deleting/omitting the "Webhook Link" slide from the NEW form does NOT remove the
webhook — the webhook is form-level config, independent of fields.

## Notes
- The template to mirror structurally is `DUPLICATE FOR OPT-IN`, but its choices are placeholder
  `Option 1/2/3` and its redirect points at an old client — calibrate, don't copy verbatim.
- All client opt-ins live in the Optimally account (workspace `JCx6nx`), never the client's own
  Typeform.
- Client alias for connections stays the canonical `Client ID`.
