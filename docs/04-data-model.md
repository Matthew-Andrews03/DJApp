# 04 — Control-Plane Data Model

Postgres (Supabase). Every table except `clients` carries `client_id` with row-level security;
the portal uses RLS-scoped access, n8n uses a service role through the internal API only.

## Tenancy & configuration

```sql
clients
  id uuid pk, created_at, status (prospect|onboarding|active|paused|churned),
  business_name, industry, timezone, hubspot_company_id, tier,
  billing_status, notes

locations
  id uuid pk, client_id fk, name, address_json (street, city, state, zip),
  phone, hours_json, holiday_hours_json, gbp_location_id, place_id,
  brightlocal_location_id, listings_health_score, primary bool

client_profile              -- the "10-15 strategic questions" + intake output
  client_id pk/fk, services text[], service_areas text[],
  target_keywords text[], competitors jsonb[],           -- {name, website, place_id, fb_page}
  brand_voice jsonb,                                     -- tone, phrases to use/avoid, persona name
  qualification_questions jsonb[],                       -- per-module Q flows (speed-to-lead)
  campaign_angles jsonb[],                               -- approved reactivation offers
  booking_link, website_url, website_platform, review_link,
  approval_preferences jsonb                             -- per-module approval modes

workflow_config             -- THE SWITCHBOARD
  client_id fk, module text,                             -- review_automation, speed_to_lead, ...
  enabled bool default false,
  settings jsonb,                                        -- module-specific config w/ defaults
  updated_by, updated_at
  pk (client_id, module)

users
  id uuid pk, client_id fk null (null = Seekly staff), email, role
  (owner|staff|seekly_admin), auth via Supabase Auth magic link
```

## Connections & credentials

```sql
connections
  id uuid pk, client_id fk, provider                     -- google_business, google_analytics,
                                                          -- search_console, meta, wordpress,
                                                          -- crm, booking, phone, social
  provider_account_id, scopes text[],
  encrypted_access_token, encrypted_refresh_token,        -- app-layer encryption, key in env
  expires_at, connection_status (pending|active|broken|revoked),
  last_verified_at, metadata jsonb                        -- selected GBP location, page id, etc.

comms_provisioning
  client_id pk/fk, twilio_subaccount_sid, phone_number,
  a2p_brand_status, a2p_campaign_status (none|submitted|approved|rejected),
  a2p_campaign_type, inbound_email_address                -- {slug}@in.seekly.app
```

## Canonical events & customer data (per client — never mixed with Seekly's own CRM)

```sql
events
  id uuid pk (idempotency key), client_id fk, type, occurred_at, received_at,
  source, payload jsonb, processed_at, status (pending|processed|failed|ignored)

customers                    -- the client's customers (reactivation audience)
  id uuid pk, client_id fk, name, phone, email,
  first_seen_at, last_visit_at, visit_count, lifetime_value_estimate,
  consent_basis (existing_customer|lead_inbound|imported_list),
  source, tags text[], external_ids jsonb                 -- POS ids etc.
  unique (client_id, phone)

opt_outs                     -- cross-engine, per client
  client_id fk, phone, opted_out_at, source_message_id
  pk (client_id, phone)
```

## Messaging & conversations

```sql
messages                     -- the message ledger (idempotent sends)
  id uuid pk, client_id fk, customer_id fk, direction (in|out),
  kind (transactional|conversational|marketing), engine,   -- wf-1|wf-2|wf-3
  body, media_urls, twilio_sid, status (queued|sent|delivered|failed|blocked),
  blocked_reason (opt_out|quiet_hours|freq_cap|a2p_pending),
  idempotency_key unique,                                  -- e.g. event_id + step
  campaign_id fk null, conversation_id fk null, created_at

conversations                -- speed-to-lead / reactivation threads
  id uuid pk, client_id fk, customer_id fk, engine,
  status (active|qualified|booked|handed_off|cold|closed),
  lead_source, lead_event_id fk, context jsonb,            -- extracted intent, answers
  first_response_seconds, outcome, booked bool, created_at, closed_at

campaigns                    -- reactivation
  id uuid pk, client_id fk, angle, copy_variants jsonb, audience_filter jsonb,
  status (draft|approved|sending|paused|done),
  stats jsonb                                              -- sent, replies, stops, bookings, revenue_est
campaign_members
  campaign_id fk, customer_id fk, status (queued|sent|replied|booked|excluded),
  pk (campaign_id, customer_id)                            -- makes batches resumable
```

## Reputation, content, intel

```sql
reviews
  id text pk (provider review id), client_id fk, location_id fk, provider (google),
  rating, author, body, received_at, response_body, response_status
  (none|drafted|pending_approval|published|failed), responded_at

content_items
  id uuid pk, client_id fk, type (blog), topic, title, body_html, schema_jsonld,
  status (queued|drafted|pending_approval|approved|published|failed),
  published_url, published_at, metrics jsonb               -- gsc clicks, rankings

syndicated_posts
  id uuid pk, content_item_id fk, client_id fk, channel (gbp|facebook),
  body, cta_url_utm, status, posted_at, external_post_id, clicks

competitor_snapshots
  id uuid pk, client_id fk, competitor_key, captured_at,
  website_text_hash, website_extract jsonb,                -- offers, prices, services
  gbp_stats jsonb                                          -- rating, review_count, post_count

intel_digests
  id uuid pk, client_id fk, period, body_html, findings jsonb, sent_at
```

## Observability & value proof

```sql
activity_log                 -- every action every engine takes; powers dashboards + case study
  id uuid pk, client_id fk, engine, action, status (ok|failed|skipped),
  entity_type, entity_id, detail jsonb, created_at

metrics_daily                -- rollups per client per day (computed nightly)
  client_id fk, date, module, metrics jsonb
  pk (client_id, date, module)
  -- e.g. reviews: {requests_sent, reviews_received, avg_rating, responses_published}
  --      leads:   {leads, median_response_s, qualified, booked}
  --      reactivation: {sent, replies, bookings, revenue_est}
```

## Design notes

- **Idempotency everywhere:** `events.id` from adapters, `messages.idempotency_key`,
  `reviews.id` from provider, `campaign_members` as the resume ledger. Any workflow may be
  re-run safely at any point.
- **Client isolation:** RLS on `client_id`; n8n never queries the DB directly — only the
  internal API, which scopes by the `client_id` in the event/run context.
- **Seekly's own sales data lives in HubSpot,** not here. This database holds the *clients'*
  operational data. The only bridge is summary-field sync (WF-0) keyed by
  `clients.hubspot_company_id`.
- **`workflow_config.settings` carries every default listed in 03-workflows.md** so behavior
  is data, not code — the same master workflow serves a golf venue and an HVAC company.
