# SBT App — Channel & Bot Architecture Report

## 1. Core Hierarchy

The entire system is built on a three-tier ownership chain:

```
Account
  └── UsecaseModel (the "bot brain")
        └── Campaign (the operational runtime of that bot)
```

- **Account** — the top-level tenant. All resources belong to an account.
- **UsecaseModel** (`usecasemodel_v2`) — defines the bot: system prompt, channel type, direction, language, voices, and channel-specific config (e.g. `wan_id` for WhatsApp). Think of it as the bot definition.
- **Campaign** — the live instance of a UsecaseModel. Multiple campaigns can run under the same UsecaseModel. Campaigns carry runtime config: server assignment, status, retry logic, contact lists (for outbound), etc.

---

## 2. Channel Types

There are four channel types, stored in the `channel_types` table and referenced by `UsecaseModel.channel_type_id`:

| ID | Name | Slug | Purpose |
|---|---|---|---|
| 1 | Telephony Agent | `telephony_agent` | Voice calls (inbound/outbound) |
| 2 | Voice Assistant | `voice_assistant` | IVR / voice-only assistant (inbound only) |
| 3 | Text Bot | `text_assistant` | Web chatbot widget (inbound only) |
| 4 | WhatsApp | `whatsapp` | WhatsApp messaging (inbound + outbound) |

> Facebook and Instagram do **not** have their own channel types. They reuse an existing channel through a connector pattern (see Section 5).

---

## 3. Direction Rules

The `direction` column on `UsecaseModel` is either `inbound` or `outbound`. Allowed directions are constrained by channel type:

| Channel Type | Inbound | Outbound | Notes |
|---|---|---|---|
| Text Bot | ✅ | ❌ | Widget-initiated conversations only |
| Voice Assistant | ✅ | ❌ | IVR / passive listener only |
| Telephony Agent | ✅ | ✅ | Full call center support |
| WhatsApp | ✅ | ✅ | Inbound: webhook bot. Outbound: template blasts |

**Additional constraint for WhatsApp inbound:** A given WhatsApp Account Number (`wan_id`) can only be linked to **one** inbound UsecaseModel at a time. This is a routing guarantee — without it, an incoming message on a number would be ambiguous between two bots. Multiple outbound UsecaseModels *can* share the same `wan_id` since outbound is push-only and doesn't need to route incoming messages.

---

## 4. Integration Architectures — Two Philosophies

Across all channels, two distinct integration patterns are used. Understanding these is key to understanding how routing works.

### 4.1 Tight Coupling — Channel config lives on UsecaseModel

Used by: **WhatsApp**

The external account/number (`wan_id`) is stored directly on the UsecaseModel. The bot *knows* which number it owns.

```
UsecaseModel
  ├── channel_type_id = 4 (WhatsApp)
  ├── direction = 'inbound' or 'outbound'
  └── wan_id → whatsapp_account_numbers.id
```

Routing path for an **inbound** WhatsApp message:
```
Incoming message (from phone number X)
  → lookup whatsapp_account_numbers where phone_number = X
  → find UsecaseModel where wan_id = that WAN, direction = 'inbound'
  → get active Campaign under that UsecaseModel
  → route message to bot
```

### 4.2 Loose Coupling — Connector record holds campaign_id

Used by: **Text Bot (Widget)**, **Facebook**, **Instagram**

The UsecaseModel and Campaign have no direct awareness of the external integration. Instead, a separate connector record is created that holds a `campaign_id` pointer. The bot is attached *after* the connector exists.

```
ConnectorRecord (widget_client / facebook_page / instagram_account)
  ├── account_id
  ├── campaign_id → campaigns.id
  └── [source-specific fields: page_id, ig_user_id, api_key, etc.]
```

Routing path for an incoming message via this pattern:
```
Incoming message (with source identifier: widget API key / FB page_id / IG user_id)
  → lookup connector record by source identifier
  → get campaign_id from connector
  → get UsecaseModel via Campaign.usecasemodel_id
  → route message to bot
```

---

## 5. Channel-by-Channel Flows

### 5.1 Text Bot (Web Widget)

**Direction:** Inbound only
**Integration pattern:** Loose coupling via `widget_clients`

