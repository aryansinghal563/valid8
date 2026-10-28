# Valid8

**One-tap verification against fake kidnapping and family-emergency scam calls.**

Team Hermes · Reforged '26 (n8n track) · Aryan Singhal, Anshu Gautam, Rohit Bora, Saurav Chakraborty

---

## The problem

Scammers clone a child's voice from a few seconds of social-media audio and call the parent: "We have your child. Don't hang up. Send money on UPI now."

Their key trick is **isolation**. While the parent stays on the line, they cannot check on the child, so they panic and pay.

## The solution

Valid8 is a Telegram bot powered by n8n. The parent stays on the call and taps **Suspicious call**. Valid8 then checks on the child in the background:

1. The child instantly gets **"Are you safe?"** with **I'm safe** and **Help** buttons.
2. If the child taps I'm safe, they confirm with a private 4-digit **Safe PIN**. The bot deletes the PIN message from the chat.
3. If the child does not reply in 30 seconds, Valid8 asks the family's **trusted contacts**.
4. The parent gets a clear verdict, usually within a minute.

| Verdict | When | What the parent is told |
|---|---|---|
| **SAFE** | Child sent the correct Safe PIN | Don't pay. High risk of scam |
| **HELP** | Child pressed Help | Call 112 now |
| **LOCKED** | 3 wrong PINs (phone may be with someone else) | Don't pay. Call 112 |
| **UNCONFIRMED** | No PIN and no reply from the child or contacts | Don't pay on the caller's word. Call 112 |

Valid8 never says "safe" without the child's own PIN, and never says "scam confirmed".

One flow covers every scam that claims a loved one is in danger: fake kidnapping, fake arrest, accident or hospital scams, the "new number" scam, and voice-clone calls to elderly parents.

## Architecture

```
Parent / Child / Trusted contacts (Telegram app)
        │
Telegram Bot API ──webhook──► Cloudflare Tunnel (HTTPS) ──► n8n (self-hosted, Docker, Azure VPS)
        ▲                                                     │
        └──────────────────── sendMessage ────────────────────┘
                                                              │
                                        n8n Data Tables: members, cases
```

No call interception, no audio processing and no telephony provider. The parent's tap is the trigger.

## n8n workflow: `Valid8 - Bot Router`

```
Telegram Trigger → Kya Aaya? (Switch)
  ├─ Text  → Alarm? ─ true  → Find Parent → Find Child → Ask Child → Open Case
  │                   │                                  → Wait 30s → Check Reply → No Reply?
  │                   │                                  → Tell Parent Asking Contacts → Find Contact → Ask Contact
  │                   │                                  → Wait 30s → Check Reply Again → Still No Reply?
  │                   │                                  → Tell Parent Unconfirmed → Close Unconfirmed
  │                   └ false → Is PIN? ─ true  → Delete PIN message → PIN Sender → PIN Parent
  │                                      │        → Get Case → Case Open? → PIN Correct?
  │                                      │           ├ true  → Tell Parent Verified → Close Safe
  │                                      │           └ false → Count Wrong PIN → Tell Parent PIN
  │                                      └ false → Menu (shows the Suspicious call button)
  └─ Button → Answer Button
            → Find Responder → Family Parent → Get Case Btn → Case Open Btn? → Mark Replied → From Contact?
                 ├ true  → Tell Parent Contact → Close Contact
                 └ false → Safe? ─ true  → Ask PIN → Wait 45s → Get Case PIN → PIN Pending?
                                  │                 → Tell Parent PIN Timeout → Close PIN Timeout
                                  └ false → Tell Parent Help → Close Help
```

## Data tables

**`members`**

| Column | Type | Example |
|---|---|---|
| `chatId` | String | Telegram chat ID |
| `name` | String | Saurav |
| `role` | String | `parent`, `child` or `contact` |
| `familyId` | String | `fam1` |
| `pin` | String | 4-digit Safe PIN (child only) |

**`cases`**

| Column | Type | Values |
|---|---|---|
| `familyId` | String | `fam1` |
| `status` | String | `open`, `safe`, `help`, `locked`, `contact_safe`, `contact_no`, `no_response`, `pin_timeout` |
| `startedAt` | String | ISO timestamp |
| `attempts` | Number | Wrong PIN count |
| `replied` | String | `yes` / `no` |

## Setup

1. Run n8n in Docker with a public HTTPS URL (we use a Cloudflare Tunnel) and set `WEBHOOK_URL` to it.
2. Create a Telegram bot with @BotFather and add it as a Telegram credential in n8n.
3. Create the `members` and `cases` data tables shown above.
4. Import `workflows/valid8-bot-router.json`, then re-select the credential and both data tables in each node.
5. Publish the workflow. Every family member presses **Start** on the bot once.
6. Add one row per person to `members` (same `familyId`, correct `role`, and a PIN for the child).

## Status

| Built and tested | Planned |
|---|---|
| Suspicious call → child popup | WhatsApp Cloud API channel (same n8n logic) |
| Safe PIN check, PIN deleted from chat | Post-call evidence: recording → transcript → PDF for 1930 / Chakshu |
| Help alert | SMS fallback for trusted contacts |
| 3 wrong PINs → LOCKED | Self-service onboarding with invite links |
| Trusted contacts after 30 s | Hashed PIN storage |
| UNCONFIRMED and PIN-timeout verdicts | |
| Old buttons disabled after a case closes | |

## Known limitations

- PINs are stored in plain text in the prototype. Production will store a hash.
- Very fast repeated PIN attempts can race the counter. Production needs an atomic counter.
- Valid8 needs the child and contacts to have joined the bot. If nobody answers, Valid8 can only say UNCONFIRMED.
- A correct PIN confirms the child is safe. It does not by itself prove who the caller is.

## Emergency numbers (India)

- **112**: single emergency number
- **1930** or cybercrime.gov.in: report financial fraud
- **Sanchar Saathi (Chakshu)**: report suspected fraud calls

## Built with

n8n · Telegram Bot API · Docker · Cloudflare Tunnel
