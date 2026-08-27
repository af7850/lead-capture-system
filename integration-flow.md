# Integration Flow — Full System Wiring

## How Every Piece Connects

```
                  +---------------------+
                  |  Prospect List CSV  |
                  |  (business_name,    |
                  |   phone, address)   |
                  +----------+----------+
                             |
                             v
                  +---------------------+
                  |  GHL Import         |  ← Manual import into contacts
                  +----------+----------+
                             |
                             v
            +----------------+----------------+
            |                                 |
            v                                 v
  +---------------------+         +---------------------+
  | QR Token Generator  |         | Click2Mail API      |
  | (per contact UUID)  |         | POST /postcards     |
  +----------+----------+         +----------+----------+
             |                                |
             |   QR URL embedded in           |
             |   postcard back HTML           |
             +----------+---------------------+
                        |
                        v
             +---------------------+
             |   Postcard Mailed   |
             |  (USPS, 1-3 days)   |
             +---------------------+
                        |
          Prospect scans QR or not...
                        |
            +-----------+-----------+
            |                       |
            v                       v
  +---------------------+  +---------------------+
  | Tracking Page       |  | SlyBroadcast        |
  | (QR scanned)        |  | Ringless VM on Day 3|
  | POSTs webhook to    |  | (only if no scan)   |
  | GHL Workflow #2     |  +----------+----------+
  +----------+----------+             |
            |                          |
            v                          v
  +---------------------+  +---------------------+
  | GHL Workflow #2     |  | GHL Workflow #6     |
  | (QR scan response)  |  | (VM drop response)  |
  | SMS, video, notify  |  | Tag, log session    |
  +----------+----------+  +----------+----------+
             |                          |
             +------------+-------------+
                          |
                          v
              +---------------------+
              |  Prospect Calls     |
              |  [GHL Phone Number] |
              +----------+----------+
                          |
              Caller ID matched or not
                          |
            +-----------+-----------+
            |                       |
            v                       v
  +---------------------+  +---------------------+
  | Tier 1-2: Known     |  | Tier 3: Unknown     |
  | Agent greets by     |  | Agent asks "which   |
  | business name       |  | business?"          |
  +----------+----------+  +----------+----------+
            |                          |
            +------------+-------------+
                         |
                         v
              +---------------------+
              | Voice AI Agent      |
              | Discovery → Bridge  |
              | → Book Jason call   |
              +---------------------+
                         |
                    Outcome?
                         |
        +----------------+-------+----------------+
        |                        |                |
        v                        v                v
  +-----------+          +-----------+      +-----------+
  | Hot Lead  |          | Warm Lead |      | Cold Lead |
  | Booked    |          | Nurture   |      | Tagged    |
  | call w/   |          | 14-day    |      | & filed   |
  | Jason     |          | drip      |      |           |
  +-----------+          +-----------+      +-----------+
        |
        v
  +---------------------+
  | Jason's 15-min Call |
  | → Close             |
  | → GHL Sub-account   |
  | → Client's Agent    |
  | → $200-300/mo MRR   |
  +---------------------+
```

## SlyBroadcast Setup

### Account Setup
1. Login to SlyBroadcast
2. Record your audio file via:
   - Option A: Use SlyBroadcast Recording Center (record over phone)
   - Option B: Upload WAV/MP3 file (record on your phone, upload)
3. Note the audio file name exactly as it appears (case sensitive)
4. Get your API credentials (email + password)

### Audio Script (15-20 seconds, your voice)
> "Hey [Business Name], Jason here. I sent you a postcard about an AI system that answers your garage door calls when you're too busy working to pick up. It's already built for your company. Call [GHL number] from your business line and you'll hear it live. If you can't call from that line, check the postcard — there's a QR code that'll get you sorted. Talk soon."

### API Integration via GHL Webhook
Use GHL's Custom Webhook action with:
- Method: POST
- Content-Type: application/x-www-form-urlencoded
- URL: https://www.mobile-sphere.com/gateway/vmb.php
- Body parameters (map from GHL custom values):
  - c_uid: (your email)
  - c_password: (your password)
  - c_url: (URL to your recorded audio)
  - c_audio: wav
  - c_phone: {{contact.phone}} (comma-separated list)
  - c_callerID: (your caller ID)
  - c_date: now
  - c_title: GD-MetroDetroit-Batch1-FollowUp
  - mobile_only: 1

## QR Tracking Server

### Minimal Node.js Handler
```javascript
// POST /track/:token
// - Look up token in contact DB
// - Log scan with timestamp, user agent, IP
// - POST to GHL inbound webhook URL
// - Redirect to tracking page HTML (merged with business data)

// GET /track/:token
// - Serve tracking page HTML with business name, phone, webhook URL merged
```

### Deployment
- Host on CT 106 (nginx reverse proxy) or $5 DigitalOcean droplet
- Domain: track.yourdomain.com
- CNAME to your server IP
- SSL via Let's Encrypt

## Click2Mail Account Setup

### Plan
Start with Developer plan ($0/mo, pay-per-piece)
- 500 mailings/month limit
- $0.645/postcard (4x6)
- API access with 1 webhook

### API Key
Generate API key from Click2Mail dashboard
Store in GHL as a custom field or external credential store

### Template Setup
- Upload `postcard-front.html` as a saved template in Click2Mail (optional — you can also pass HTML inline in the API call)
- Upload `postcard-back.html` as a saved template
- Test with a single test postcard to your own address first

## GHL Settings Required

### Custom Fields (Contact Level)
Create these custom fields in your GHL sub-account:
- `business_name` (text) — The prospect's business name
- `campaign_id` (text) — Campaign identifier
- `campaign_batch` (text) — Batch identifier
- `qr_token` (text) — Unique QR token per contact
- `postcard_sent_date` (date) — When postcard was mailed
- `qr_scan_date` (datetime) — When QR was scanned
- `slybroadcast_session_id` (text) — Ringless VM session ID
- `agent_deployed` (boolean) — Whether the client's agent is live
- `alternate_phone` (phone) — Alternate callback number
- `priority` (boolean) — Priority flag for callback queue

### Tags (Contact Level)
Create these tags:
- `postcard-sent`, `qr-scanned`, `callback-requested`
- `vm-drop-sent`
- `hot-lead`, `warm-lead`, `cold-lead`
- `callback-scheduled`
- `inbound-prospect`
- `client-active`, `client-paused`

### Pipeline Stages (Opportunities)
Create pipeline:
- Stage 1: Discovery Call (booked with Jason)
- Stage 2: Demo Completed
- Stage 3: Proposal Sent
- Stage 4: Closed Won
- Stage 5: Closed Lost

### Phone Number
Provision 1 GHL phone number:
- Configure with Voice AI agent as "Answer calls directly"
- Set working hours to 24/7
- Disable call forwarding
- Enable call recording
- Enable transcription

### Number Pool (Optional)
If you want to scale to multiple numbers later, create a Number Pool. For the pilot, one number is sufficient.