**Setup flow:**
1. Create a UsecaseModel with channel_type = Text Bot, direction = inbound.
2. Create one or more Campaigns under it.
3. Go to **Widget Clients** section → create a widget client.
4. During widget client creation, select the UsecaseModel and a Campaign.
5. System generates an HTML shortcode (embed script/iframe).
6. Client pastes the shortcode into their website.

**Runtime flow:**
```
User opens website with embedded widget
  → Widget loads, identifies itself with API key
  → Backend looks up widget_client by api_key
  → Gets campaign_id from widget_client
  → Gets UsecaseModel via campaign
  → Chat session begins under that Campaign
```

**Key point:** The UsecaseModel and Campaign are completely unaware of the widget. The widget knows about the campaign, not the other way around.

---

### 5.2 WhatsApp — Inbound

**Direction:** Inbound
**Integration pattern:** Tight coupling — `wan_id` on UsecaseModel

**Setup flow:**
1. WhatsApp Account Number (WAN) is provisioned and stored in `whatsapp_account_numbers`.
2. Create a UsecaseModel with channel_type = WhatsApp, direction = inbound, select a WAN → `wan_id` is stored on the UsecaseModel.
3. A given WAN can only be used by **one** inbound UsecaseModel (enforced to guarantee routing).
4. Create Campaigns under this UsecaseModel.

**Runtime flow:**
```
User sends WhatsApp message to number X
  → Meta webhook fires to our endpoint
  → Lookup WAN by phone number X
  → Find inbound UsecaseModel with that wan_id
  → Pick active Campaign
  → Continue/start conversation
```

---

### 5.3 WhatsApp — Outbound

**Direction:** Outbound
**Integration pattern:** Tight coupling — `wan_id` on UsecaseModel

**Setup flow:**
1. Create a UsecaseModel with channel_type = WhatsApp, direction = outbound, select a WAN.
2. Multiple outbound UsecaseModels can share the same WAN (same number, different campaigns/templates — this is fine because outbound is push-only).
3. Create a Campaign under the UsecaseModel, load a contact list.

**Runtime flow:**
```
Campaign triggers outbound blast
  → For each contact, send WhatsApp template via the WAN number
  → If the contact replies, message is routed to the *inbound* UsecaseModel for that number
  → (Outbound UsecaseModel is not involved in the reply routing)
```

---

### 5.4 Telephony Agent — Inbound

**Direction:** Inbound
**Integration pattern:** DID numbers assigned to Campaigns (`campaign_dids`)

**Setup flow:**
1. Create a UsecaseModel with channel_type = Telephony, direction = inbound.
2. Create a Campaign, assign DID phone numbers to it via `campaign_dids`.

**Runtime flow:**
```
Caller dials a DID number
  → Lookup campaign_dids by number
  → Get Campaign → get UsecaseModel
  → Start voice call session
```

---

### 5.5 Telephony Agent — Outbound

**Direction:** Outbound
**Integration pattern:** Campaign manages a contact/phone list

**Setup flow:**
1. Create a UsecaseModel with channel_type = Telephony, direction = outbound.
2. Create a Campaign, upload/assign phone contacts (`campaign_phones`).

**Runtime flow:**
```
Campaign scheduler triggers
  → Dials each contact in campaign_phones
  → Runs voice conversation via UsecaseModel
  → Logs call result to campaign_phone_calls
```

---

### 5.6 Voice Assistant — Inbound

**Direction:** Inbound only
**Integration pattern:** Similar to Telephony inbound (DID-based)

A simpler voice channel — IVR-style. Follows the same DID-based routing as Telephony inbound but is configured for assistant-style interactions rather than agent-style.

---

### 5.7 Facebook Messenger

**Direction:** Inbound (webhook-driven)
**Integration pattern:** Loose coupling via `facebook_pages`
**Channel type used:** None — reuses an existing channel type's UsecaseModel

**Setup flow:**
1. User goes to Facebook integration section → clicks **Connect Page**.
2. OAuth flow runs, Meta page access token is obtained.
3. A `facebook_pages` record is created: `page_id`, `page_name`, `page_access_token`, `account_id`.
4. Webhook subscription is registered with Meta for that page.
5. User then links a Campaign to the page: `facebook_pages.campaign_id = campaign.id`.

