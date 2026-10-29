# Integrations & Field Mapping — Implementation Plan

> Customer connects their Salesforce → we read their schema → they map their fields onto
> ours → scoring can run.
>
> This plan covers the whole path from "customer clicks Connect" to "the scorecard has every
> required field it needs." It is written against the vocabulary already in
> [backend-architecture.md](backend-architecture.md) §7 (Connection, Capability,
> FieldMapping) and §10.2 (SourceBinding), not a new set of names.
>
> **Status:** plan only. Nothing is built. Two decisions in section 2 need an answer before
> the first PR, and one of them (2.2) blocks the Salesforce setup.

---

## 1. The model, stated precisely

| Term | What it is | Where it lives |
| ---- | ---------- | -------------- |
| **Canonical field** | One of Zeke's ~50 scorecard data points. `account.arr`, `account.renewal_date`. Global, same for every customer. | `integrations.field_catalog` — global table |
| **Source schema** | A snapshot of *this customer's* Salesforce: objects and fields, including their custom ones. Schema only, never data. | `integrations.source_object` / `source_field` — per tenant |
| **Field mapping** | "Your `ARR__c` is our `account.arr`." One row per canonical field. | `integrations.field_mapping` — per tenant |
| **Connection** | This tenant's authorised Salesforce. §7.1. | `integrations.connection` — per tenant |
| **Readiness** | Are all *required* canonical fields mapped? Computed, never stored. | derived |

The flow:

```
 signup ──▶ Connect Salesforce ──▶ Nango OAuth consent ──▶ connection active
                                                                │
                                   ┌────────────────────────────┘
                                   ▼
                        discover schema (describe API)
                        store objects + fields, no data
                                   │
                                   ▼
                        mapping UI: our 50 fields on the left,
                        dropdown of their fields on the right
                        (pre-filled with our suggestions)
                                   │
                                   ▼
                        save mapping ──▶ readiness check
                                   │
                        ┌──────────┴───────────┐
                        ▼                      ▼
             all required mapped      some required missing
                        │                      │
                        ▼                      ▼
              scoring can run      "map these 6 fields, or
                                    connect another system"
```

**Multiple integrations is the same loop, run again.** Salesforce supplies 30 of the 50;
the customer connects a second system later and maps the remaining 20. Nothing about the
design is Salesforce-specific except the connector itself — which is the point.

---

## 2. Two decisions needed before we start

### 2.1 "We do not store their data" — what exactly does that exclude?

This is the one place the plan contradicts the existing architecture.
[backend-architecture.md](backend-architecture.md) §7.6 defines the sync pipeline as
`pull batch → map → **upsert canonical (customer-data)** → emit → persist cursor`, and §10.1
scores daily from that stored canonical data. The new requirement removes the upsert step.

Fetching on demand is **technically fine**. SOQL is bulk — 2000 records per page, all mapped
fields in one query. A 10,000-account tenant costs roughly 20–50 API calls for a full daily
read. Volume is not the problem. Four other things are:

| Consequence | Detail |
| ----------- | ------ |
| **No trends** | [Zeke schema.docx](<Zeke schema.docx>) §11 needs history — *"active licences fell from 78% to 51% in 60 days"* is a risk driver. A live snapshot cannot produce it. |
| **No point-in-time** | A past score cannot be reproduced or explained. §10 of the schema doc requires lineage. |
| **Interactive reads are slow** | A DSM opening an account page triggers a live Salesforce call. |
| **No deletion detection** | Not actually a problem — a full read each time never sees stale rows. This one is a *benefit*. |

**Recommendation — three tiers, not one rule:**

| Tier | Example | Store? |
| ---- | ------- | ------ |
| Raw source records | the full `Account` row, contact emails, case bodies | ❌ never — fetch, project, discard |
| Zeke-derived values | `feature_snapshot` numeric features, scores, bands, drivers | ✅ yes — this is our output, not their data |
| Account directory | external id, account name, owner id | ⚠️ **needs your answer** |

Without the third tier the UI cannot list accounts, and no score can be attached to anything
a human recognises — every screen becomes a live Salesforce call. "Compliance" usually means
personal data and record contents, not the existence of an account name. **Worth confirming
precisely what the constraint forbids**, because an account directory is the difference
between a usable product and a slow one.

