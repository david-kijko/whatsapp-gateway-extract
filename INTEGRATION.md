# WhatsApp Gateway — Coding Agent Integration Guide

This document explains exactly how to wire the WhatsApp gateway into another coding agent system. It covers every function call, event, message format, and data flow.

---

## The Core Contract

The gateway is built around one interface: `ChannelAdapter` (defined in `src/channels/adapter.ts`).

```typescript
interface ChannelAdapter {
  name: string;
  channelType: string;     // 'whatsapp' or 'whatsapp-cloud'
  supportsThreads: boolean; // always false for WhatsApp

  setup(config: ChannelSetup): Promise<void>;  // starts the connection
  teardown(): Promise<void>;                   // stops the connection
  isConnected(): boolean;

  deliver(platformId: string, threadId: string | null, message: OutboundMessage): Promise<string | undefined>;
  setTyping?(platformId: string, threadId: string | null): Promise<void>;
  syncConversations?(): Promise<ConversationInfo[]>;
}
```

Your system calls `setup()` to connect and receive events. It calls `deliver()` to send messages. That's the entire surface area.

---

## Step 1: Register & Initialize

```typescript
import './src/channels/whatsapp.js';           // self-registers on import
// OR
import './src/channels/whatsapp-cloud.js';     // for Cloud API

import { initChannelAdapters } from './src/channels/channel-registry.js';
import type { ChannelSetup } from './src/channels/adapter.js';

await initChannelAdapters((adapter) => makeChannelSetup(adapter));
```

`initChannelAdapters` iterates registered adapters, calls `factory()` to instantiate (returns `null` if credentials missing), then calls `adapter.setup(yourHandlers)`.

---

## Step 2: Implement ChannelSetup (your event handlers)

```typescript
function makeChannelSetup(adapter: ChannelAdapter): ChannelSetup {
  return {
    /**
     * Called when an inbound message arrives.
     * platformId: WhatsApp JID (e.g. "14155551234@s.whatsapp.net" or "abc@g.us")
     * threadId:   always null for WhatsApp (no threads)
     * message:    InboundMessage
     */
    async onInbound(platformId, threadId, message) {
      const content = message.content as {
        text: string;
        sender: string;          // phone JID of sender
        senderName: string;      // WhatsApp display name (push name)
        attachments?: Array<{
          type: 'image' | 'video' | 'audio' | 'document';
          name: string;
          localPath: string;     // relative path under DATA_DIR
        }>;
        fromMe: boolean;
        isBotMessage: boolean;   // true if message starts with "AssistantName:"
        isGroup: boolean;
        chatJid: string;         // same as platformId
      };

      // message.id       — WhatsApp message key ID
      // message.kind     — always 'chat' for Baileys, 'chat-sdk' for Cloud
      // message.timestamp — ISO string
      // message.isMention — true for DMs (WhatsApp DMs are always "mentioned")
      // message.isGroup   — true for group chats

      await routeToYourSystem(platformId, message);
    },

    /**
     * Called by admin/CLI transports that want to inject a message with
     * a custom reply-to address. Not needed for basic WhatsApp integration.
     */
    async onInboundEvent(event) { /* optional */ },

    /**
     * Called when adapter discovers conversation metadata.
     * Use this to maintain a local group name cache.
     */
    onMetadata(platformId, name, isGroup) {
      if (name) myGroupNameCache.set(platformId, name);
    },

    /**
     * Called when user clicks a button / answers an ask_question.
     * questionId:      the ID you passed in the ask_question payload
     * selectedOption:  the value the user chose
     * userId:          the sender's JID (Baileys) or userId (Cloud)
     */
    onAction(questionId, selectedOption, userId) {
      myQuestionHandler(questionId, selectedOption, userId);
    },
  };
}
```

---

## Step 3: Send Messages

