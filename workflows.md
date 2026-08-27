# GHL Automation Workflows

## Workflow 1: Postcard Campaign Launch

### Trigger
Manual bulk action (or batch import) — adding tag `batch-2026-09-01-gd`

### Steps
1. **Generate QR token** — Create unique `custom_field_qr_token` (UUID) for each contact
2. **Generate tracking URL** — Set `custom_field_tracking_url` = `https://track.yourdomain.com/{qr_token}`
3. **Log postcard sent date** — Set `custom_field_postcard_sent_date` = today's date
4. **Tag contact** — Add `postcard-sent` tag
5. **Create Click2Mail order** — Webhook POST to Click2Mail API to create postcard with merge variables (business_name, phone_number, qr_code_url, campaign_id)
6. **Log campaign session** — Store Click2Mail order ID in contact notes

### Click2Mail API Payload (Webhook Action)
```
Method: POST
URL: https://api.click2mail.com/molpro/v1/postcards
Headers:
  Content-Type: application/json
  X-API-Key: {{your_api_key}}
Body:
{
  "address": {
    "name": "{{contact.business_name}}",
    "address1": "{{contact.address1}}",
    "city": "{{contact.city}}",
    "state": "{{contact.state}}",
    "postalCode": "{{contact.postal_code}}"
  },
  "frontHtml": "[HTML template with {{contact.business_name}} merged]",
  "backHtml": "[HTML template with {{contact.business_name}}, {{custom.phone_number}}, {{custom.qr_code_url}}, {{custom.campaign_id}} merged]",
  "size": "4x6",
  "mailType": "usps_first_class",
  "sendDate": "[today + 1 business day]"
}
```

---

## Workflow 2: QR Code Scanned

### Trigger
Inbound webhook from tracking page — event type `qr_scanned`

### Steps
1. **Find contact by QR token** — Match `{{qr_token}}` to `custom_field_qr_token`
2. **Log scan timestamp** — Set `custom_field_qr_scan_date` = current time
3. **Tag contact** — Add `qr-scanned` tag, remove `postcard-sent` tag
4. **Send personalized video** — Webhook to trigger personalized video generation (Clarity/Loom)
5. **Update lead score** — +30 points
6. **Send SMS** — Text the number from the contact:
   > "Hey [Business Name] — just saw you checked out the AI front desk preview. Give the number a call from your business line to hear it live. — Jason"
7. **Notify Jason** — Send push notification or SMS to Jason:
   > "QR scanned: [Business Name] — [time] — phone: [contact.phone]"
8. **Add to priority callback queue** — Set `custom_field_priority` = true

---

## Workflow 3: Callback Requested (Different Number)

### Trigger
Inbound webhook from tracking page — event type `callback_requested`

### Steps
1. **Find contact by QR token** — Match `{{qr_token}}` to `custom_field_qr_token`
2. **Log the callback number** — Set `custom_field_alternate_phone` = submitted number
3. **Tag contact** — Add `callback-requested` tag
4. **Send SMS to alternate number**:
   > "Hey [Business Name] — Jason here. You requested a callback about the AI front desk system. I'll reach out shortly. In the meantime, if you want to hear the system live, call [your_GHL_number] from your business line. — Jason"
5. **Notify Jason** — Push notification:
   > "Callback requested: [Business Name] — alternate #: [alternate_phone] — [time]"
6. **Add to callback queue** — Priority medium

---

## Workflow 4: Inbound Call from Prospect

### Trigger
Appointment booked via Voice AI agent (when prospect books a call with Jason)

### Steps
1. **Tag contact** — Add `hot-lead`, `callback-scheduled`
2. **Send confirmation SMS to prospect**:
   > "Confirmed: Jason will call [Business Name] on [date] at [time]. If you need to reschedule, reply here. — Jason"
3. **Notify Jason** — SMS/notification:
   > "BOOKED: [Business Name] — [date] [time] — phone: [contact.phone]"
4. **Create opportunity in pipeline** — Move to "Discovery Call" stage
5. **Send calendar block** — Add to Jason's calendar via calendar integration
6. **Send pre-call brief** — Email/webhook to Jason with contact details, pain points from call

---

## Workflow 5: Lead Not Ready (Warm Lead)

### Trigger
Voice AI agent tags contact as `warm-lead`

### Steps
1. **Add to nurture sequence** — Enter 14-day email/SMS drip
2. **Day 1 SMS**:
   > "Hey [Business Name] — thinking more about capturing those missed calls? The system's already built for you. Call [GHL number] anytime to hear it live. No rush."
3. **Day 4 SMS**:
   > "Quick question — what's the one call you wish you could've answered yesterday? Whatever it is, the system can handle it. Call when you're ready to try."
4. **Day 7 Email**:
   Send a short case study or success story from another garage door company
5. **Day 14 SMS**:
   > "Last thought — you only pay if it works. The system's standing by. Call [GHL number] to give it a try."
6. **Day 14** — Remove from nurture, add to `long-term-nurture`

---

## Workflow 6: Ringless Voicemail Follow-Up (3 Days After Postcard)

### Trigger
Scheduled — 3 days after `custom_field_postcard_sent_date`

### Condition
Only run if contact does NOT have tag `qr-scanned` or `callback-requested`

### Steps
1. **Collect phone numbers** — Gather contact.phone for all eligible contacts
2. **Send to SlyBroadcast API** — Webhook POST:
   ```
   POST https://www.mobile-sphere.com/gateway/vmb.php
   Content-Type: application/x-www-form-urlencoded
   
   c_uid={{your_slybroadcast_email}}
   c_password={{your_slybroadcast_password}}
   c_url={{url_to_your_recorded_audio.wav}}
   c_audio=wav
   c_phone={{comma_separated_phone_numbers}}
   c_callerID={{your_caller_id_number}}
   c_date=now
   c_title=GD-MetroDetroit-Batch1-FollowUp
   mobile_only=1
   ```
3. **Log session ID** — Store returned session_id in `custom_field_slybroadcast_session_id`
4. **Tag contact** — Add `vm-drop-sent`

---

## Workflow 7: Voice AI Agent Dispatches

### Trigger
Inbound call answered by Voice AI agent

### Steps
1. **Check caller ID against contacts** — If match found, personalize greeting
2. **Log call** — Create call log entry with duration, disposition, transcript
3. **Update contact** — Set last call date, increment call count
4. **Run intent logic** — Based on conversation outcome:
   - If hot → trigger Workflow 4
   - If warm → trigger Workflow 5
   - If cold → tag `cold-lead`
5. **If unknown number and caller provides business name** — Create new contact with business name, phone, tag `inbound-prospect`