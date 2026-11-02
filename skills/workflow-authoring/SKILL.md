---
name: workflow-authoring
description: Build, edit, test, publish, stop and run respond.io Workflows safely through the respond.io MCP tools. Use when creating a workflow from scratch or a template, changing one, testing it, publishing or stopping it, or running a Shortcut workflow on a contact.
---

# Workflow authoring

A respond.io Workflow is one trigger followed by a tree of steps. These tools edit the same workflows the respond.io builder does, and every run touches real contacts. Work in this order, and get the user's explicit go-ahead before each step that changes something: saving, testing, publishing, stopping, deleting and running.

1. Check the workspace and limits.
2. Start from a template or from scratch.
3. Draft (`create_workflow`) and edit (`update_workflow`).
4. Fix validation problems.
5. Test (`start_test_workflow` / `end_test_workflow`).
6. Publish (`publish_workflow`) or stop (`stop_workflow`).
7. For Shortcut workflows, run on a contact (`trigger_shortcut`).

The tools enforce these rules on their own as well. Some ask the user to confirm before acting; if the user declines, nothing changed — do not retry without asking again.

## 1. Check before acting

- `get_user_profile` gives the connected workspace. Every tool works on that workspace only.
- `list_workflows` shows what already exists, with each workflow's status (`draft`, `published` or `stopped`). Workflow names must be unique.
- `get_workflow` returns a workflow's trigger and steps in the exact form `update_workflow` takes.
- Resolve every name to an id before using it. Workflows store ids, not names:
  - channels: `list_space_channels`
  - tags: `search_tags`
  - lifecycle stages: `list_lifecycles`
  - users (AI agents are users too): `list_space_users`
  - teams: `list_teams`
  - knowledge sources for AI steps: `search_knowledge_sources`
  - other workflows: `list_workflows`
  - Meta / TikTok / Google Sheets: see section 4.