This plan assumes: raw records never stored, derived values stored, account directory
stored. Say if that is wrong and sections 6 and 9 change.

### 2.2 Classic Connected App or Packaged External Client App? — blocks Salesforce setup

Your customers' Salesforce orgs are **different orgs from your developer org**. That rules
out the default choice:

- A **Local External Client App** only works inside the org that created it. Fine for a POC
  against your own org, useless for customers.
- A **classic Connected App** works across orgs — this is what every Salesforce ISV has used.
  Salesforce has announced support ends **Summer '27**.
- A **Packaged External Client App** (2GP managed package) is the forward path, and is more
  setup work.

**Recommendation: classic Connected App now**, with the migration to a Packaged ECA tracked
as dated work before Summer '27. Getting to a working customer integration matters more
today than avoiding one future migration, and the OAuth surface Nango talks to is identical
either way — the migration is a credential swap, not a rewrite.

This decision determines what you create in section 10, so it needs answering first.

---

## 3. Data model

All tables in the `integrations` schema, all tenant-scoped with RLS forced, following the
pattern in `Migration0001Tenancy` (`tenantRlsPolicySql`, `grantAppRoleSql`). Except
`field_catalog`, which is global.

### 3.1 `field_catalog` — Zeke's 50 data points (global)

```
key                 text primary key    -- 'account.arr'
entity              text                -- account | contact | opportunity | case | contract
label               text                -- 'Annual Recurring Revenue'
description         text                -- shown as dropdown help
data_type           text                -- see 5.1
is_required         boolean
enum_values         jsonb               -- for data_type = 'enum'
unit                text                -- 'months', 'percent', 'usd'
cardinality         text                -- 'one' | 'derived'  (see 3.5)
introduced_version  int                 -- catalog version this appeared in
required_from       timestamptz         -- null until it becomes required (see 8.1)
deprecated_at       timestamptz
sort_order          int
```

**Global, not per tenant.** Every customer is scored against the same scorecard; a
per-tenant catalog would mean per-tenant scoring semantics, which §10 of the architecture
explicitly does not have.

**Seeded by migration, not by an admin UI.** The other team is still deciding the fields, so
they will churn — but a migration is reviewable, auditable and deploys in minutes. A staff
console editor can come later; starting there would let someone break every tenant's
readiness with a typo and no review.

Adding 5 fields later = one migration inserting 5 rows. No code change, no connector change,
no API change. That is the flexibility requirement, met.

### 3.2 `connection` — the tenant's authorised Salesforce

```
id                      uuid pk
tenant_id               uuid        -- RLS
connector_key           text        -- 'crm-salesforce'
connector_version       int
external_connection_id  text        -- Nango's connection id
status                  text        -- pending | active | reauthorize_required | error | disabled
instance_url            text        -- each Salesforce org has its own host
config                  jsonb
last_error_code         text
connected_at            timestamptz
created_at, updated_at
unique (tenant_id, connector_key)   -- v1: one Salesforce per tenant
```

No tokens here. Credentials live in Nango and are resolved per call through the
`ConnectionAuthBroker` port (section 4.1).

### 3.3 `schema_snapshot`, `source_object`, `source_field`

```
schema_snapshot
  id, tenant_id, connection_id
  status          text          -- running | ready | failed
  object_count, field_count     int
  checksum        text          -- detects "nothing changed" cheaply
  started_at, completed_at

source_object
  id, tenant_id, snapshot_id, connection_id
  api_name        text          -- 'Account', 'Custom_Thing__c'
  label           text
  is_custom       boolean
  queryable       boolean
  unique (tenant_id, snapshot_id, api_name)

source_field
  id, tenant_id, source_object_id
  api_name        text          -- 'ARR__c'
  label           text          -- 'Annual Recurring Revenue'
  source_type     text          -- Salesforce describe type: currency, picklist, ...
  is_custom       boolean
  is_nillable     boolean
  length, precision, scale      int
  reference_to    text[]        -- for type 'reference'
  picklist_values jsonb         -- [{value,label,active}]
  unique (tenant_id, source_object_id, api_name)
```

