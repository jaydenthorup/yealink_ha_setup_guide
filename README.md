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
  down") without anyone picking up the handset — then the call is dropped by the
  `S(8)` Dial option (see below).

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

## Hanging up after the announcement

HA's VoIP satellite never ends the call by itself: `assist_satellite.announce`
loops the message until the phone hangs up, and `start_conversation` listens
until the phone hangs up — and an auto-answering intercom does neither. There is
no `assist_satellite` stop/hangup service (only `announce`, `start_conversation`,
`ask_question`).

So the call is capped on the Asterisk side with `Dial`'s `S(x)` option (hang up
`x` seconds after the called party answers):

```
same => n,Dial(PJSIP/yealink,30,S(8)b(intercom^s^1))
```

Set `x` to comfortably exceed the announcement length. When Asterisk hangs up,
HA's SIP leg gets a BYE and the service call returns cleanly. Keep
`start_conversation` (not `announce`) for a single one-way message — `announce`
repeats until the cap.

## Two phones

Add one `[yealinkN]` endpoint block per extra phone in `pjsip.conf` — its own
auth and AOR, and the **AOR name must differ from the first phone's** (two
phones cannot share one AOR while `remove_existing=yes`, or they evict each
other). Give each phone `context=from_yealink` so it can also dial the
assistant (hotline 100 → HA).

### One HA call reaches exactly ONE phone

Home Assistant's VoIP satellite holds **one** call. Two consequences, both
verified the hard way:

- `Dial(PJSIP/a&PJSIP/b)` rings both but connects only the **first** to answer,
  so only one phone speaks.
- **Chaining** two `Dial()`s in a single call does not work either: as soon as
  the first phone's media stops, HA's `voip_utils` tears its *own* call down (a
  ~2s RTP watchdog), so the second `Dial()` has no caller left.

So the automation fires the announcement **once per phone**:

```yaml
  actions:
  - action: assist_satellite.start_conversation
    data: {start_message: "Internet is Down"}
    target: {entity_id: assist_satellite.asterisk_local}
  - action: assist_satellite.start_conversation   # 2nd call = 2nd phone
    data: {start_message: "Internet is Down"}
    target: {entity_id: assist_satellite.asterisk_local}
```

and the dialplan **alternates** which phone each call reaches:

```
exten => asterisk,1,GotoIf($["${GLOBAL(HA_PHONE)}" = "yealink2"]?y2)
 same => n,Set(GLOBAL(HA_PHONE)=yealink2)
 same => n,Dial(PJSIP/yealink,30,S(6)b(intercom^s^1))
 same => n,HangUp()
 same => n(y2),Set(GLOBAL(HA_PHONE)=yealink)
 same => n,Dial(PJSIP/yealink2,30,S(6)b(intercom^s^1))
 same => n,HangUp()
exten => _X.,1,Goto(asterisk,1)
```

The phones announce a few seconds apart, each with the full message.

### Do NOT use `Page()`/`ConfBridge` here

It looks tempting — `Page` bridges every answering phone into one conference so
both play at once — and it briefly works, but three fatal problems:

1. **It clips the announcement.** `Page` answers the caller up-front, so HA's
   TTS starts before the phones have joined; the first ~1s is lost.
2. **`TIMEOUT(absolute)` does not fire inside `ConfBridge`.** The call hangs
   open and the phones are left **off the hook**. (`Page` has no `S(x)` like
   `Dial`. And `TIMEOUT(absolute)` is a duration in seconds, not an epoch —
   `$[${EPOCH}+8]` silently never fires.)
3. **It deadlocked this PBX.** Channels stopped answering hangup requests and
   only a full restart cleared them. `Page` also needs `codec_opus` to transcode
   opus↔slin for mixing — extra machinery for a method you should avoid anyway.

A plain `Dial()` has none of these: the caller is answered only when the phone
answers (no clip) and `S(x)` is a reliable hangup.

### Aside: opus ↔ slin transcoding (only needed for conferences)

If you *do* want `Page`/`ConfBridge`/`MeetMe` for something else, mixing needs
an opus translator, which a stock Asterisk lacks (only `res_format_attr_opus`,
passthrough):

```bash
asterisk -rx 'core show translation paths opus'   # all "No Translation Path" = missing
```

Install the Digium module (Asterisk 22 shown), then **restart** — Asterisk reads
third-party XML docs only at startup, so `xmldoc reload` won't pick it up:

```bash
cd /tmp
curl -sLO https://downloads.digium.com/pub/telephony/codec_opus/asterisk-22.0/x86-64/codec_opus-22.0-current-x86_64.tar.gz
tar -xzf codec_opus-22.0-current-x86_64.tar.gz
cp codec_opus-22.0_*/codec_opus.so               /usr/lib/asterisk/modules/
cp codec_opus-22.0_*/codec_opus_config-en_US.xml /var/lib/asterisk/documentation/thirdparty/
systemctl restart asterisk
```

Watch the LSB init script: `systemctl restart asterisk` can report success while
the old daemon survives (`Unit process NNN remains running after unit stopped`)
and the ctl socket disappears. If `asterisk -rx` then says "Unable to connect",
do `pkill -x asterisk; sleep 3; systemctl stop asterisk; systemctl start asterisk`.


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
