# Troubleshooting workflow tools

Errors come back as `Error: <message>`. Find the closest match below. A fix that needs the user means telling them exactly what to do and waiting — don't guess around it.

## "The workflow was not saved — fix these problems …"

`update_workflow` checked the whole graph and saved nothing. Every problem is listed; fix all of them and send the **full** payload again (start from `get_workflow` if you no longer have it). `get_workflow_template` lists the same problems as `needsInput`, and they are fixed the same way.

**Graph shape**

- `parentId "x" is not a step id` / `exactly one step must have parentId 1`: every step except the first needs an existing step id as its parent; the first one uses `1`.
- `has N child steps; only one step can follow it`: a regular step can lead to one step only. Use a Branch to split.
- `has branches, so "x" must use one of its connector ids as parentId`: a step after a question, assignment, Date & Time, branch or AI step hangs off one of its connector steps, not off the step itself.
- `a xConnector must have a x step as its parentId` / `connectorType must be one of …`: the connector sits under the wrong kind of step, or uses an outcome that step doesn't have.
- `needs 2–10 branches including Else`: a Branch needs at least one branch with conditions.
- `is part of a parentId cycle`: steps point at each other as parents. Loops are made with a `jumpTo` step, never with `parentId`.
- `jumpTo can only target a regular step`: point `stepId` at a box, not a connector or another jump.
- `id is used by more than one step` / `another step is already named …`: ids and names must be unique.
- `at most 100 steps`: the workflow is too big — split it and link the parts with a `triggerWorkflow` step.

**Values**

- `… must be a numeric tag id — look it up with search_tags`, `… a user id from list_space_users`, `… a team id from list_teams`, `… a lifecycle id from list_lifecycles`, `… a channel id from list_space_channels`, `… a workflow id from list_workflows`, `… conditions take tag ids …, not names`: a name was sent where an id belongs. Call the named tool and use the id.
- `variable name … is already saved by …`: two places save to the same workflow variable. Rename one, and update the text that reads it.
- `… events send no properties`: `Spam` and `DisqualifiedConversation` take none. Set the named properties to null.
- `… must be an ISO 4217 currency code`: use a code such as `USD`, or a variable.
- `contents can have at most 20 items` / `content_id is required once any of its fields is set`: trim the list, or give each filled item a `content_id`.
- `remove payload.properties.…` (or `event.…`, `datasets.…`, `payload.…`): the builder takes only the listed keys at every level, not just at the top.
- `ask the user which timezone this schedule uses`: ask, then set `timezone` to an IANA name or `"workspace"`.
- `must be between … and 7 days` (or 1 day for AI steps): durations are whole numbers with unit `seconds`, `minutes`, `hours` or `days`.

**Values from a lookup tool** — each message names the tool to call next:

- `botIntegrationId must be a Meta|TikTok connection id from list_ads_integrations` / `botIntegrationId N is not a Meta|TikTok connection`: use an id from that provider's list, not the other one's.
- `businessId … is not a business portfolio of connection …` / `datasets.<platform> … is not a <platform> account of business …`: call `list_meta_datasets` with that connection and copy the business and accounts as-is.
- `adAccount must be an ad account from list_meta_ad_accounts, copied as-is`
- `adAccounts must list ad accounts of that connection from list_ads_integrations`
- `ads must list the {adId, adName} of ads from list_ctc_ads` / `selected must put the {adId, adName} …`
- `adAccount … is not an ad account of connection …` / `adAccounts: … not an ad account of connection …`: copy the account from `list_meta_ad_accounts` (Meta) or `list_ads_integrations` (TikTok) for that same connection.
- `… is not an ad of …`: use an ad `list_ctc_ads` returns for that ad account. `… is a <destination> ad — put it under selected.<destination>`: file each TikTok ad under the `destination` it was listed with.
- `template "…" is not a template of channel …`: copy the template as-is from `list_message_templates` for the step's channel. `is a WhatsApp template, but channel … is a <source> channel` / `needs a WhatsApp channelId`: templates go only to a WhatsApp channel set on the step; send text instead on other channels.
- `template <component> has N variable(s)`: give each `{{n}}` a value in that component's `parameters`, as `{"type": "text", "text": "..."}`. Ask the user for the values.
- `knowledgeSourceIds … is not a knowledge source of this workspace` / `is still "<status>", not completed`: use a `completed` source from `search_knowledge_sources`.
- `category "…" is not a closing category`: use a category from `list_closing_notes`, or leave it empty.
- `workflowId is this workflow` / `workflowId … is not a workflow of this workspace` / `stepId "…" is not a step of workflow …`: pick another workflow from `list_workflows`, and a step id from its `get_workflow` (not a connector), or `null` to start from the beginning.
- `contactFieldValue for <type> field … must be …`: list fields take one of the options `list_custom_fields` shows, date fields `YYYY-MM-DD`, checkbox fields `true` or `false`. None of these take a variable.
- `… isn't a contact field here`: use one of the field keys the error lists or a custom field slug from `list_custom_fields`. Field keys are camelCase (`firstName`, `countryCode`, `lang`), unlike `{{$contact.firstname}}` placeholders.
- `businessId and at least one of datasets.facebook, instagram or whatsapp must come from list_meta_datasets`
- `sheetName and columns must come from get_google_sheet for this step`