**Why snapshots rather than overwriting:** re-discovery must not delete rows that live
mappings point at. A new snapshot is written, then mappings are re-validated against it,
then the old snapshot is retired. A field that disappeared shows up as a *broken mapping*
with a name the customer recognises, instead of a foreign-key error.

Only `Account`, `Contact`, `Opportunity`, `Case`, `Contract`, `User`, `Task`, `Event` and
every custom object are discovered. A full Salesforce describe is ~500 objects, most of them
platform internals nobody will ever map.

### 3.4 `field_mapping` — the actual answer

```
id                      uuid pk
tenant_id               uuid
connection_id           uuid        -- which integration supplies this field
canonical_field_key     text        -- fk to field_catalog.key
source_object           text        -- 'Account'
source_path             text        -- 'ARR__c' or 'Owner.Email' (see 3.5)
source_type             text        -- copied at map time, to detect drift
coercion                text        -- see 5.2
value_map               jsonb       -- picklist value → our enum value (see 5.3)
status                  text        -- active | broken | needs_review
status_reason           text
mapped_by               uuid
mapped_at               timestamptz
unique (tenant_id, canonical_field_key)
```

**That unique constraint is what makes multi-integration work.** One canonical field has
exactly one source. 30 rows point at the Salesforce connection, 20 point at whatever comes
next. Readiness is one query over this table and does not care how many integrations exist.

### 3.5 Scope line: `one` vs `derived`

Some canonical fields are a field on a record (`account.name` → `Account.Name`). Others are
an aggregate (`open_case_count`, `days_since_last_activity`). Aggregates cannot be mapped to
a dropdown entry — there is nothing to point at.

- `cardinality = 'one'` → mappable. Direct field, or a lookup path up to **2 hops**
  (`Owner.Email`, `Account.Parent.Name`). Two hops covers the real cases and keeps the SOQL
  builder simple.
- `cardinality = 'derived'` → **not shown in the mapping UI at all.** Zeke computes it from
  other mapped fields. It still counts toward readiness, but its requirement is "the fields
  it derives from are mapped."

Getting this line wrong is how the mapping UI ends up with 50 dropdowns the customer cannot
answer. Be strict about it when the catalog is seeded.

---

## 4. Ports and connectors

### 4.1 `ConnectionAuthBroker` (restore from the POC)

The port from `feat/connection-auth-broker` is still correct and should be cherry-picked
rather than rewritten: `startAuthorization` / `resolve` / `revoke`, with the Nango adapter
behind it and a `dev-static` adapter for local work. Keeping it a port is what lets Nango be
replaced later without touching any business code.

### 4.2 New capability: `schema.describe`

```ts
export interface SourceSchemaDescriber {
  listObjects(ctx: ConnectorCallContext): Promise<Result<readonly SourceObjectDescriptor[]>>;
  describeObject(apiName: string, ctx: ConnectorCallContext): Promise<Result<SourceObjectSchema>>;
}
```

Added to `CAPABILITIES` in `integrations/contracts`. Salesforce implements it with
`/services/data/v59.0/sobjects` and `/sobjects/{name}/describe`. Any future connector
implements the same two methods and the whole mapping UI works for it unchanged.

`describe` respects Field-Level Security, so we only ever discover fields the connected user
can actually read — permission problems surface at connect time rather than mid-score.

### 4.3 New capability: `records.query`

```ts
export interface RecordQuerier {
  query(spec: RecordQuerySpec, ctx: ConnectorCallContext): AsyncIterable<RecordPage>;
}
```

`RecordQuerySpec` is `{ object, fields, filter?, since? }` — vendor-neutral. The Salesforce
adapter turns it into SOQL and follows `nextRecordsUrl`. This replaces the fixed
`crm.accounts.read` of the POC: with mapping-driven fields, the connector can no longer know
its own select list at compile time.

---

## 5. Types and coercion

### 5.1 Canonical types

`string`, `text`, `integer`, `decimal`, `money`, `percent`, `boolean`, `date`, `timestamp`,
`enum`, `email`, `url`, `id_ref`.

### 5.2 Compatibility matrix