```typescript
import { getChannelAdapter } from './src/channels/channel-registry.js';
import type { OutboundMessage } from './src/channels/adapter.js';

const wa = getChannelAdapter('whatsapp'); // or 'whatsapp-cloud'

// --- Plain text ---
await wa.deliver(platformId, null, {
  kind: 'chat',
  content: { text: 'Hello world' },
});

// --- Markdown text ---
await wa.deliver(platformId, null, {
  kind: 'chat',
  content: { markdown: '**Bold** and _italic_ and `code`' },
  // Baileys adapter auto-converts markdown to WhatsApp format
});

// --- File attachment ---
import fs from 'fs';
await wa.deliver(platformId, null, {
  kind: 'chat',
  content: { text: 'Here is your report' },
  files: [{
    filename: 'report.pdf',
    data: fs.readFileSync('/path/to/report.pdf'),
  }],
});

// --- Ask a question (Baileys: text with /slash-commands) ---
await wa.deliver(platformId, null, {
  kind: 'chat',
  content: {
    type: 'ask_question',
    questionId: 'q-1234',
    title: 'Approve deployment?',
    question: 'Deploy version 2.1.0 to production?',
    options: [
      { label: 'Yes', value: 'yes', selectedLabel: '✅ Approved' },
      { label: 'No', value: 'no', selectedLabel: '❌ Rejected' },
    ],
  },
});
// User replies "/yes" → onAction('q-1234', 'yes', senderJid) fires

// --- Reaction ---
await wa.deliver(platformId, null, {
  kind: 'chat',
  content: {
    operation: 'reaction',
    messageId: 'BAEBB123...',  // WhatsApp message key ID
    emoji: '👍',
  },
});

// --- Typing indicator ---
await wa.setTyping?.(platformId, null);
```

---

## Message Formats

### InboundMessage (what you receive)

```typescript
interface InboundMessage {
  id: string;          // WhatsApp message key ID (e.g. "3EB0CF1234567890ABCD")
  kind: 'chat';        // always 'chat' for Baileys; 'chat-sdk' for Cloud
  timestamp: string;   // ISO 8601

  isMention?: boolean; // true for DMs (always addressed to bot), undefined for groups
  isGroup?: boolean;   // true for group chats (@g.us JIDs)

  content: {           // JSON-stringified before DB storage; parsed at routing
    text: string;
    sender: string;        // phone JID ("14155551234@s.whatsapp.net") or group participant
    senderName: string;    // WhatsApp push name
    fromMe: boolean;       // true if the authenticated account sent this
    isBotMessage: boolean; // true if text starts with "AssistantName:"
    isGroup: boolean;
    chatJid: string;       // the conversation JID (= platformId)
    attachments?: Array<{
      type: 'image' | 'video' | 'audio' | 'document';
      name: string;          // filename
      localPath: string;     // "attachments/{filename}" relative to DATA_DIR
    }>;
  };
}
```

### OutboundMessage (what you send)

```typescript
interface OutboundMessage {
  kind: string;         // e.g. 'chat'
  content: unknown;     // see formats below
  files?: OutboundFile[]; // optional binary attachments
}

// Text message content:
{ text: 'Hello' }
{ markdown: '**Bold** text' }

// Ask question content:
{
  type: 'ask_question',
  questionId: string,   // your ID, returned in onAction
  title: string,        // displayed as bold header
  question: string,     // the question body
  options: NormalizedOption[],  // { label, value, selectedLabel }
}

// Reaction content:
{
  operation: 'reaction',
  messageId: string,  // WhatsApp message key ID to react to
  emoji: string,
}
```

---

## Platform IDs

### Baileys
- **DM**: `{e164_phone_without_plus}@s.whatsapp.net`
  - US number +1-415-555-1234 → `14155551234@s.whatsapp.net`
- **Group**: `{groupId}@g.us`
  - e.g. `120363123456789012@g.us`

### WhatsApp Cloud API
- Platform ID = the **Phone Number ID** from Meta for Developers (not the actual phone number)
- All conversations use the same Phone Number ID as the top-level identifier

---

## Message Flow Diagram