**Only the user can fix these** — tell them what to do, then continue:

- `if none is listed, the user must connect Meta|TikTok in Settings > Integrations`: no connection exists for that provider.
- `Google Sheets isn't connected` / `the spreadsheet can only be picked in the respond.io workflow builder`: the user adds the Google Sheets step in the builder, connects Google, picks the spreadsheet and saves. Then re-read with `get_workflow` and continue with `get_google_sheet` (SKILL.md, section 5).
- `has no uploaded file — files can't be uploaded through this tool`: the user uploads the attachment on that step in the builder. An outside URL is refused too; keep the url `get_workflow` returns after the upload.
- `remove payload[i].messageTag`: Messenger message tags are deprecated; drop the field.

## Platform errors

Formatted `Platform API error (<status> <METHOD> <path>): <reason>`.

- **4xx with a reason** (for example on `publish_workflow`, `PUT /api/v2/workflows/status`): the platform refused for that reason. Fix what it names and try again; don't retry unchanged. If the reason says the workspace's plan lacks a feature — likely for templates marked `business: true` or with `meta` features — explain this to the user. Don't silently strip the steps that use it.
- **5xx**, `Platform API timeout` or `Platform API network error`: a platform-side problem. Retry once; if it persists, tell the user.
- **A permission error from `list_ads_integrations`, `list_meta_ad_accounts`, `list_ctc_ads` or `list_meta_datasets`**: these read manager-only settings, so the user's role can't read integrations. A workspace manager has to set up that trigger or step.

## Specific tool errors

- `Workflow <id> is published. Stop it with stop_workflow first.` (`delete_workflow`): stopping removes every contact in it — confirm with the user first.
- `update_workflow` refused because the workflow is published: stop, edit, then publish again, with the same warning.
- `Call Ended workflows can only be tested on a whatsapp_business channel.` (`start_test_workflow`): pick a WhatsApp Business channel from `list_space_channels`.
- `Channel <id> not found; use list_space_channels.`
- `Unknown template "<id>"; use list_workflow_templates.`: use an id from the current catalog. The catalog changes, so don't reuse ids from memory.
- `adAccountIds must hold exactly one Meta ad account id.` (`list_ctc_ads`): call it once per Meta ad account.
- `Workflow <id> has no saved Google Sheets step "<stepId>".` (`get_google_sheet`): the user hasn't added and saved the step in the builder yet, or the `stepId` is wrong. Re-read it from `get_workflow`.
- `No spreadsheet is chosen on this step …` (`get_google_sheet`): the user still has to connect Google and pick the spreadsheet on that step, then save.
- `Workflow <id> is not a published Shortcut workflow; use list_shortcuts.` (`trigger_shortcut`): it is a draft, stopped, or not Shortcut-triggered. Publish it (with the user's go-ahead) or pick an id from `list_shortcuts`.
- `Unknown shortcut fields: … Expected: …` (`trigger_shortcut`): key `fields` by the listed `variable`s, not the field names.
- `Missing required shortcut fields: …` (`trigger_shortcut`): ask the user for each listed field, then call again with it in `fields`. Don't fill one in yourself.
- A contact lookup error from `trigger_shortcut`: find the contact with `search_contacts` and pass `id:<contactId>`.

## Session and permission errors

- `Access token expired or was revoked. Refresh the access token and retry the call.`: refresh the token and retry the same call.
- `Missing scope '<scope>' for tool '<tool>'`: the user didn't grant that permission when connecting. They have to reconnect and approve it; retrying won't help.
- `Cancelled: the user declined to confirm …`: nothing changed. Don't retry unless the user asks again.