The dropdown only offers compatible fields. This is most of the value of the whole
feature — it stops a customer mapping a picklist onto a currency and discovering it at
scoring time.

| Salesforce type | May map to | Coercion |
| --------------- | ---------- | -------- |
| `string`, `textarea` | string, text, email, url | trim; validate format for email/url |
| `email` | email, string | — |
| `phone`, `url` | string, url | — |
| `int` | integer, decimal | — |
| `double` | decimal, integer | integer requires explicit `round` or `truncate` |
| `currency` | money, decimal | carries a currency code — see below |
| `percent` | percent, decimal | Salesforce stores 0–100; our `_pct` is 0–100. Direct. |
| `boolean` | boolean | — |
| `date` | date | — |
| `datetime` | timestamp, date | → date needs a timezone, see below |
| `picklist` | enum, string | enum requires a `value_map` — see 5.3 |
| `multipicklist` | string | semicolon-delimited; enum not allowed |
| `reference` | id_ref | or traverse one more hop to a real field |
| `id` | id_ref, string | — |
| `combobox`, `anyType` | string | low confidence; flag in the UI |
| `base64`, `address`, `encryptedstring` | — | never mappable |

Two cases that need a decision at map time, not a silent default:

- **`currency` → money.** If the org has multi-currency, the amount is meaningless without
  `CurrencyIsoCode` from the same record. The mapping captures a companion currency field;
  if the org is single-currency, the org default is recorded once on the connection.
  Architecture §10 requires `amount_usd` + `fx_rate_date`, so conversion happens on our side
  after the read.
- **`datetime` → `date`.** Truncating in UTC shifts renewals near midnight by a day — the
  exact failure the schema doc §10 calls out. The mapping records which timezone to truncate
  in: the account's, the org's, or UTC.

### 5.3 Value mapping for enums

This is a second mapping layer and it is easy to miss. A customer's
`Customer_Segment__c` picklist holds `ENT / MM / SMB`; our `account.segment` enum is
`enterprise / mid_market / smb`. Mapping the *field* is not enough — the *values* must map
too.

```json
{ "ENT": "enterprise", "MM": "mid_market", "SMB": "smb" }
```

Stored in `field_mapping.value_map`. Rules, straight from schema doc §10:

- Unmapped source values map to `other`, they are never dropped.
- The raw value is preserved as `source_value` alongside the mapped one.
- A picklist value added in Salesforce after mapping appears in the next discovery as a
  **new unmapped value**, and the mapping goes `needs_review` — not broken, since scoring can
  continue with `other`.

The UI should pre-fill this with case-insensitive and prefix matching; most picklists map
themselves.

### 5.4 Null is unknown, never zero

Schema doc §10, and it carries all the way through: an unmapped or null field produces
`null`, which lowers `data_completeness_pct` on the score (Z3) rather than scoring as a
measured zero. The machinery for honest degradation already exists in the scoring contract —
this plan just has to not defeat it.

---

## 6. Reading data without storing it

At scoring time, for one tenant:

1. Load the tenant's active mappings, group by `source_object`.
2. For each object, build one query: the union of mapped `source_path`s plus `Id`.
3. Page through (2000 rows per page), project each row through the coercion rules into
   canonical shape **in memory**.
4. Hand the canonical records to the scoring feature builder.
5. **Persist only the derived output** — features, scores, bands, drivers. Discard the source
   rows.

A tenant with four mapped objects and 10,000 accounts costs roughly 20–50 Salesforce API
calls for a full pass.

**API limits belong to the customer, not us.** Salesforce request limits are per-org and
shared with everything else they run. A wasteful sync becomes their outage and they will
read it as Zeke breaking their Salesforce. Budget per connection, back off hard on
`REQUEST_LIMIT_EXCEEDED`, and surface consumption next to freshness in `data_source_sync`
(schema doc Z9).

---

## 7. API design

All under the tenant-scoped API, `integrations:manage` permission for writes and
`integrations:read` for reads, following the `@ApiRoute` / `RequirePermissions` conventions
in `DomainsController`.

