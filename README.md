# Asterisk PJSIP Configuration for Home Assistant

This is a working Asterisk config that bridges Yealink SIP phones to Home Assistant Voice. 

## What This Does

- Yealink phone registers with Asterisk
- When you pick up the phone, it auto-dials extension 100
- Asterisk bridges the call to Home Assistant
- You can talk to Home Assistant Assist

## How It's Built

**pjsip.conf** defines:
- **[yealink]** endpoint - Your phone registers here with username `yealink` / password `password123`
- **[homeassistant]** endpoint - Outbound connection to Home Assistant at `192.168.2.245:5060`
- Codecs: OPUS, ULAW, ALAW, GSM (in that priority)
- `direct_media=no` - Audio goes through Asterisk, not peer-to-peer

**extensions.conf** routes:
- Extension **100** - The hotline (calls Home Assistant)
- Extension **s** - Offhook without dialing (auto-dials 100 after 4 seconds)
- Extension **_X.** - Any other digits forward to Home Assistant

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
   asterisk -rx 'core reload'
   ```
   Or restart: `systemctl restart asterisk`

5. Set up the phone using YEALINK_SETUP.md