**Runtime flow:**
```
User sends message on Facebook Page
  → Meta webhook fires with page_id
  → Lookup facebook_pages by page_id
  → Get campaign_id from facebook_pages
  → Get UsecaseModel via Campaign
  → Route message to bot
```

**Key point:** The UsecaseModel and Campaign don't know they're serving a Facebook page. The Facebook page knows the campaign.

---

### 5.8 Instagram DM

**Direction:** Inbound (webhook-driven)
**Integration pattern:** Loose coupling via `instagram_accounts`
**Channel type used:** None — reuses an existing channel type's UsecaseModel

**Setup flow:**
1. User goes to Instagram integration section → clicks **Connect Account**.
2. OAuth flow runs, IG access token is obtained.
3. An `instagram_accounts` record is created: `ig_user_id`, `ig_username`, `account_type` (BUSINESS or MEDIA_CREATOR), `access_token`, `account_id`.
4. Webhook subscription is registered with Meta for that IG account.
5. User links a Campaign to the IG account: `instagram_accounts.campaign_id = campaign.id`.

**Runtime flow:**
```
User sends Instagram DM
  → Meta webhook fires with ig_user_id
  → Lookup instagram_accounts by ig_user_id
  → Get campaign_id from instagram_accounts
  → Get UsecaseModel via Campaign
  → Route message to bot
```

---

## 6. Routing Summary Table

| Channel | Incoming Identifier | Lookup Table | Gets campaign via |
|---|---|---|---|
| Text Bot (Widget) | API key | `widget_clients` | `widget_clients.campaign_id` |
| WhatsApp Inbound | Phone number | `whatsapp_account_numbers` → `usecasemodel_v2` | active campaign of that UsecaseModel |
| Telephony Inbound | DID number | `campaign_dids` | `campaign_dids.campaign_id` |
| Facebook Messenger | Page ID | `facebook_pages` | `facebook_pages.campaign_id` |
| Instagram DM | IG User ID | `instagram_accounts` | `instagram_accounts.campaign_id` |

---

## 7. Connector Records vs Channel Types — The Key Distinction

| Integration | Has its own ChannelType? | Config lives on UsecaseModel? | Connector record? |
|---|---|---|---|
| Text Bot | ✅ Yes (Text Bot) | No wan-like field | `widget_clients` |
| WhatsApp | ✅ Yes (WhatsApp) | ✅ Yes (`wan_id`) | No separate connector |
| Telephony | ✅ Yes (Telephony Agent) | No connector field | `campaign_dids` |
| Voice Assistant | ✅ Yes (Voice Assistant) | No connector field | `campaign_dids` |
| Facebook | ❌ No | No | `facebook_pages` |
| Instagram | ❌ No | No | `instagram_accounts` |

Facebook and Instagram deliberately avoid introducing new channel types. This keeps UsecaseModel clean and avoids proliferating channel-specific bot config. The trade-off is that the bot has no native awareness of which social platform it's serving — that context must be inferred at runtime from the incoming webhook.

---

## 8. Architectural Observations

1. **Two routing philosophies coexist.** WhatsApp is tight-coupled (channel config on UsecaseModel). Social (FB/IG) and Widget are loose-coupled (campaign_id on the connector). Both work, but they answer "which bot handles this message?" differently — one by looking at UsecaseModel, one by looking at the connector.

2. **The inbound WAN uniqueness constraint is a routing guarantee, not just a business rule.** Without it, an incoming WhatsApp message would be undecidable between two bots on the same number.

3. **Loose coupling for social is the right long-term call.** By not baking FB/IG into UsecaseModel, the same bot definition can serve multiple channels. The UsecaseModel stays channel-agnostic.

4. **Channel-specific behavior is a future friction point.** Since Facebook and Instagram share a UsecaseModel with no channel awareness, any channel-specific logic (e.g. Instagram supports reactions, different message formats, IG-only features) must be handled at the routing/handler layer rather than at the bot config layer.

5. **Campaign is the universal join point.** Regardless of channel, the campaign is always the bridge between the external world and the bot. The connector pattern (widget, FB, IG) and the DID pattern (telephony) both ultimately resolve to a `campaign_id`, from which the UsecaseModel is derived.