- Tell the user the limits before designing anything larger:
  - exactly 1 trigger
  - at most 100 steps (connectors don't count)
  - a Branch step has 2–10 branches including Else, each with at most 10 conditions
  - waits and question/assignment timeouts are at most 7 days (AI steps: 1 day)
  - no scheduled or time-based triggers

### Supported triggers

`conversationOpened`, `conversationClosed`, `callEnded`, `contactFieldUpdated`, `contactTagUpdated`, `lifecycleUpdated`, `shortcut`, `incomingWebhook`, `manualTrigger`, `ctcAds` (Meta click-to-chat ads), `tiktokCTCAds` (TikTok click-to-chat ads).

### Supported steps

`sendMessage`, `askQuestion`, `assignTo`, `branch`, `updateContactField`, `updateContactTag`, `openConversation`, `closeConversation`, `addComment`, `jumpTo`, `dateTime`, `wait`, `httpRequest`, `addGoogleSheetRow`, `triggerWorkflow`, `updateLifecycle`, `sendConversionsApiEvent`, `sendTikTokLowerFunnelEvent`, `aiAgent`.

Multi-outcome steps lead to their next steps through connector steps: `branchConnector`, `askQuestionConnector`, `assignToConnector`, `dateTimeConnector`, `sendMessageConnector` and `aiAgentConnector`.

If the user asks for something not on these lists, say it isn't supported. Never swap in something else silently.

Only two things need the respond.io workflow builder, because no tool can do them:

- uploading a file for an attachment message
- connecting Google and picking the spreadsheet on a Google Sheets step (section 5)

## 2. Start from a template

Offer a template before building from scratch whenever one matches the user's goal.

1. **Pick.** Call `list_workflow_templates` and shortlist two to four. Match on `tags` (for example `autoresponder`, `routing_assignment`) and the words in the `id`; prefer a lower `difficulty`. For each, say up front what it needs: `business: true` uses Business-plan features, and `meta` lists the plan features it needs, such as `googleSheets` or `httpRequest`. Let the user choose.
2. **Load.** `get_workflow_template` returns `trigger`, `triggerPayload`, `steps` and `needsInput`. `needsInput` lists the blanks `update_workflow` would reject, worded the same as its errors. Draw the template (see [visualizing](references/visualizing.md)) so the user sees what they will get before anything is created.
3. **Fill every `needsInput` entry.** Resolve ids with the lookup tools, and ask the user for content and choices such as message text or a timezone. Never leave placeholder text or invent values. Change the steps the user asks to change, and keep the template's step ids.
4. **Dependencies.** A template with a `triggerWorkflow` step needs its target workflow first — for example, `multi_level_chat_menu` needs `multi_level_chat_menu_sub`. Build the dependency from its own template, then set `workflowId` to its new id.
5. **Save.** `create_workflow` (named after the template unless the user picks a name), then `update_workflow` with the filled graph. Continue with sections 6–9.

If the template has a Google Sheets step, follow section 5.

## 3. Draft and edit

- `create_workflow` makes an empty draft. Its trigger and steps are set with `update_workflow`.
- `update_workflow` **replaces the whole trigger and step list**. Always call `get_workflow` first and send every step back, changed or not. A partial list deletes the steps it leaves out.
- Send the `updatedAt` from your latest read of the workflow: `get_workflow`, `create_workflow`, or the previous `update_workflow`, `publish_workflow` or `stop_workflow` result. If someone saved it in the builder since, the save is refused so their edits aren't lost: tell the user, call `get_workflow` again and re-apply your changes to that version.
- Only draft and stopped workflows can be edited. A published workflow must be stopped first (section 8).
- Graph shape:
  - Every step has a unique string `id` (not `"1"`) and a `parentId`. Exactly one step has `parentId: 1` — the step right after the trigger.
  - A regular step has at most one child.
  - A multi-outcome step (`askQuestion`, `assignTo`, `dateTime`, `sendMessage` with failure handling, `branch`, `aiAgent`) leads on through its connector steps. Children hang off the connector id, not the step id. The server adds the default connectors (for example the question's Success and Failure: Invalid) if you leave them out.
  - A `branch` needs at least one `branchConnector` with `conditions`; the Else branch (`conditions: null`) is added if missing. Branches are checked in `data.connectors` order.
  - `jumpTo` loops back to an earlier regular step by `stepId`, at most `maxJumps` (1–10) times.
- Values from earlier steps are referenced in text as `{{$contact.firstname}}`, `{{$workflow.<variable>}}` and so on: two braces, a `$`, the type, a dot and the name, all lowercase. Contact keys are `firstname`, `lastname`, `name`, `email`, `phone`, `country`, `language`, `id` or a custom field slug. A `{{$workflow.<variable>}}` must be saved by the trigger or a step. Anything else is sent to the contact as typed, so the save refuses it. Workflow variable names must be unique across the trigger and all steps.
- An HTTP Request step's `method` is lowercase: `get`, `post`, `put`, `delete`, `head` or `patch`.
- For a Date & Time step, always ask the user which timezone the schedule uses. Offer the workspace timezone (`"workspace"`) or an IANA name such as `Asia/Kuala_Lumpur`.
- A WhatsApp template message needs a WhatsApp channel on the step and a template from `list_message_templates` for that channel. Fill every `{{n}}` in it.
- Update Contact Field values must fit the field: a list field takes one of its options, a date field `YYYY-MM-DD`, a checkbox `true` or `false`.
- A contact field set in a step, trigger or condition (`contactField`, `saveResponse.field`, a `contactField` condition's `field`) is a field key, not a placeholder key: `firstName`, `lastName`, `email`, `phone`, `countryCode`, `lang` or a custom field slug from `list_custom_fields`. `firstname`, `language` and `country` are only for `{{$contact.<key>}}`.
- Common shapes are in [patterns](references/patterns.md).

## 4. Integration-backed triggers and steps

Copy every account, ad and dataset object exactly as the lookup tool returns it. The builder rejects extra or renamed keys.

- Start with `list_ads_integrations`. It gives the Meta and TikTok connection ids (`botIntegrationId`) and the TikTok ad accounts. If the provider the user needs has no connection, the user connects it in Settings > Integrations — the agent can't.
- **Meta ads trigger (`ctcAds`)**: `list_meta_ad_accounts` gives the ad accounts; save one as `adAccount`. `adsSelectionType` is `"all"` or `"selected"`. For `"selected"`, call `list_ctc_ads` with `platform: "meta"` and exactly one `act_` id, and save the chosen ads as `{ adId, adName }` in `ads`.
- **TikTok ads trigger (`tiktokCTCAds`)**: `adAccounts` come from `list_ads_integrations`. For `"selected"`, `list_ctc_ads` with `platform: "tiktok"` returns each ad's `destination`; put `{ adId, adName }` under `selected.<destination>.ads` (`whatsapp`, `messenger` or `tiktok`).
- **Conversions API event (`sendConversionsApiEvent`)**: `list_meta_datasets` gives the business portfolios. Save one portfolio's id as `businessId`, and at most one of its accounts each as `datasets.facebook`, `datasets.instagram` and `datasets.whatsapp`. `event.name` is `LeadSubmitted` or `Purchase` (case-sensitive). A `Purchase` event needs `event.currency` and `event.value`.
- **TikTok event (`sendTikTokLowerFunnelEvent`)**: `botIntegrationId` is a TikTok connection from `list_ads_integrations`. `payload.event` decides which `payload.properties` it takes:
  - `Spam`, `DisqualifiedConversation`: no properties. Leave `description`, `content_type`, `currency` and `value` null and `contents` empty.
  - `PlaceAnOrder`, `CompletePayment`, `CancelAnOrder`: `currency` and `value` are required.
  - `SubmitForm`: every property is optional.
  - `content_type` is `product` or `product_group`. `contents` holds up to 20 items of `content_id`, `content_category`, `content_name`, `brand`, `price` and `quantity`, all strings. An item with any field set needs a `content_id`.
- **Currency** (TikTok `currency`, CAPI `event.currency`): an ISO 4217 code such as `USD`, or a variable. `value` is a string, such as `"49.90"`.
- The connection must belong to the step's provider: a Meta id on a TikTok step, or a dataset from another business, is refused.
- Show the user the accounts, ads and datasets to choose from. Never pick one on their behalf.

## 5. Google Sheets step (two phases)

`update_workflow` rejects an `addGoogleSheetRow` step until a spreadsheet is chosen, and only the user can choose one, in the builder.

1. Save the rest of the workflow without the step. If it belongs mid-graph, give its would-be child the step's parent for now.
2. Ask the user to add the Google Sheets step at that point in the builder, connect Google, pick the spreadsheet, and save.
3. Call `get_workflow` again. The user's save changed the workflow, and sending an older copy would undo it. Keep the saved `email`, `spreadsheetId` and `spreadsheetName` as they are.
4. Call `get_google_sheet` with `workflowId` and the step's `stepId` to list the tabs, then again with the chosen `sheetName` for its `columns`. Copy the columns as returned.
5. Set `sheetName`, `columns` and `rowValues` (one entry per column, at least one filled), and save with `update_workflow`.

## 6. Validate

- `update_workflow` checks the whole graph and reports **every** problem at once. Nothing is saved until all are fixed. Fix them all and send the full payload again.
- `publish_workflow` validates the saved version again on the platform, which can reject what only the workspace knows — for example an id that isn't in this workspace.
- `update_workflow` and `publish_workflow` also return `warnings`: connector paths (a branch, Else, Success, a Failure) with no next step, where contacts leave the workflow, and Trigger Another Workflow steps whose target isn't published, which contacts never enter. They don't block the call. Show each one to the user and confirm the path is meant to end there.
- Each problem names the field and, usually, the tool that gives the right value. See [troubleshooting](references/troubleshooting.md).

## 7. Test

Testing runs the **last saved version for real** on whoever sends the test code.

Before calling `start_test_workflow`, tell the user:

- which channel the test uses (from `list_space_channels`)
- that they will get a `TESTnnnn` code to send, exactly, from a test contact on that channel
- what the steps will really do to that contact: messages sent, tags, fields and lifecycle changed, assignment and conversation status changed, comments added, HTTP requests sent, spreadsheet rows added, events sent to Meta or TikTok, other workflows triggered
- any sample trigger variable values the run will use

Call Ended workflows (`callEnded`) can only be tested on a WhatsApp Business (`whatsapp_business`) channel.

A test run has no real trigger event, so steps that read `$conversation.*`, `$clicktochat.*`, `$tiktokmessagingad.*`, `$call.*`, a Shortcut field or an Incoming Webhook variable see them empty unless you pass sample values in `variables`, keyed by the variable name without the `$` (e.g. `conversation.first_incoming_message`, or a Shortcut field's `variable`). Ask the user for these values when the steps branch on them. Only those triggers take variables; the tool refuses names the trigger does not offer and lists the valid ones.

When the user is done, call `end_test_workflow` with the code.

## 8. Publish, stop and delete

- `publish_workflow` makes the trigger live: from then on it enrolls every contact that matches. Only a draft or stopped workflow can be published. Show the user the final diagram first.
- `stop_workflow` removes **every contact currently in the workflow, immediately**, partway through its steps. Publishing again does not put them back. Tell the user this before stopping — including when stopping only to make an edit.
- Published workflows that start this one from a Trigger Another Workflow step skip it while it is stopped or deleted, with no error. `stop_workflow` and `delete_workflow` name them; tell the user.
- `delete_workflow` is permanent and refuses a published workflow; stop it first.

## 9. Run a workflow on one contact (Shortcuts)

`trigger_shortcut` is the only way to run a workflow on demand, and only for a **published** workflow whose trigger is `shortcut`. If the user wants to run any other workflow on a contact, explain this. Offer to build a Shortcut version as a new workflow, or a test run (section 7).

- **Build one**: `triggerPayload.name` is the label in the inbox's shortcut menu (1–255 characters). `description` is optional. Add up to 10 `fields`, each with a `name`, a `type` (`text`, `list`, `boolean`, `number`, `date`, `time`, `phone`, `email`, `url`) and a unique `variable` the steps can read as `{{$workflow.<variable>}}`. A `list` field needs 1–10 unique `values` of up to 20 characters. Set `required: true` on a field the shortcut can't run without. Publish it, or it won't appear in `list_shortcuts`.
- **Change its fields**: stop it, update it, publish it again.
- **Run one**:
  1. Get its `id` and `fields` from `list_shortcuts`.
  2. Resolve the contact with `search_contacts` and pass it as `id:<contactId>`.
  3. Ask the user for each field value; never invent one. Every `required` field must have a value, or `trigger_shortcut` refuses to run. A `list` field takes one of its `values`; a date is `YYYY-MM-DD` or `YYYY-MM-DD HH:mm`; a time is `HH:mm`. Optional fields left out are sent empty (`false` for a boolean).
  4. Tell the user which contact it runs on and what the steps will do — it runs the published version for real, not a test — then call `trigger_shortcut` with `fields` keyed by `variable`. Your client may show the user a form with these fields, pre-filled with your values; the shortcut then runs with whatever they submit.

## Show the workflow

Before the user chooses a template, and before they confirm an update or a publish, draw the workflow as a diagram from a fresh `get_workflow` (or `get_workflow_template`) result. Redraw after saving, because the server adds default connectors and names. How to draw it: [visualizing](references/visualizing.md).