```
INBOUND
────────────────────────────────────────────────────────────────────
  WhatsApp servers
       │
       ▼ (Baileys WebSocket / Cloud webhook)
  whatsapp.ts / whatsapp-cloud.ts
       │  LID translation, media download, echo filter, question matching
       ▼
  ChannelSetup.onInbound(platformId, null, message)
       │
       ▼ (your router)
  Resolve conversation → create/find session
       │
       ▼
  Write to messages_in (your session DB)
       │
       ▼
  Wake agent container

OUTBOUND
────────────────────────────────────────────────────────────────────
  Agent writes messages_out row
       │
       ▼ (your host delivery loop)
  getChannelAdapter('whatsapp').deliver(platformId, null, message)
       │
       ▼
  whatsapp.ts deliver():
    - ask_question → text + pendingQuestions map
    - reaction     → sock.sendMessage({ react })
    - file         → sock.sendMessage(mediaMessage)
    - text         → formatWhatsApp() + optional prefix → sock.sendMessage({ text })
       │
       ▼
  WhatsApp servers → user's phone
```

---

## Events to Listen For

| Event | Handler | When it fires |
|-------|---------|---------------|
| New message | `onInbound` | Every message in a DM or group the bot is in |
| Button/question answer | `onAction` | User replies to an `ask_question` with a `/command` |
| Group discovery | `onMetadata` | On every inbound message (group JIDs), and after `syncConversations()` |

---

## State You Need to Manage

1. **Authentication credentials** — persisted to `store/auth/` by Baileys automatically. Must survive restarts.
2. **Conversation → agent group mapping** — your system maps `platformId` to which agent handles it.
3. **Pending questions map** — the adapter manages this internally (`pendingQuestions` map in `whatsapp.ts`). You don't need to track it.
4. **Session DB** — the NanoClaw architecture uses per-session SQLite DBs (`messages_in` / `messages_out`). In your integration, this can be any queue/state mechanism that holds messages until the agent processes them.

---

## Initialization Sequence

```
1. Import adapter module (triggers registerChannelAdapter call)
2. Call initChannelAdapters(setupFn)
   a. factory() runs → checks for store/auth/creds.json (Baileys) or env vars (Cloud)
   b. Returns null → adapter skipped (no credentials)
   c. Returns adapter → setup(yourHandlers) called
      - Baileys: connectSocket() → Baileys WebSocket connects → resolveFirstOpen resolves
      - Cloud:   Chat SDK initializes → registerWebhookAdapter() starts HTTP server
3. Adapter is now live in activeAdapters map
4. Your deliver() calls work
```

---

## Error Handling

### Connection Failures (Baileys)
The adapter auto-reconnects on every close except `DisconnectReason.loggedOut`. Reconnect loop:
1. Immediate reconnect attempt
2. On failure: retry after `RECONNECT_DELAY_MS` (5000ms)

### Disconnected During Send
If `connected === false`, the message goes into `outgoingQueue` (in-memory). On next reconnect, `flushOutgoingQueue()` drains it in order.

### Logged Out
`connection.update` fires with `DisconnectReason.loggedOut`. Delete `store/auth/` and re-authenticate.

### WA Web Version Fetch Failure
The adapter throws during `setup()` if it can't fetch the current WhatsApp Web version from either wppconnect.io or Baileys' built-in fetch. Check network connectivity.

---

## Teardown

```typescript
import { teardownChannelAdapters } from './src/channels/channel-registry.js';
await teardownChannelAdapters(); // calls adapter.teardown() on each
```

Baileys teardown calls `sock.end(undefined)` — clean WebSocket close.

---

## Threading Model

WhatsApp has no threads. `threadId` is always `null` for both inbound and outbound. `supportsThreads` is `false`. The router collapses any thread context at the adapter boundary.

---

## Security Notes

1. **Attachment filenames are attacker-controlled** — the adapter sanitizes with `isSafeAttachmentName()` to prevent path traversal.
2. **Echo filter** — the adapter drops `fromMe` messages to prevent infinite reply loops, with special handling for self-chats.
3. **Baileys credentials** in `store/auth/` are session keys that give full access to the linked WhatsApp account. Protect this directory.
4. **Cloud API webhook signature** — the `@chat-adapter/whatsapp` library verifies the `WHATSAPP_APP_SECRET` HMAC on every inbound webhook.