### Connections

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `GET` | `/v1/integrations/connectors` | What can be connected, with the capabilities each offers |
| `POST` | `/v1/integrations/connections` | Start auth. Returns a Nango Connect session token for the frontend widget |
| `GET` | `/v1/integrations/connections` | List, with status and last error |
| `GET` | `/v1/integrations/connections/{id}` | One |
| `DELETE` | `/v1/integrations/connections/{id}` | Revoke. Cascades mappings to `broken` |
| `POST` | `/v1/integrations/connections/{id}/reauthorize` | New session token for an expired connection |

### Schema

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `POST` | `/v1/integrations/connections/{id}/schema/refresh` | Trigger discovery. Returns a snapshot id |
| `GET` | `/v1/integrations/connections/{id}/schema` | Snapshot status, counts, age |
| `GET` | `/v1/integrations/connections/{id}/schema/objects` | Paginated, `?search=` |
| `GET` | `/v1/integrations/connections/{id}/schema/objects/{apiName}/fields` | Paginated, `?search=` |

### Catalog and mapping

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `GET` | `/v1/field-catalog` | Zeke's 50 fields, grouped, with required flags. The left column of the UI |
| `GET` | `/v1/field-mappings` | Current mapping, joined to catalog and source labels |
| `GET` | `/v1/field-mappings/candidates?canonicalField=account.arr&connectionId=…` | **The dropdown.** Type-compatible source fields only, ranked, with a suggestion flagged |
| `PUT` | `/v1/field-mappings` | Bulk upsert. Whole-set save, so the UI never half-commits |
| `DELETE` | `/v1/field-mappings/{canonicalFieldKey}` | Unmap |
| `POST` | `/v1/field-mappings/validate` | Dry run: sample ~200 live records, report coercion failures and fill rate |
| `GET` | `/v1/field-mappings/readiness` | Can scoring run? What is missing? |

`readiness` is the endpoint the rest of the product asks:

```json
{
  "ready": false,
  "requiredTotal": 24,
  "requiredMapped": 18,
  "missing": [
    { "key": "account.renewal_date", "label": "Renewal Date", "reason": "unmapped" },
    { "key": "account.arr", "label": "ARR", "reason": "broken",
      "detail": "Field ARR__c no longer exists in Salesforce" }
  ],
  "optionalMapped": 19,
  "optionalTotal": 26,
  "connections": [{ "id": "…", "connectorKey": "crm-salesforce", "status": "active", "supplies": 30 }]
}
```

Scoring calls this before a run and refuses with a typed error listing `missing` — the same
payload the UI renders as "complete these 6 fields."

### Webhook

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `POST` | `/webhooks/nango` | On `webhook-ingress`. Auth success, token refresh failure, connection deleted |

Signature-verified. A refresh failure flips the connection to `reauthorize_required` and
readiness starts reporting it — this is how a silently-expired token becomes visible instead
of becoming a mysterious scoring failure three weeks later.

### 7.1 Auto-suggestion matters more than it sounds

Fifty dropdowns is a lot of work to hand a new customer on day one, and a half-finished
mapping means no score, which means no product. Pre-fill it:

1. **Connector defaults** — §7.4 of the architecture already reserves `FieldMappingDefaults`.
   Salesforce standard fields are predictable: `account.name` → `Account.Name`,
   `account.employee_count` → `Account.NumberOfEmployees`. Ship ~20 of these and they are
   right in nearly every org.
2. **Name similarity** on the rest — normalised edit distance over api name and label, so
   `ARR__c`, `Annual_ARR__c` and `Customer ARR` all surface for `account.arr`.
3. **Type compatibility** as a hard filter, always.

Target: the customer confirms 30 suggestions and hand-maps 20, not 50 from scratch.

---

## 8. Edge cases

### 8.1 A new required field is added to the catalog

The sharpest one. Fifty becomes fifty-five, one of the new ones is required, and **every
existing tenant is instantly not ready.** If readiness is a simple boolean, scoring stops for
all of them overnight.

Handling: `field_catalog.required_from` is a timestamp, not a boolean flag.

- New fields land as optional with `required_from` set to a future date.
- Readiness reports them as `upcoming` with the date, and does not block.
- Tenants and their CSM are notified at the point the field appears, not when it bites.
- On that date it becomes required and the scorecard degrades gracefully — `null`, lower
  `data_completeness_pct` — rather than refusing to score.

