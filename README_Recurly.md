# Recurly Paid Invoice → Classify → Slack

Automatically receive Recurly paid invoice webhooks, classify the transaction type, and post a celebratory Slack notification — with a random GIF. 🎉

Two versions are included:

| File | Environment | Status |
|------|-------------|--------|
| `QA_Recurly_Paid_Invoice___Classify___Slack_PUBLIC.json` | QA / Testing | Inactive by default |
| `Prod_Recurly_Paid_Invoice___Classify___Slack_PUBLIC.json` | Production | Active by default |

Both workflows are functionally identical. Use the QA version to test with your sandbox Recurly account before going live.

---

## What It Does

```
Recurly fires a webhook
        ↓
Webhook node receives the paid_charge_invoice_notification
        ↓
Extract Fields — parses account, invoice, subscription IDs
        ↓
Recurly API — fetches full invoice details
        ↓
Recurly API — fetches subscription details (plan code, quantity, origin)
        ↓
Classify Customer — determines transaction type:
   🆕 NEW_CUSTOMER        origin = purchase
   🔄 RENEWAL             origin = renewal
   ⬆️ UPGRADE_TO_ANNUAL   immediate_change + credit from prior monthly plan
   📈 EXPANSION           immediate_change + quantity increased
        ↓
Format Slack Message — builds a rich Slack block with customer info + random GIF
        ↓
IF nodes — route based on customer type + plan code
        ↓
Send a message — posts to your Slack channel
```

---

## Prerequisites

You need accounts and credentials for:

