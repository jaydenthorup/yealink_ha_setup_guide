# Yealink Phone Setup

## Web Interface Configuration

1. Open the phone's web UI: `http://<phone-ip>`
2. Go to **Account** → **Account 1**
3. Fill in:
   - **Server Address**: Your Asterisk server IP
   - **SIP User ID**: `yealink`
   - **Authenticate User ID**: `yealink`
   - **Authenticate Password**: `password123`
   - **Enable this account**: Yes
4. Click **Confirm**

Phone will register with Asterisk and display **Registered** status.

## Hotline Configuration

1. Go to **Feature** → **General Settings** (or similar - varies by firmware)
2. Find **Hotline**:
   - **Hotline Number**: `100`
   - **Hotline Delay**: `4` (seconds)
3. Click **Confirm**

Now when you pick up the handset, it will automatically dial Home Assistant after 4 seconds.

## Auto Answer (required for the HA → phone intercom)

So Home Assistant announcements auto-answer on the speaker instead of just ringing:

1. Go to **Account** → **Account 1** → **Advanced** (or **Account** → **Codec/Advanced**).
2. Enable **Auto Answer** (a.k.a. "Auto Answer" / intercom auto-answer).
   - In a config file this is `account.1.auto_answer = 1`.
3. **Confirm**.

The phone still only auto-answers when the incoming INVITE carries the intercom
header that Asterisk injects (`Alert-Info: info=alert-autoanswer`, see
`configs/extensions.conf` → `[intercom]`). Normal calls still ring.

If your firmware exposes it, the relevant Yealink header values the phone honours
for hands-free answer are any of:

```
Alert-Info: auto answer
Alert-Info: info=alert-autoanswer
Alert-Info: alert-autoanswer ;info=alert-autoanswer
Alert-Info: answer-after=0
Call-Info: ;answer-after=0
```

## Codecs

Make sure **OPUS** is enabled and, ideally, first in the codec list — Home
Assistant Voice uses opus. (Account → Codec.) This is separate from the intercom
issue above; opus is not the reason a call fails to ring.

## Adding a second phone

Repeat the above on the second phone, but give it the credentials that match its
own `[yealinkN]` block in `pjsip.conf` — e.g. **SIP User ID** and
**Authenticate User ID** `yealink2`, password `password123`, server = the same
Asterisk IP. Each phone **must use a different account name**: two phones sharing
one AOR with `remove_existing=yes` will evict each other's registration (the
classic "only one phone ever works" symptom).

Same hotline (100), Auto Answer, and OPUS settings as the first phone. Both
phones then ring for HA announcements, and both can dial the assistant.

## That's It

- Pick up the phone and talk to Home Assistant.
- Have Home Assistant call the phone (e.g. an automation) and it answers on
  speaker as an intercom.
