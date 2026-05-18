# Software Requirements Specification

**Project:** Meta — Facebook Messenger Auto-Reply
**Stack:** Laravel 13, PHP 8.3, PostgreSQL

---

## 1. Purpose

A web app that lets a user connect their Facebook Page(s), receives incoming Messenger messages via Meta webhooks, and sends an automatic reply back to the sender.

## 2. Scope

**In scope**
- User registration & login (email/password)
- Connect / disconnect Facebook Pages per user via Meta OAuth
- Receive Messenger messages over webhook
- Send auto-reply to incoming messages
- Dashboard listing connected Pages

**Out of scope (this phase)**
- AI-generated replies
- Conversation history & threading
- Multi-tenant teams

---

## 3. External Dependencies — Meta Developer Account

A Meta App must be created at https://developers.facebook.com/apps. The following are **required** to run the system.

### 3.1 Credentials

| Item | Source | Stored in `.env` as |
|---|---|---|
| **App ID** | Meta App → Settings → Basic | `FACEBOOK_APP_ID` |
| **App Secret** | Meta App → Settings → Basic | `FACEBOOK_APP_SECRET` |

Both are used for:
- Exchanging the OAuth `code` for an access token
- Verifying webhook payload signatures (`X-Hub-Signature-256`)
- Making outbound Graph API calls

### 3.2 OAuth Scopes

The Facebook Login flow must request these permissions:

| Scope | Why it's needed |
|---|---|
| `pages_show_list` | List the Pages the user manages |
| `pages_messaging` | Send replies on behalf of the Page |
| `pages_manage_metadata` | Subscribe the Page to our webhook |

### 3.3 Webhook Configuration

In **Meta App → Messenger → Settings → Webhooks**:

| Field | Value |
|---|---|
| Callback URL | `${APP_URL}/webhook/facebook` |
| Verify Token | Any random string — must match `FACEBOOK_WEBHOOK_VERIFY_TOKEN` in `.env` |
| Subscription fields | `messages` |
| Pages | Add each Page that should receive auto-replies |

### 3.4 Public Compliance URLs

Meta App Review requires both of these to be publicly reachable (no auth, no login wall):

| Link | Route |
|---|---|
| **Privacy Policy** | `${APP_URL}/privacy` |
| **Terms of Service** | `${APP_URL}/terms` |

Both URLs must be entered in **Meta App → Settings → Basic**.

---

## 4. Functional Requirements

### 4.1 Auth
- User can register, log in, and edit their profile.

### 4.2 Connect Facebook Page
- Authenticated user clicks **Connect Facebook Page** → redirected to Meta OAuth with required scopes.
- On callback, server exchanges `code` → short-lived user token → fetches managed Pages from `GET /me/accounts`.
- Each Page is upserted to the database with its long-lived Page access token.
- Each Page is subscribed to the app's webhook (`subscribed_fields=messages`).

### 4.3 Webhook Verification
- `GET /webhook/facebook` returns `hub.challenge` only when `hub.verify_token` matches the configured verify token.

### 4.4 Webhook Message Handling
- `POST /webhook/facebook` receives events from Meta.
- For each entry, identify the receiving Page → look up its owner.
- For each incoming text message, pick a reply and send via the Messenger Send API.
- The endpoint must return `200` within Meta's retry window.

### 4.5 Disconnect
- User can disconnect any of their connected Pages from the dashboard.
- The Page record is deleted locally. (Token is not revoked on Meta's side; reconnecting is supported.)

---

## 5. Non-Functional Requirements

- **Security**
  - Webhook POSTs are verified via HMAC SHA-256 (`X-Hub-Signature-256`) using `FACEBOOK_APP_SECRET`.
  - Page access tokens are stored encrypted at rest.
  - HTTPS enforced (`URL::forceScheme('https')`).
- **Reliability**
  - The webhook endpoint responds in <1s; heavy work runs through the queue worker.
- **Privacy**
  - Logs do not contain raw message content or access tokens.

---

## 6. `.env` Configuration Reference

```env
APP_URL=https://your-domain.example

FACEBOOK_APP_ID=                    # from Meta App → Settings → Basic
FACEBOOK_APP_SECRET=                # from Meta App → Settings → Basic
FACEBOOK_REDIRECT_URI=${APP_URL}/facebook/callback
FACEBOOK_WEBHOOK_VERIFY_TOKEN=      # any random string, must match dashboard
```

---

## 7. Acceptance Criteria

1. A user can register, log in, connect a Facebook Page, and disconnect it.
2. Sending a message to a connected Page produces an auto-reply within a few seconds.
3. `/privacy` and `/terms` are reachable without authentication and are linked in Meta App settings.
4. `GET /webhook/facebook` with the correct verify token returns the challenge string; an incorrect token returns 403.
5. `POST /webhook/facebook` with a valid signature is accepted; an invalid signature is rejected.
