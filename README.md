# n8n-revops-sentinel-monitor

An automated **n8n health-monitoring watchdog for HubSpot CRM**. Every day it audits your pipeline and inbound lead flow, computes a single 0-100 database health score, and posts one consolidated executive alert to Slack.

Built with 100% native n8n nodes. No community nodes, no external services beyond HubSpot and Slack.

---

## Table of Contents

1. [Overview](#overview)
2. [Workflow Architecture](#workflow-architecture)
3. [Health Score Model](#health-score-model)
4. [Repository Structure](#repository-structure)
5. [Prerequisites](#prerequisites)
6. [Setup Guide](#setup-guide)
7. [Configuration Reference](#configuration-reference)
8. [Customization](#customization)
9. [Troubleshooting](#troubleshooting)
10. [Safety & Responsibilities](#safety--responsibilities)
11. [License](#license)

---

## Overview

CRM decay is silent. Deals stop moving, inbound leads sit without an owner, and nobody notices until pipeline reviews go wrong. RevOps Sentinel watches for the two most expensive failure modes and reports them before they compound:

| Check | What it detects | HubSpot object |
|---|---|---|
| **Stagnant pipeline deals** | Open deals with no logged activity for more than 14 days (or never touched and older than 14 days) | Deals |
| **Unassigned inbound leads** | Recent leads in inbound lifecycle stages with no contact owner | Contacts |
| **Database health score** | Weighted 0-100 score derived from both ratios | Computed |

The workflow is **read-only**. It uses HubSpot's CRM search endpoints (which are `POST` requests but do not modify data) and never writes back to your CRM.

---

## Workflow Architecture

```
Daily Schedule Trigger ─┐
                        ├─► Build Audit Config ─┬─► Query Stagnant Deals ───────┐
Manual Test Trigger ────┘   (Code: dates,       ├─► Query Open Deals Total ─────┤
                             filters, weights)  ├─► Query Unassigned Leads ─────┼─► Sync Parallel Queries
                                                └─► Query Inbound Leads Total ──┘        (Merge, 4 inputs)
                                                                                              │
                                                                                              ▼
                                                                                   Score & Aggregate
                                                                                   (Code: health score,
                                                                                    Slack Block Kit)
                                                                                              │
                                                                                              ▼
                                                                                      Should Alert?
                                                                                      │          │
                                                                                    true        false
                                                                                      ▼          ▼
                                                                    Post Executive Alert   No Alert Needed
                                                                          to Slack
```

### Nodes

| Node | Type | Purpose |
|---|---|---|
| **Daily Schedule Trigger** | Schedule Trigger | Fires every day at 08:00 (workflow timezone, default UTC). |
| **Manual Test Trigger** | Manual Trigger | Lets you run the audit on demand while testing. |
| **Build Audit Config** | Code | Single place for thresholds, weights, and lookback windows. Computes the date cutoffs and builds all four HubSpot search request bodies. |
| **Query Stagnant Deals** | HTTP Request (HubSpot credential) | Deals search: `hs_is_closed EQ false` AND `notes_last_updated LT cutoff`, OR never-touched deals with `createdate LT cutoff` (uses `NOT_HAS_PROPERTY`). |
| **Query Open Deals Total** | HTTP Request (HubSpot credential) | Counts all open deals (denominator). Uses `limit: 1` and reads `total`. |
| **Query Unassigned Leads** | HTTP Request (HubSpot credential) | Contacts search: `hubspot_owner_id NOT_HAS_PROPERTY` AND `createdate GTE lookback` AND `lifecyclestage IN [...]`. |
| **Query Inbound Leads Total** | HTTP Request (HubSpot credential) | Counts all inbound leads in the window (denominator). |
| **Sync Parallel Queries** | Merge | Waits for all four parallel branches to finish before scoring. |
| **Score & Aggregate** | Code | Computes ratios, penalties, the final score and status, and assembles the Slack Block Kit payload. |
| **Should Alert?** | IF | Posts always, or only when status is not Healthy (controlled by `notifyAlways`). |
| **Post Executive Alert to Slack** | Slack | Sends the Block Kit executive summary. |
| **No Alert Needed** | No Operation | Terminates the quiet branch cleanly. |

### Why HTTP Request nodes for HubSpot?

HubSpot's CRM Search API supports the full operator set (`LT`, `GTE`, `IN`, `NOT_HAS_PROPERTY`, multi-group OR logic) and returns an exact `total` count in a single call, so counts stay accurate even when thousands of records match. The HTTP Request node is a native n8n node and authenticates through n8n's built-in **HubSpot App Token** credential, so no API keys are hard-coded.

---

## Health Score Model

```
stagnantRatio   = stagnantDeals   / openDeals
unassignedRatio = unassignedLeads / inboundLeads (in lookback window)

dealPenalty = stagnantRatio   × 100 × weights.deals   (default 0.6)
leadPenalty = unassignedRatio × 100 × weights.leads   (default 0.4)

score = round(100 − dealPenalty − leadPenalty)   // clamped to 0-100
```

| Score | Status | Indicator |
|---|---|---|
| 85-100 | Healthy | 🟢 |
| 70-84 | Needs Attention | 🟡 |
| 0-69 | Critical | 🔴 |

Example: 12 of 40 open deals are stagnant (30%) and 9 of 60 inbound leads are unassigned (15%) → penalties of 18 and 6 → score **76**, Needs Attention.

---

## Repository Structure

```
n8n-revops-sentinel-monitor/
├── workflows/
│   └── n8n-revops-sentinel-workflow.json   # importable n8n workflow
├── README.md
└── LICENSE
```

---

## Prerequisites

- An **n8n** instance (cloud or self-hosted). Version **1.49 or newer** is recommended (required for the 4-input Merge node).
- A **HubSpot** account with permission to create a Private App.
- A **Slack** workspace where you can add a bot to a channel.

---

## Setup Guide

### 1. Create a HubSpot Private App

1. In HubSpot go to **Settings → Integrations → Private Apps → Create private app**.
2. Under **Scopes**, grant read-only access:
   - `crm.objects.deals.read`
   - `crm.objects.contacts.read`
3. Create the app and copy the **access token**.

### 2. Create a Slack bot

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From scratch**.
2. Under **OAuth & Permissions**, add the bot scope `chat:write` (add `chat:write.public` if you want to post to channels the bot has not joined).
3. **Install to Workspace** and copy the **Bot User OAuth Token** (`xoxb-...`).
4. Invite the bot to your alert channel: `/invite @YourBotName`.
5. Copy the **channel ID** (right-click the channel → *View channel details* → bottom of the About tab, format `C0123456789`).

### 3. Import the workflow

1. In n8n choose **Workflows → Import from File**.
2. Select `workflows/n8n-revops-sentinel-workflow.json`.

### 4. Add credentials

1. Open any of the four **Query ...** nodes → **Credential** → **Create new** → **HubSpot App Token** and paste your token. Select the same credential on the other three query nodes.
2. Open **Post Executive Alert to Slack** → create a **Slack API** credential with your bot token.
3. In the same Slack node, set the **Channel** to your channel ID.

### 5. Tune the configuration

Open **Build Audit Config** and edit the `CONFIG` block at the top (see the [reference](#configuration-reference)). Optionally set `hubspotPortalId` to turn record names in Slack into clickable HubSpot links.

### 6. Test

1. Click **Execute workflow** from the **Manual Test Trigger**.
2. Inspect the output of **Score & Aggregate** and confirm the message arrived in Slack.

### 7. Activate

1. In **Workflow Settings** confirm the **Timezone** (default `UTC`) so the 08:00 run lands at the hour you want.
2. Toggle the workflow to **Active**.

---

## Configuration Reference

All settings live in the `CONFIG` object inside the **Build Audit Config** node.

| Key | Default | Description |
|---|---|---|
| `stagnantDays` | `14` | Open deals with no activity for longer than this are flagged. |
| `leadLookbackDays` | `30` | Only contacts created within this window are audited for missing owners. |
| `inboundLifecycleStages` | `['subscriber','lead','marketingqualifiedlead']` | Internal lifecycle stage values treated as inbound leads. |
| `weights.deals` / `weights.leads` | `0.6` / `0.4` | Penalty weights. Keep the sum at `1.0`. |
| `thresholds.healthy` | `85` | Score at or above this is Healthy. |
| `thresholds.warning` | `70` | Score at or above this (and below healthy) is Needs Attention. |
| `notifyAlways` | `true` | `false` = only post to Slack when status is not Healthy. |
| `hubspotPortalId` | `''` | Your HubSpot Hub ID. Enables clickable record links. |
| `topN` | `5` | Records listed per section in the Slack message. |

---

## Customization

- **Different inbound definition:** edit the `lifecyclestage IN` filter, or swap it for `hs_analytics_source`, `hs_lead_status`, or a custom property in the `unassignedLeads` and `inboundLeads` query objects.
- **Pipeline-specific audits:** add `{ propertyName: 'pipeline', operator: 'EQ', value: '<pipeline_id>' }` to the deal filters.
- **Different activity definition:** replace `notes_last_updated` with another date property such as `hs_lastmodifieddate` or `notes_last_contacted`.
- **Extra checks:** duplicate a query node, add a request body in **Build Audit Config**, connect it to a free **Sync Parallel Queries** input (increase `numberInputs`), and fold it into the score in **Score & Aggregate**.
- **Other channels:** replace the Slack node with Microsoft Teams, Email, or Telegram nodes. The output of **Score & Aggregate** includes plain `slackText` and structured `metrics`.
- **Failure alerting:** create a small separate workflow starting with an **Error Trigger** node and set it as the *Error Workflow* in this workflow's settings, so a failed audit never goes unnoticed.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `401` / `403` from HubSpot | Token missing, wrong credential selected, or the Private App lacks the read scopes above. |
| `429` from HubSpot | Search API rate limit. The nodes already retry 3 times; avoid running other heavy search workflows at the same time. |
| `channel_not_found` / `not_in_channel` | Invite the bot to the channel, or use the channel ID rather than the name. |
| Counts look too low or zero | Check that your deal and contact property internal names match the filters (custom setups sometimes differ). |
| Stagnant deal list is capped | Each list query returns at most 100 records for display. The **counts and score are exact** because they use HubSpot's `total`. |
| Deal stage shows as an ID | HubSpot returns internal stage IDs. Map them to labels in **Score & Aggregate** if you want names in Slack. |
| Slack node shows a version warning | Re-select the Slack operation in the node panel; n8n will upgrade the node parameters for your instance version. |

---

## Safety & Responsibilities

- This workflow only **reads** from HubSpot and **posts** to Slack. Review every node and filter before activating it against production data.
- Credentials (HubSpot token, Slack token) must be stored in n8n's credential manager. Never commit tokens to your repository.
- Test with the Manual Test Trigger and a non-production Slack channel first.
- Slack messages may contain contact names, company names, and deal values. Post only to channels with an appropriate audience.

**Disclaimer:** If anyone wants to try it then they should try with their own risk because dhrobinsystems is not liable for anything.

---

## License

Released under the [MIT License](LICENSE). Copyright (c) 2026 DH Robin Systems.
