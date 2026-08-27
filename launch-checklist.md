# Launch Checklist — Go-Live Readiness

## Phase 1: Foundation Setup (1 session)

### GHL
- [ ] Create custom fields: `business_name`, `campaign_id`, `campaign_batch`, `qr_token`, `postcard_sent_date`, `qr_scan_date`, `slybroadcast_session_id`, `agent_deployed`, `alternate_phone`, `priority`
- [ ] Create tags: `postcard-sent`, `qr-scanned`, `callback-requested`, `vm-drop-sent`, `hot-lead`, `warm-lead`, `cold-lead`, `callback-scheduled`, `inbound-prospect`, `client-active`, `client-paused`
- [ ] Create pipeline: Discovery Call → Demo Completed → Proposal Sent → Closed Won → Closed Lost
- [ ] Provision 1 GHL phone number
- [ ] Record your ringless voicemail audio (15-20 seconds) and upload to a URL or SlyBroadcast

### Click2Mail
- [ ] Sign up for Developer plan ($0/mo)
- [ ] Generate API key
- [ ] Test postcard — send 1 to your address to verify print quality
- [ ] Upload or prepare HTML templates for front and back

### QR Tracking
- [ ] Deploy tracking handler (Node.js on CT 106 or $5 droplet)
- [ ] Set up domain (track.yourdomain.com) with SSL
- [ ] Test QR scan → redirect → webhook to GHL end-to-end

### Voice AI Agent
- [ ] Create Voice AI agent in GHL
- [ ] Paste the system prompt from `agent-prompt.md`
- [ ] Add knowledge base (garage door FAQs, pricing ranges, service area)
- [ ] Set up actions: appointment booking, SMS, webhook triggers
- [ ] Configure phone number assignment (Answer calls directly)
- [ ] Test calling the number — verify Tier 1, 2, 3 routing works

### SlyBroadcast
- [ ] Confirm audio file is recorded and accessible via URL
- [ ] Test single drop to your own phone
- [ ] Note API credentials for GHL webhook

## Phase 2: Batch Assembly

- [ ] Generate prospect list (CSV): 200-300 garage door companies in Metro Detroit
- [ ] Columns: business_name, phone, address, website, campaign_id, campaign_batch
- [ ] Import into GHL as contacts
- [ ] Run Workflow 1 — generates QR tokens, sends batch to Click2Mail
- [ ] Confirm Click2Mail order submitted — verify tracking

## Phase 3: Campaign Live

### Day 0
- [ ] Postcards go in mail (Click2Mail handles printing + posting)
- [ ] No other actions — let postcards arrive

### Day 1-2 (postcards arriving)
- [ ] Monitor for QR scans (Workflow 2 fires automatically)
- [ ] If QR scans come in — Jason receives notifications
- [ ] Respond within 1 hour to any hot leads

### Day 3
- [ ] Run Workflow 6 — SlyBroadcast ringless voicemail drop to non-scanners
- [ ] Monitor for callbacks to the GHL number
- [ ] Voice AI agent handles inbound calls automatically

### Day 4-7
- [ ] Monitor agent call logs daily
- [ ] Tag and disposition every call
- [ ] Jason takes booked discovery calls
- [ ] Track: scans, calls, booked calls, closes

## Phase 4: Measure & Iterate

### Metrics to Track
| Metric | Target | How to Measure |
|--------|--------|---------------|
| Postcards sent | 200-300 | Click2Mail order logs |
| QR scans | 5-8% of sent | GHL `qr-scanned` tag count |
| Callback rate | 3-5% of sent | GHL `hot-lead` + `warm-lead` tags |
| Booked discovery calls | 1.5-2% of sent | Pipeline Stage 1 count |
| Close rate | 40-60% of discovery calls | Pipeline Stage 4 / Stage 1 ratio |
| MRR per client | $200-300/mo | GHL subscription records |
| Time to first close | <14 days from batch send | First Closed Won date |

### After First Batch
- [ ] Review which copy variant performed best
- [ ] Review which vertical (garage door vs HVAC vs plumbing) had highest callback rate
- [ ] Adjust agent prompt based on actual call transcripts
- [ ] Rinse and repeat — next batch of 300 with refined copy