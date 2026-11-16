# Workflow patterns

Each pattern is a complete `update_workflow` payload (`trigger`, `triggerPayload`, `steps`; the workflow `id` is added at call time). Every number and account id in them is made up — replace each with a real one from the lookup tool named beside it. Leave out what the server fills in: step `name`s, `type: "step"`, default connectors and default field values.

## Ask a question, tag on success

Asks new conversations for an email address and tags the contact once they answer with a valid one. The server adds the question's `Failure: Invalid` connector; nothing follows it here, so an invalid answer simply ends the run.

- `value`: tag ids from `search_tags`.

```json
{
  "trigger": "conversationOpened",
  "triggerPayload": { "oncePerContact": true },
  "steps": [
    {
      "id": "ask-email",
      "parentId": 1,
      "data": {
        "type": "askQuestion",
        "question": { "text": "What's the best email to reach you?", "type": "email" },
        "saveResponse": { "field": null, "variable": "email_answer", "tag": false }
      }
    },
    {
      "id": "ask-email-ok",
      "parentId": "ask-email",
      "data": { "type": "askQuestionConnector", "connectorType": "success" }
    },
    {
      "id": "tag-lead",
      "parentId": "ask-email-ok",
      "data": { "type": "updateContactTag", "action": "addTag", "value": [101] }
    }
  ]
}
```

## Business-hours routing