**Never let a catalog change stop an existing tenant's scoring on the day it ships.**

### 8.2 The customer's admin renames or deletes a mapped field

Found at the next discovery, or as a runtime error first. Mapping → `broken` with a readable
reason, readiness reports it, scoring treats the field as null. Never a 500.

### 8.3 Field-level security changes

The connected user loses read on `ARR__c`. It vanishes from the next `describe` and looks
identical to a deletion. Same handling — the reason text should say "no longer readable,
which may be a permission change" since the fix is different.

### 8.4 Token revoked, password changed, user deactivated

Nango returns 424. Connection → `reauthorize_required`, webhook arrives, readiness surfaces
it, the UI offers Reconnect. The mapping is untouched and survives reconnection.

### 8.5 Two connections claim the same canonical field

Prevented by `unique (tenant_id, canonical_field_key)`. The API returns a typed conflict
naming the current owner, and the UI offers to move it.

### 8.6 Org-shape surprises

| Case | Effect |
| ---- | ------ |
| **Person Accounts** enabled | `Account` carries contact fields; discovery must not assume the standard split |
| **Record Types** | Same object, different picklist values per record type. `value_map` must cover the union |
| **Multi-currency off** | No `CurrencyIsoCode` field at all. Record the org default on the connection |
| **Contacts to Multiple Accounts off** | `AccountContactRelation` does not exist |
| **Large orgs** | Thousands of fields. Search and paginate server-side; never ship the whole list to the browser |

### 8.7 Mapped but empty

A field maps cleanly and is null in 95% of records. Technically ready, practically useless.
The `validate` endpoint samples live records and reports fill rate, so this is caught during
onboarding rather than showing up as an inexplicably low `data_completeness_pct` later.

### 8.8 Schema drift between discovery and query

A field dropped between the snapshot and the scoring run gives `INVALID_FIELD`. Catch it,
mark the mapping broken, re-run the query without that field rather than failing the whole
pass for one column.

### 8.9 Disconnect and deletion

Disconnecting revokes at Nango, marks the connection disabled and its mappings broken.
Because no source records are stored, tenant deletion has almost nothing to purge — schema
snapshots and mappings. That is a real benefit of decision 2.1 and worth stating to
customers.

---

## 9. Build sequence

One PR each, in order. Each is independently reviewable and leaves the build green.

### Phase A — connection

| # | PR | Contents |
| - | -- | -------- |
| A1 | `feat/connection-auth-broker` | Cherry-pick the POC port + Nango adapter + `dev-static`. Already written and tested |
| A2 | `feat/integrations-connection-store` | `connection` table, migration with RLS, repository |
| A3 | `feat/integrations-connect-api` | Connect / list / revoke / reauthorize endpoints, Nango Connect session |
| A4 | `feat/nango-webhook` | Webhook route on `webhook-ingress`, signature verification, status transitions |

### Phase B — schema discovery

| # | PR | Contents |
| - | -- | -------- |
| B1 | `feat/schema-describe-capability` | `SourceSchemaDescriber` port in contracts |
| B2 | `feat/salesforce-describe-adapter` | Salesforce implementation + recorded-fixture tests |
| B3 | `feat/source-schema-store` | `schema_snapshot` / `source_object` / `source_field` + migration |
| B4 | `feat/schema-discovery-workflow` | Temporal workflow, refresh endpoint, snapshot retirement |
| B5 | `feat/schema-browse-api` | Paginated object and field endpoints with search |

### Phase C — catalog and mapping

| # | PR | Contents |
| - | -- | -------- |
| C1 | `feat/field-catalog` | Table, seed migration with the agreed ~50 fields, `GET /v1/field-catalog` |
| C2 | `feat/field-mapping-store` | `field_mapping` table + migration |
| C3 | `feat/type-coercion` | Compatibility matrix, coercion functions, value-map handling. Pure logic, heavily unit-tested |
| C4 | `feat/mapping-api` | Bulk upsert, delete, candidates with suggestions |
| C5 | `feat/mapping-readiness` | Readiness endpoint and the typed error scoring raises |
| C6 | `feat/mapping-validate` | Live sample dry-run and fill-rate report |

