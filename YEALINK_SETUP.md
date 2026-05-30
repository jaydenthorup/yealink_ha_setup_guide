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

## That's It

Pick up the phone and talk to Home Assistant.