Inside business hours (the Date & Time step's `success` connector) the conversation goes to a team; outside them (`failure`) the contact gets an away message. The default schedule is Monday to Sunday, 09:00–17:00 — set `times` to change it, and always ask the user for the timezone.

- `teamId`: from `list_teams`.

```json
{
  "trigger": "conversationOpened",
  "steps": [
    {
      "id": "hours",
      "parentId": 1,
      "name": "Within business hours?",
      "data": { "type": "dateTime", "action": "businessHours", "timezone": "workspace" }
    },
    {
      "id": "hours-open",
      "parentId": "hours",
      "data": { "type": "dateTimeConnector", "connectorType": "success" }
    },
    {
      "id": "hours-closed",
      "parentId": "hours",
      "data": { "type": "dateTimeConnector", "connectorType": "failure" }
    },
    {
      "id": "assign-support",
      "parentId": "hours-open",
      "name": "Assign to Support",
      "data": {
        "type": "assignTo",
        "assignTo": "teamUser",
        "teamId": 7,
        "assignmentLogic": "RoundRobin"
      }
    },
    {
      "id": "away",
      "parentId": "hours-closed",
      "name": "Away message",
      "data": {
        "type": "sendMessage",
        "payload": [
          {
            "message": {
              "type": "text",
              "text": "Thanks for reaching out! We're closed right now and will reply when we're back."
            }
          }
        ]
      }
    }
  ]
}
```

## Branch on a tag, with Else

VIP-tagged contacts go to a named user; everyone else goes to a team. Branches are checked in order and the first match wins; the server adds the Else branch if it's missing, but naming it here makes the graph easier to read.

- `contactTag` condition `value`: tag ids from `search_tags`.
- `userId`: from `list_space_users`. `teamId`: from `list_teams`.

```json
{
  "trigger": "conversationOpened",
  "steps": [
    { "id": "route", "parentId": 1, "name": "VIP?", "data": { "type": "branch" } },
    {
      "id": "route-vip",
      "parentId": "route",
      "name": "VIP",
      "data": {
        "type": "branchConnector",
        "conditions": {
          "$and": [{ "category": "contactTag", "field": null, "operator": "hasAnyOf", "value": [205] }]
        }
      }
    },
    {
      "id": "route-else",
      "parentId": "route",
      "data": { "type": "branchConnector", "conditions": null }
    },
    {
      "id": "assign-manager",
      "parentId": "route-vip",
      "name": "Assign to account manager",
      "data": { "type": "assignTo", "assignTo": "specificUser", "userId": 31 }
    },
    {
      "id": "assign-team",
      "parentId": "route-else",
      "name": "Assign to Support",
      "data": {
        "type": "assignTo",
        "assignTo": "teamUser",
        "teamId": 7,
        "assignmentLogic": "LeastContacts"
      }
    }
  ]
}
```

## Handle a message that fails to send

A Send a Message step has no connectors by default. Adding one adds both `success` and `sendingFailure`, so the run can continue differently when delivery fails — here, leaving a comment for the team.

- `channelId`: from `list_space_channels`, or `null` for the contact's last interacted channel (first message only).

```json
{
  "trigger": "contactTagUpdated",
  "triggerPayload": { "action": "tagAdded", "tag": 101 },
  "steps": [
    {
      "id": "welcome",
      "parentId": 1,
      "name": "Welcome lead",
      "data": {
        "type": "sendMessage",
        "channelId": 5512,
        "payload": [{ "message": { "type": "text", "text": "Hi {{$contact.firstname}}, thanks for your interest!" } }]
      }
    },
    {
      "id": "welcome-sent",
      "parentId": "welcome",
      "data": { "type": "sendMessageConnector", "connectorType": "success" }
    },
    {
      "id": "welcome-failed",
      "parentId": "welcome",
      "data": { "type": "sendMessageConnector", "connectorType": "sendingFailure" }
    },
    {
      "id": "open",
      "parentId": "welcome-sent",
      "data": { "type": "openConversation" }
    },
    {
      "id": "note-failure",
      "parentId": "welcome-failed",
      "name": "Flag failed welcome",
      "data": { "type": "addComment", "comment": "The welcome message to this lead failed to send." }
    }
  ]
}
```

## Click-to-chat ads, then a Conversions API purchase

Runs for contacts who start a chat from selected Meta ads, asks for their order number, and reports a purchase back to Meta.

- `botIntegrationId`: a Meta connection from `list_ads_integrations`.
- `adAccount`: one entry from `list_meta_ad_accounts`, copied as returned.
- `ads`: `{ adId, adName }` of the chosen entries from `list_ctc_ads`.
- `businessId` and `datasets`: from `list_meta_datasets`, accounts copied as returned.

```json
{
  "trigger": "ctcAds",
  "triggerPayload": {
    "botIntegrationId": 12,
    "adAccount": { "id": "act_1001", "account_id": "1001", "name": "Harbor Ads" },
    "adsSelectionType": "selected",
    "ads": [{ "adId": "120200000000001", "adName": "Spring promo" }]
  },
  "steps": [
    {
      "id": "ask-order",
      "parentId": 1,
      "name": "Ask for order number",
      "data": {
        "type": "askQuestion",
        "question": { "text": "Thanks for clicking our ad! Reply with your order number to claim the offer.", "type": "text" },
        "saveResponse": { "field": null, "variable": "order_id", "tag": false }
      }
    },
    {
      "id": "ask-order-ok",
      "parentId": "ask-order",
      "data": { "type": "askQuestionConnector", "connectorType": "success" }
    },
    {
      "id": "report-purchase",
      "parentId": "ask-order-ok",
      "name": "Report purchase to Meta",
      "data": {
        "type": "sendConversionsApiEvent",
        "botIntegrationId": 12,
        "businessId": "2001",
        "datasets": { "facebook": { "id": "3001", "name": "Harbor Page", "datasetId": "4001" } },
        "event": { "name": "Purchase", "currency": "USD", "value": "49.90" }
      }
    }
  ]
}
```

## Shortcut with fields

An inbox shortcut agents run on a contact: it asks for a reason and details, leaves them as a comment, and assigns the conversation to Billing. Once published, `list_shortcuts` lists it and `trigger_shortcut` runs it with `fields: { "refund_reason": "...", "refund_details": "..." }`.

- `teamId`: from `list_teams`.

```json
{
  "trigger": "shortcut",
  "triggerPayload": {
    "name": "Refund Request",
    "description": "Log a refund request and hand the conversation to Billing.",
    "fields": [
      {
        "name": "Reason",
        "type": "list",
        "variable": "refund_reason",
        "required": true,
        "values": ["Damaged", "Late delivery", "Wrong item"]
      },
      { "name": "Details", "type": "text", "variable": "refund_details" }
    ]
  },
  "steps": [
    {
      "id": "log-refund",
      "parentId": 1,
      "name": "Log refund request",
      "data": {
        "type": "addComment",
        "comment": "Refund requested ({{$workflow.refund_reason}}): {{$workflow.refund_details}}"
      }
    },
    {
      "id": "assign-billing",
      "parentId": "log-refund",
      "name": "Assign to Billing",
      "data": {
        "type": "assignTo",
        "assignTo": "teamUser",
        "teamId": 9,
        "assignmentLogic": "RoundRobin"
      }
    }
  ]
}
```