### Phase D — reading

| # | PR | Contents |
| - | -- | -------- |
| D1 | `feat/record-query-capability` | `RecordQuerier` port |
| D2 | `feat/salesforce-soql-builder` | Mapping → SOQL, paging, two-hop paths, `INVALID_FIELD` recovery |
| D3 | `feat/canonical-projection` | Rows → canonical records through the coercion rules |

### Phase E — handoff

| # | PR | Contents |
| - | -- | -------- |
| E1 | `feat/feature-provider` | What scoring calls to get canonical records for a tenant |

Phases A and B are independent of the catalog being finalised, so **work can start before
the other team settles the 50 fields.** Only C1 is blocked on that list.

---

## 10. What you need to do by hand

### 10.1 Salesforce — new developer org

Answer decision 2.2 first; these steps assume **classic Connected App**.

1. Setup → App Manager → **New Connected App** (not "New External Client App").
2. Enable OAuth Settings.
3. Callback URL: `https://api.nango.dev/oauth/callback`
4. Scopes: `api` (Manage user data via APIs), `refresh_token, offline_access`, and `id,
   profile, email` for the connecting user's identity.
5. ✅ **Require Proof Key for Code Exchange (PKCE)**
6. ✅ **Enable Client Credentials Flow** — off. Not used.
7. ✅ **Introspect All Tokens** — Nango needs it.
8. Save, wait ~10 minutes for propagation.
9. Manage → Edit Policies → **Refresh Token Policy: Refresh token is valid until revoked.**
   Anything else silently logs every customer out on a schedule.
10. Manage Consumer Details → copy **Consumer Key** and **Consumer Secret**.

> The Consumer Key and Secret go into **Nango only**. They never enter the Zeke codebase or
> its environment — the Nango config has exactly three settings (secret key, base URL,
> timeout). Worth saying explicitly since it is a reasonable thing to assume otherwise.

> Your developer org is a *registration home*, not a gateway. It is never in the data path
> and gives you no access to any customer's org. Each customer authorises their own org and
> gets their own `instance_url`.

### 10.2 Nango

1. Dashboard → Integrations → new integration, provider **Salesforce**, key `salesforce`.
   - A separate `salesforce-sandbox` integration is needed later for customers who want to
     connect a sandbox first — different Salesforce login host, so Nango treats it as a
     different provider.
2. Paste the Consumer Key and Secret from 10.1.
3. Scopes: `api refresh_token offline_access`
4. Settings → Webhooks → point at `https://<dev-host>/webhooks/nango`, copy the signing
   secret.
5. Environment Settings → copy the **Secret Key** (backend) and **Public Key** (frontend
   Connect widget).

### 10.3 AWS

1. `NANGO_SECRET_KEY` and `NANGO_WEBHOOK_SECRET` → Secrets Manager, in the path the task
   definition already reads.
2. `terraform apply` in `50-compute` for the IAM read on those two secrets.
3. Confirm `/webhooks/*` reaches `webhook-ingress` through Caddy.

### 10.4 Frontend

The Nango Connect widget needs the **public key** and the session token from
`POST /v1/integrations/connections`. Nothing else about Salesforce reaches the browser.

### 10.5 Rotate the POC Nango key

It was pasted in plaintext in an earlier conversation. Rotate before any of this goes live.

---

## 11. Open questions

1. **Decision 2.1** — may we store an account directory (external id, name, owner)? Sections
   6 and 9 assume yes.
2. **Decision 2.2** — classic Connected App or Packaged ECA?
3. **The 50 fields.** C1 is blocked until the list exists. Everything else is not.
4. **Who maps?** A customer admin during onboarding, or Zeke staff on their behalf? Changes
   the permission model and how much auto-suggestion has to carry.
5. **Re-discovery cadence.** On demand only, or a weekly schedule? A weekly pass is what
   turns a renamed field into a notification instead of a scoring failure.
6. **One Salesforce per tenant?** The schema assumes it (`unique (tenant_id, connector_key)`).
   Large customers with several orgs would need that relaxed, and it is much cheaper to
   decide now than later.
