# Asterisk PJSIP Configuration for Home Assistant

A working Asterisk config that bridges Yealink SIP phones to Home Assistant Voice —
in both directions, with the Home Assistant → phone direction working as an
**intercom** (the phone auto-answers hands-free on its speaker).

## What This Does

- Yealink phone registers with Asterisk.
- **Phone → HA:** pick up the phone, it auto-dials extension `100`, Asterisk
  bridges the call into Home Assistant Assist. Talk to your smart home.
- **HA → phone (intercom):** Home Assistant places a call to the `asterisk.local`
  VoIP endpoint; Asterisk rings the Yealink with an intercom header so the phone
  **auto-answers on speaker** and plays HA's announcement (e.g. "Internet is
  down") without anyone picking up the handset.

## How It's Built

**pjsip.conf** defines:

- **[yealink]** endpoint — Your phone registers here with username `yealink` / password `password123`
- **[homeassistant]** endpoint — Outbound connection to Home Assistant at `192.168.2.245:5060`
- Codecs: OPUS, ULAW, ALAW, GSM (in that priority)
- `direct_media=no` — Audio goes through Asterisk, not peer-to-peer

**extensions.conf** routes:

- `[from_yealink]` extension **100** — the hotline (phone → Home Assistant)
- `[from_yealink]` extension **s** — offhook without dialing, auto-dials 100
- `[from_yealink]` extension **_X.** — any other digits forward to Home Assistant
- `[from_homeassistant]` — Home Assistant → Yealink. `Dial(PJSIP/yealink,30,b(intercom^s^1))`
- `[intercom]` — pre-dial handler that adds `Alert-Info: info=alert-autoanswer`
  to the outbound INVITE so the Yealink auto-answers on speaker

## Why the intercom bits matter (the part that is easy to get wrong)

Home Assistant's built-in **Voice over IP** integration does not implement SIP
`REGISTER`. It only learns a device when the device *calls it*, and it builds
outgoing INVITEs from whatever `From` URI the device last presented:

- If the phone called HA as a bare IP, the device id becomes `sip:192.168.2.44`
  (no user part), and HA then sends `INVITE sip:192.168.2.44` — which a Yealink
  discards as a spoofing attempt (`code=0x04030004 ... disgard this invite!`).
  Symptom: `start_conversation` / `announce` "does nothing — no ring, nothing".
- That is **not** a codec problem — opus is fine.
- Routing through Asterisk sidesteps this entirely, because Asterisk's HA-facing
  peer (`from_user=asterisk`) presents a username, so HA's INVITE is accepted.
  Point HA's automations at the `assist_satellite.asterisk_local` entity.

Also note: **do not `Answer()` in `[from_homeassistant]` before the `Dial()`**.
If Asterisk answers HA's leg first, HA starts streaming TTS into a channel with
no second leg up and the beginning of the announcement is dropped. Letting
`Dial()` answer when the phone answers keeps the audio intact.

## Deploy

1. Backup your current configs:
   ```bash
   cp /etc/asterisk/pjsip.conf /etc/asterisk/pjsip.conf.backup
   cp /etc/asterisk/extensions.conf /etc/asterisk/extensions.conf.backup
   ```

2. Copy these files to Asterisk:
   ```bash
   scp configs/pjsip.conf root@<asterisk-ip>:/etc/asterisk/
   scp configs/extensions.conf root@<asterisk-ip>:/etc/asterisk/
   ```

3. Update these values in pjsip.conf:
   - Line with `contact=sip:192.168.2.245:5060` → change IP to your Home Assistant
   - Line with `match=192.168.2.245` → change IP to your Home Assistant
   - `[yealink_auth]` section → update username/password if you want different creds

4. Reload:
   ```bash
   asterisk -rx 'dialplan reload'
   asterisk -rx 'pjsip reload'
   ```
   Or restart: `systemctl restart asterisk`

5. Set up the phone using YEALINK_SETUP.md (including the **Auto Answer** step
   needed for intercom).

## Home Assistant side

In the automation, target the Asterisk peer, not the phone:

```yaml
- action: assist_satellite.start_conversation     # two-way line (talk back)
  data:
    start_message: "Internet is Down"
  target:
    entity_id: assist_satellite.asterisk_local
```

Use `assist_satellite.announce` instead if you want it to play and hang up.

Verify the call bridged — both legs should be `Up`:

```bash
asterisk -rx 'core show channels concise'
# PJSIP/homeassistant-...  from_homeassistant  Up  Dial PJSIP/yealink,30,b(intercom^s^1)
# PJSIP/yealink-...        Up (Outgoing Line)   <-- phone auto-answered
```