- **[Recurly](https://recurly.com)** — billing platform (Private API Key)
- **[Slack](https://slack.com)** — messaging (Bot Token with `chat:write` scope)
- **[n8n](https://n8n.io)** — self-hosted or n8n Cloud

---

## Setup Instructions

### 1. Import the Workflow

In your n8n instance:
1. Go to **Workflows** → **Import from file**
2. Upload the JSON file for your target environment (QA or Prod)

### 2. Configure Credentials

You will need to create **3 credentials** in n8n (`Settings → Credentials → Add`):

#### A. Recurly Webhook — Basic Auth (HTTP Basic Auth)
Secures the incoming webhook endpoint so only Recurly can call it.

| Field | Value |
|-------|-------|
| User | Any username you choose (e.g. `recurly`) |
| Password | Any strong password you choose |

> ⚠️ You'll use these same values when configuring the webhook URL in the Recurly dashboard.

#### B. Recurly Private API Key (HTTP Basic Auth)
Used to call the Recurly REST API to fetch invoice and subscription details.

| Field | Value |
|-------|-------|
| User | Your Recurly **Private API Key** |
| Password | *(leave blank)* |

> Find your API key in Recurly → **Integrations → API Credentials**.

#### C. Slack Bot Token (Slack API)
Used to post messages to a Slack channel.

| Field | Value |
|-------|-------|
| Access Token | Your Slack Bot OAuth Token (`xoxb-...`) |

> In Slack: **api.slack.com/apps** → Your App → **OAuth & Permissions** → copy `Bot User OAuth Token`.
> Required scope: `chat:write`

### 3. Assign Credentials to Nodes

After importing, open each of these nodes and select the matching credential you just created:

| Node | Credential to assign |
|------|----------------------|
| **Webhook - Recurly** | Recurly Webhook Basic Auth |
| **Recurly API - Get Invoice details** | Recurly Private API Key |
| **Recurly API - Get Subscription** | Recurly Private API Key |
| **Send a message** (both) | Slack Bot Token |

### 4. Set Your Slack Channel

Open each **Send a message** node and set the **Channel** field to your target Slack channel (e.g. `#revenue` or `#sales`).

### 5. Register the Webhook URL in Recurly

1. In n8n, open the **Webhook - Recurly** node and copy the **Production URL** (or Test URL for QA)
2. In Recurly, go to **Integrations → Webhooks → Add Endpoint**
3. Paste the URL, set **HTTP Auth** with the username/password from Step 2A
4. Select the event: **Paid charge invoice** (`paid_charge_invoice_notification`)

### 6. Activate the Workflow

Toggle the workflow to **Active** in n8n. It's now live.

---

## Classification Logic

The **Classify Customer** node determines transaction type based on two Recurly fields:

| Type | `origin` value | Additional condition |
|------|---------------|----------------------|
| `NEW_CUSTOMER` | `purchase` | — |
| `RENEWAL` | `renewal` | — |
| `UPGRADE_TO_ANNUAL` | `immediate_change` | Credit payments exist from a prior monthly plan |
| `EXPANSION` | `immediate_change` | Seat quantity increased (no upgrade credit) |
| `UNKNOWN` | anything else | Falls through to a no-op node |

The IF nodes further route by `plan_code` (e.g. `true_platform_teams_monthly`, `true_platform_teams_annual`) — update these to match your own Recurly plan codes.

---

## Customisation Tips

- **Plan codes** — the IF nodes filter on specific plan codes. Update `rightValue` in each IF node to match your Recurly plan codes.
- **Slack message format** — edit the **Format Slack Message** code node to change the message layout, fields shown, or GIF pool.
- **Add more customer types** — extend the `if/else` block in **Classify Customer** and add a new IF node + Slack node for the new type.
- **Multiple channels** — duplicate the **Send a message** node and route different types to different channels.

---

## Workflow Nodes Overview

| # | Node | Type | Purpose |
|---|------|------|---------|
| 0 | Webhook - Recurly | Webhook | Entry point — receives Recurly `paid_charge_invoice_notification` |
| 1 | Recurly API - Get Invoice details | HTTP Request | Fetches full invoice from Recurly API |
| 2 | Extract Fields | Code (JS) | Parses notification payload into clean fields |
| 3 | Recurly API - Get Subscription | HTTP Request | Fetches subscription details (plan, quantity) |
| 4 | Classify Customer | Code (JS) | Determines transaction type |
| 5 | Format Slack Message | Code (JS) | Builds Slack block kit message + random GIF |
| 6 | If Renewal | IF | Routes RENEWAL transactions |
| 7 | If New | IF | Routes NEW_CUSTOMER transactions |
| 8 | If Upgrade | IF | Routes UPGRADE_TO_ANNUAL transactions |
| 9 | If Expansion | IF | Routes EXPANSION transactions |
| 10 | Format Slack Message (alt) | Code (JS) | Secondary formatter for alternate route |
| 11–13 | Send a message (×2) + No-op | Slack / NoOp | Sends to Slack or silently drops UNKNOWN types |

---

## QA vs Production Differences

| | QA | Production |
|--|----|------------|
| Default status | Inactive | Active |
| Webhook URL slug | `test-in-QA` | `recurly-paid-invoice` |
| Recurly credential | QA/sandbox API key | Production API key |
| Recommended use | Test with Recurly sandbox | Live customer transactions |

Use the QA workflow to trigger test webhooks from the Recurly sandbox and verify Slack messages look correct before switching to Production.

---

## Troubleshooting

**Webhook returns 401 Unauthorized**
→ The Basic Auth credentials on the Webhook node don't match what Recurly is sending. Double-check Step 2A and the Recurly webhook configuration.

**`Could not find notification data` error in Extract Fields**
→ The webhook payload structure is unexpected. Check the raw webhook body in the n8n execution log and confirm Recurly is sending `paid_charge_invoice_notification`.

**Customer type is `UNKNOWN`**
→ The `origin` value from Recurly doesn't match any of the expected values. Log `webhookData.origin` in the Classify Customer node to see what value is arriving.

**Slack message not posting**
→ Verify the bot is invited to the target channel (`/invite @YourBotName`) and that the `chat:write` scope is granted.

---

## License

MIT — free to use, modify, and share.
