# WhatsApp Gateway — Standalone Module Extract

Extracted from [nanocoai/nanoclaw](https://github.com/nanocoai/nanoclaw). This directory contains the complete working implementation of NanoClaw's WhatsApp gateway, plus all the interface contracts it depends on.

Two gateway variants exist:
- **Baileys (native)** — `src/channels/whatsapp.ts` — direct WhatsApp Web protocol via `@whiskeysockets/baileys`. No Meta account needed. Uses QR or pairing-code auth.
- **WhatsApp Cloud API** — `src/channels/whatsapp-cloud.ts` — official Meta Business API via `@chat-adapter/whatsapp`. Requires a Meta for Developers app.

---

## How the WhatsApp Gateway Works

### Message Flow (Inbound)

```
Phone/WhatsApp → [Baileys WebSocket] → messages.upsert event
                                             ↓
                                    whatsapp.ts processes:
                                    - LID→phone JID translation
                                    - media download
                                    - slash-command question answers
                                    - fromMe echo filter
                                             ↓
                         setupConfig.onInbound(chatJid, null, message)
                                             ↓
                                    router.ts routeInbound()
                                    - resolve/create messaging group
                                    - evaluate engage mode (pattern/mention/sticky)
                                    - write to session DB messages_in
                                    - wake container
```

### Message Flow (Outbound)

```
Agent container writes messages_out row
        ↓
host delivery.ts reads row, calls:
adapter.deliver(platformId, null, message)
        ↓
whatsapp.ts deliver():
  - ask_question → text with /slash-command options
  - reaction → sock.sendMessage({ react })
  - file → sock.sendMessage with image/video/audio/document
  - text → formatWhatsApp(markdown) → sock.sendMessage({ text })
        ↓
sock.sendMessage() via Baileys WebSocket → WhatsApp
```

### WhatsApp Cloud Flow (Inbound)

```
WhatsApp platform → POST /webhook/whatsapp (HTTP)
                          ↓
               webhook-server.ts routes to:
               Chat SDK bridge → chat.onDirectMessage()
                          ↓
               setupConfig.onInbound(platformId, null, message)
                          ↓
               same router.ts path as Baileys
```

### Authentication (Baileys)

1. First run: Baileys connects to WhatsApp Web servers
2. If `WHATSAPP_PHONE_NUMBER` set → requests numeric pairing code via `sock.requestPairingCode(phone)`, saves to `store/pairing-code.txt`
3. Otherwise → generates QR code, printed to log / terminal
4. User scans QR / enters pairing code on phone
5. Credentials saved to `store/auth/` (multi-file auth state)
6. Subsequent restarts: credentials loaded from `store/auth/creds.json`, no re-auth needed

**Important**: Baileys v7 (`@whiskeysockets/baileys@7.0.0-rc.9`) is pinned. The library is unmaintained. The adapter includes a custom `resolveWaWebVersion()` that fetches the current WA Web version from wppconnect.io as a fallback because Baileys' built-in fetcher is frequently rate-limited (429).

---

## File Structure

```
src/
  channels/
    whatsapp.ts           ← Baileys native adapter (self-registers)
    whatsapp-cloud.ts     ← Meta Cloud API adapter (self-registers via Chat SDK bridge)
    adapter.ts            ← ChannelAdapter interface contract
    channel-registry.ts   ← Adapter registration + lifecycle
    chat-sdk-bridge.ts    ← Bridge: wraps @chat-adapter/* libs → ChannelAdapter
    ask-question.ts       ← NormalizedOption types + helpers
  webhook-server.ts       ← HTTP server for Cloud API webhooks (/webhook/{name})
  router.ts               ← Inbound routing: adapter → session DB
  config.ts               ← Env config (ASSISTANT_NAME, DATA_DIR, etc.)
  env.ts                  ← .env file reader
  types.ts                ← Core DB types (MessagingGroup, Session, etc.)
setup/
  whatsapp-auth.ts        ← Standalone Baileys auth runner (QR / pairing-code)
  whatsapp-setup-flow.ts  ← Interactive setup wizard (clack UI)
  install-whatsapp.sh     ← Idempotent install script (fetches from channels branch)
  install-whatsapp-cloud.sh ← Same for Cloud adapter
```

---

## Environment Variables

### Baileys (Native WhatsApp)

| Variable | Required | Description |
|----------|----------|-------------|
| `WHATSAPP_PHONE_NUMBER` | Optional | Phone number for pairing-code auth (digits only, with country code, e.g. `14155551234`). If absent, falls back to QR code. |
| `WHATSAPP_ENABLED` | Optional | Set to `true` to force-enable the adapter even without a phone number (QR mode). Without this or `WHATSAPP_PHONE_NUMBER`, the adapter is skipped if `store/auth/creds.json` doesn't exist. |
| `ASSISTANT_HAS_OWN_NUMBER` | Optional | Set `true` if the WhatsApp number is dedicated to the bot. When `false` (default), outbound messages are prefixed with `{ASSISTANT_NAME}: ` so the user can distinguish bot messages in a shared chat. |
| `ASSISTANT_NAME` | Optional | Bot display name (default: `Andy`). Used as message prefix in shared mode and for bot-mention normalization. |

### WhatsApp Cloud API

| Variable | Required | Description |
|----------|----------|-------------|
| `WHATSAPP_ACCESS_TOKEN` | **Yes** | System User access token with `whatsapp_business_messaging` permission. |
| `WHATSAPP_PHONE_NUMBER_ID` | **Yes** | The Phone Number ID from Meta for Developers → WhatsApp → API Setup (NOT the phone number itself). |
| `WHATSAPP_APP_SECRET` | **Yes** | App Secret from Meta for Developers → Settings → Basic. Used to verify webhook signatures. |
| `WHATSAPP_VERIFY_TOKEN` | **Yes** | Any random string you choose. Must match what you set in Meta's webhook configuration. |

### General

| Variable | Default | Description |
|----------|---------|-------------|
| `WEBHOOK_PORT` | `3000` | HTTP port for the Cloud API webhook server. |
| `DATA_DIR` | `./data` | Base directory for attachments and container data. |

---

## Step-by-Step Integration Guide

### Option A: Baileys (Personal/Business WhatsApp, no Meta account)

**Prerequisites**
- Node.js 18+ / Bun
- A WhatsApp-linked phone number
- `pnpm` (or npm/yarn)

**1. Install dependencies**

```bash
pnpm install @whiskeysockets/baileys@7.0.0-rc.9 qrcode@1.5.4 @types/qrcode@1.5.6 pino@9.6.0
```

**2. Register the adapter**

In your channel barrel (`src/channels/index.ts`):
```typescript
import './whatsapp.js';
```

The adapter self-registers by calling `registerChannelAdapter('whatsapp', { factory })` on import.

**3. Set up auth directory**

```bash
mkdir -p store/auth
```

**4. Authenticate (first time)**

QR code:
```bash
pnpm exec tsx setup/whatsapp-auth.ts -- --method qr
```

Pairing code:
```bash
pnpm exec tsx setup/whatsapp-auth.ts -- --method pairing-code --phone 14155551234
# Watch store/pairing-code.txt for the 8-digit code, enter it on your phone
```

**5. Initialize adapters in your host**

```typescript
import { initChannelAdapters } from './src/channels/channel-registry.js';

await initChannelAdapters((adapter) => ({
  onInbound: (platformId, threadId, message) => {
    // Handle incoming message
    console.log('Inbound from', platformId, message.content);
  },
  onInboundEvent: (event) => { /* ... */ },
  onMetadata: (platformId, name, isGroup) => { /* ... */ },
  onAction: (questionId, selectedOption, userId) => { /* ... */ },
}));
```

**6. Send a message**

```typescript
import { getChannelAdapter } from './src/channels/channel-registry.js';

const wa = getChannelAdapter('whatsapp');
await wa?.deliver('14155551234@s.whatsapp.net', null, {
  kind: 'chat',
  content: { text: 'Hello from the bot!' },
});
```

**Platform ID format for Baileys:**
- DMs: `{phone}@s.whatsapp.net` (e.g. `14155551234@s.whatsapp.net`)
- Groups: `{groupId}@g.us` (e.g. `12345678901234567890@g.us`)

---

### Option B: WhatsApp Cloud API (Meta Business)

**Prerequisites**
- Meta for Developers account
- Verified Business Manager
- A publicly accessible HTTPS URL for webhooks

**1. Install dependencies**

```bash
pnpm install @chat-adapter/whatsapp@4.27.0
```

**2. Meta setup**

1. Create an app at [developers.facebook.com](https://developers.facebook.com/apps/) (type: Business)
2. Add the **WhatsApp** product
3. Under WhatsApp → API Setup: note the **Phone Number ID**, generate a **permanent System User access token** with `whatsapp_business_messaging` permission
4. Under WhatsApp → Configuration: set webhook URL to `https://your-domain/webhook/whatsapp`, set a **Verify Token**, subscribe to `messages` webhook field
5. Under Settings → Basic: copy the **App Secret**

**3. Configure `.env`**

```bash
WHATSAPP_ACCESS_TOKEN=your-access-token
WHATSAPP_PHONE_NUMBER_ID=your-phone-number-id
WHATSAPP_APP_SECRET=your-app-secret
WHATSAPP_VERIFY_TOKEN=your-verify-token
```

**4. Register the adapter**

```typescript
import './whatsapp-cloud.js';
```

The Cloud adapter registers itself and starts a webhook server on `WEBHOOK_PORT` (default 3000). Point your Meta webhook configuration to `https://your-domain/webhook/whatsapp`.

**5. Same `initChannelAdapters` + `deliver` API as Baileys**

**Platform ID format for Cloud API:**  
The Phone Number ID from Meta (not the phone number). All conversations use this as the platform ID.

---

## Key Behaviors

### LID Handling (Baileys)
WhatsApp uses "LID" (a new identifier format) alongside phone JIDs. The adapter:
1. Checks local `lidToPhoneMap` cache
2. Uses `remoteJidAlt` / `participantAlt` from Baileys v7 `extractAddressingContext` (available on every inbound message)
3. Falls back to `sock.signalRepository.lidMapping.getPNForLID()`
4. Always resolves to `{phone}@s.whatsapp.net` before emitting to the host

### Echo Loop Prevention
- For non-self-chats: drops all `msg.key.fromMe === true` messages
- For self-chat (messaging own number): allows `fromMe` messages that are NOT in `sentMessageCache`

### Outgoing Queue
Messages sent while disconnected are queued in memory (`outgoingQueue`). On reconnect, the queue is flushed in order.

### Group Metadata Sync
On connect: fetches all participating groups via `sock.groupFetchAllParticipating()`. Repeated every 24h. 1-minute TTL cache for outbound sends.

### Markdown → WhatsApp Formatting
Applied to all outbound text (before prefix):
- `**bold**` → `*bold*`
- `*italic*` → `_italic_`
- `## Heading` → `*Heading*`
- `[text](url)` → `text (url)`
- Code blocks/inline code: preserved unchanged

### Ask Question (Baileys)
WhatsApp has no button UI. Questions are sent as text with slash command options:
```
*Approve deployment?*

Deploy to production now?

Reply with:
  /yes
  /no
  /defer
```
User's `/yes` reply is matched against `pendingQuestions` map, `onAction` is called, and a confirmation is sent.
