---
marp: true
theme: default
paginate: true
size: 16:9
header: 'yProHub — Architecture & Money Flow'
footer: 'Internal engineering walkthrough'
style: |
  section { font-size: 22px; }
  h1 { color: #1a3a6b; }
  h2 { color: #1a3a6b; }
  code { font-size: 0.85em; }
  table { font-size: 0.8em; }
  pre { font-size: 0.7em; }
---

<!-- _class: lead -->

# yProHub
## Architecture & Money-Flow Walkthrough

A professional-services marketplace — from system layout
to the job/bid lifecycle to the payment split & payout path.

---

# Agenda

1. **System overview** — the four components
2. **High-level architecture** — clients → API → data & integrations
3. **Backend layering** & plugin abstractions
4. **Core domain model**
5. **Job / bid lifecycle** — three state machines
6. **Payment split** — how the money is divided
7. **Partial payments** — proration & payment requests
8. **Payout path** — money out to the pro
9. **Recently closed & remaining gaps**

---

# 1. System overview — four components

| Component | Stack | Role |
|-----------|-------|------|
| **prohub-ui** | Angular 20, TS 5.9 strict, Material, SCSS | The actual app — all UI & business logic |
| **ProHubAPI** | ASP.NET Core 8, C# 13, EF Core 8, SQL Server | REST API + SignalR hub + integrations |
| **prohub-android** | Capacitor 8 WebView | Packages the Angular build (`www/`) as an APK |
| **prohub-ios** | Capacitor 8 WKWebView | Same, as an IPA (macOS to build) |

> **Key point:** Android/iOS are *thin wrappers* serving a **static snapshot** of the Angular build in `www/`.
> No native code beyond an empty `BridgeActivity`. UI changes need `npm run web:sync` to reach the apps.

---

# 2. High-level architecture

```
                          ┌─────────────────────────────┐
                          │   prohub-ui (Angular 20)     │
                          │   standalone comps, JWT, i18n│
                          └──────────────┬──────────────┘
                            built output (dist → www/)
              ┌────────────────────┼────────────────────┐
       ┌──────▼──────┐   ┌─────────▼────────┐   ┌────────▼───────┐
       │ Web browser │   │ prohub-android   │   │  prohub-ios    │
       │ :4200       │   │ Capacitor WebView│   │ Capacitor WKWeb│
       └──────┬──────┘   └────────┬─────────┘   └────────┬───────┘
              │   HTTPS/JSON (JWT Bearer) + SignalR (WebSocket)
              └───────────────────┼──────────────────────┘
                    ┌─────────────▼───────────────┐
                    │  ProHubAPI (ASP.NET Core 8)  │
                    │  Controllers → Services → EF │
                    └─────────────┬───────────────┘
      ┌──────────┬────────────────┼───────────┬──────────────┐
 ┌────▼────┐┌────▼────┐    ┌──────▼───┐  ┌─────▼────┐  ┌──────▼─────┐
 │SQL Server││Razorpay │    │  Msg91   │  │   SMTP   │  │ Nominatim  │
 │(EF Core) ││pay+payout│   │   SMS    │  │  email   │  │ geocoding  │
 └─────────┘└─────────┘    └──────────┘  └──────────┘  └────────────┘
```

---

# 3. Backend layering

```
Controllers (HTTP)  →  Services (business logic)  →  ApplicationDbContext (EF Core)  →  SQL Server
      │                       │
      │                       └── Plugin abstractions (Services/Abstractions/):
      │                             IPaymentProvider    → Razorpay / Mock
      │                             INotificationChannel → Msg91 SMS
      │                             IInsuranceProvider   → PlaceholderInsuranceProvider ("yProHub Protect")
      │                             IFileStorageService  → LocalFileStorageService (wwwroot/uploads)
      │
      ├── Hosted services: BidExpirationService (periodic sweep, 15 min)
      └── SignalR NotificationHub (/hubs/notifications) — real-time notifications
```

- **Auth:** JWT Bearer; every token carries a `jti`. Logout revokes it via `RevokedTokens`; an
  `OnTokenValidated` hook rejects revoked tokens. Azure MSAL also supported.
- **Migrations:** auto-applied on startup (`MigrateAsync()`); seed data runs on boot.
- **CORS:** dev allows any origin (restrict in prod).

---

# 4. Core domain model

- **User** (consumer) posts jobs · **Pro** offers services — separate entities, separate verification flows.
- Job lifecycle:
  **posted → pros submit `JobBid`s → user accepts one (`AssignedProId`) → payment → work → `JobCompletion` → payout**

**Supporting entities**

| Entity | Purpose |
|--------|---------|
| `Service` / `ServiceCategory` | Catalogue of offerings |
| `Message` / `MessageIndex` | Conversation pairs |
| `Payment` / `PaymentRequest` / `Payout` | Money in, nudges, money out |
| `JobCompletion` | Completion proof (notes + uploaded photos), verify/dispute |
| `JobInsurance` | Policy record — consumer self-service purchase or admin entry |
| `ServiceArea` | Country → State → District → PIN hierarchy |
| `PaymentMethod` | Pro's bank / UPI details for payouts |
| Uploaded files | `wwwroot/uploads/files` via `FilesController` + `IFileStorageService` |

---

# 5. Job / bid lifecycle — actors

- **User (consumer)** — posts jobs, accepts/rejects bids, pays, verifies or disputes completion.
- **Pro** — browses available jobs, submits bids, confirms & starts work, submits completion.
- **Admin** — resolves disputes, retries failed payouts, manages insurance.

The lifecycle is tracked by **three status fields moving in lockstep**:
`Job.Status`, `JobBid.Status`, `JobCompletion.Status`.

---

# 5. Job.Status state machine

```
 ┌──────┐  accept bid  ┌──────────────┐  pay  ┌──────────────┐ pro    ┌─────────────┐
 │ Open │─────────────▶│ Bid Accepted │──────▶│ Payment Made │───────▶│ In Progress │
 └──┬───┘              └──────────────┘       └──────────────┘confirm └──────┬──────┘
    │ cancel (only before Payment Made)                                       │ submit
    ▼                                                                         ▼ completion
┌───────────┐                                                  ┌──────────────────────┐
│ Cancelled │                                                  │ Completion Submitted │
└───────────┘                                                  └───────────┬──────────┘
                                        consumer verifies │                │ disputes
                                                           ▼                ▼ (Disputed)
                                                     ┌───────────┐   admin resolves:
                                                     │ Completed │   complete │ refund
                                                     └─────┬─────┘   (refund reopens job)
                                                           ▼
                                                     payout to pro
```

---

# 5. Lifecycle — step by step

1. **Post** — `POST /api/jobs` → `Status=Open`.
2. **Bid** — `POST /jobs/{id}/bid`: email-verified pro, one bid/job, `ExpiresAt` default **+5 days**.
3. **Owner decides** — **Accept** (auto-rejects all other pending bids, sets `AssignedProId`, `Bid Accepted`) · **Reject** · pro may **Withdraw**.
4. **Pay** — Razorpay capture → `Bid Accepted → Payment Made`. Job can no longer be cancelled.
5. **Pro confirms** — `pro-confirm` → `In Progress`.
6. **Submit completion** — `PUT /jobs/{id}/complete` → `JobCompletion{Submitted}` (notes + optional uploaded photos), job `Completion Submitted`. Re-submit only if previously `Disputed`.
7. **Verify / Dispute** — verify → `Completed` **+ triggers payout**; dispute → `Disputed` (admin resolves).

| Field | Values |
|-------|--------|
| `Job.Status` | Open → Bid Accepted → Payment Made → In Progress → Completion Submitted → Completed / Cancelled |
| `JobBid.Status` | Pending → Accepted / Rejected / Withdrawn / **Expired** |
| `JobCompletion.Status` | Submitted → Verified / Disputed / Refunded |

---

# 6. Payment split — `RateSplitService`

**Config source:** `commission.*` keys in `AppSettings` (admin-editable), with defaults:

| Key | Default | Meaning |
|-----|---------|---------|
| `commission.user_percent` | 10% | Fee **added on top** of the bid, charged to consumer |
| `commission.pro_percent` | 10% | Fee **deducted** from the pro's payout |
| `commission.gst_percent` | 18% | GST on the user commission |
| `commission.min_fee` | ₹10 | Floor on user commission |
| `commission.max_percent` | 20% | Cap on user commission |

```
userCommission      = clamp(bid × 10%, min ₹10, max bid×20%)
gstOnUserCommission = userCommission × 18%
totalAmountUserPays = bid + userCommission + gstOnUserCommission   ← consumer charged
proDeduction        = bid × 10%
proPayout           = bid − proDeduction                           ← pro receives
platformEarnings    = userCommission + proDeduction
```

---

# 6. Split — worked example (bid ₹1,000)

| Line | Amount |
|------|--------|
| Bid (principal) | ₹1,000 |
| + User commission (10%) | ₹100 |
| + GST on commission (18%) | ₹18 |
| **Consumer pays** | **₹1,118** |
| − Pro deduction (10%) | ₹100 |
| **Pro receives** | **₹900** |
| **Platform keeps** | **₹200** (+ ₹18 GST pass-through) |

- Consumer effectively pays **+11.8%**; pro nets **90%**.
- Two-sided commission: charged to consumer *and* deducted from pro.

---

# 7. Partial payments — proration

Jobs can be paid in slices. Always compute the **full-bid** split, then scale to the slice:

```
factor = principal / fullBidAmount
each fee (userCommission, GST, proDeduction) × factor, then rounded
proPayout = principal − scaledProDeduction
```

- Guarantees a series of partials **sums exactly** to the full-bid split (no rounding drift).
- Each `Payment` row stores `PrincipalAmount`, `Amount` (total charged), `PlatformFee`, `ProPayout`.
- Accounting is **principal-based**:
  `alreadyPaid = Σ PrincipalAmount of Completed payments (≤ bid)` · `remaining = bid − alreadyPaid`.

---

# 7. Payment request flow — the "nudge"

A `PaymentRequest` is the assigned pro asking the consumer to pay a specific amount. It moves **no money** — it sets an expectation + a floor. **One `Pending` request per job at a time.**

| Type | RequestedAmount | Behavior |
|------|-----------------|----------|
| **None** | 0 | Created already `Fulfilled`; job jumps to `Payment Made` — pro waives upfront pay so work can begin |
| **Full** | `remaining` | Blocked if already fully paid |
| **Partial** | requested | Must be > 0 and ≤ remaining; `MinPercent` = min fraction payable per slice |

**Floor enforced at order creation:**
`floor = min(RequestedAmount × MinPercent/100, remaining)` → payment must be ≥ floor.

---

# 7. Payment request — end to end

```
Pro: POST /request  (Partial ₹5000, MinPercent 40%)
        ▼
   PaymentRequest{Pending}  ──notify──▶ consumer
        │
Consumer: GET /job/{id}/summary → ActiveRequest, MinAmount ₹2000, remaining
        │
Consumer: POST /create-order (principal ₹2000)   floor 2000 ≥ 5000×40% ✓
        ▼  prorate fees to slice → Razorpay order → Payment{Pending}
POST /verify (signature ok)
        ├─ Payment → Completed  (PrincipalAmount ₹2000 counted)
        ├─ PaymentRequest → Fulfilled
        └─ Job: Bid Accepted → Payment Made
        │
   remaining ₹3000 → pro may raise a NEW request for the balance
```

---

# 8. Payout path — money out (`PayoutService`)

**Trigger:** consumer verifies completion → find `Completed` payment → `CreateAndProcessPayoutAsync(payment.ProPayout)`.

```
1. Idempotency: one Payout per PaymentId
2. Insert Payout { Pending }
3. Look up pro's default PaymentMethod
   └─ none / incomplete → stays Pending, notify pro to add bank/UPI
4. SendToRazorpayAsync:
     ensure RazorpayContactId      → fail → Failed
     ensure RazorpayFundAccountId  → fail → Failed
        UPI  → mode "UPI"   ·   bank → mode "IMPS"
     InitiatePayoutAsync → id → Payout = Processing
```

**Async finalization via webhook:** `payout.processed → Processed` · `payout.failed → Failed`.
**Recovery:** pro adds bank details later, or admin `POST /payouts/{id}/retry`.

---

# 8. Full money path

```
bid ₹1000 ──prorate──▶ consumer charged ₹1118 ──Razorpay──▶ payment.captured
                                                                 │
                                              Payment=Completed · Job=Payment Made
                              (platform holds funds; ProPayout ₹900 stored on Payment)
                                                                 │
                    pro works → submits completion → consumer VERIFIES
                                                                 │
                                          CreateAndProcessPayoutAsync(₹900)
                                     ┌───────────────────────────┴───────────────┐
                            no bank details                              SendToRazorpay
                                  │                                            │
                          Payout Pending (notify)                     contact→fund acct→initiate
                                  │                                            │
                             [admin retry]                             Payout Processing
                                                                               │
                                        webhook payout.processed ─▶ Processed ✅ pro paid ₹900
                                        webhook payout.failed    ─▶ Failed    ↺ admin retry
```

---

# 9. Recently closed & remaining gaps

**Recently implemented**
- **`BidExpirationService`** — periodic background sweep marks overdue `Pending` bids `Expired` (accept-time check stays as a safety net).
- **`IInsuranceProvider`** → `PlaceholderInsuranceProvider` — coverage tiers priced off the job budget; **consumer self-service purchase** (`/insurance/options`, `/insurance/purchase`) + a "Job Protection" UI section.
- **`IFileStorageService`** → `LocalFileStorageService` + `FilesController` upload/fetch; **completion photos** persisted on `JobCompletion`; checkout gains a **saved payment-method picker**.

**Still open**
- **Insurance provider is self-backed** — placeholder issues policies without charging a premium or calling a real underwriter.
- **File storage is local disk** — swap `LocalFileStorageService` for Blob/S3 in production.
- **Verification SMS** is console-only — bypasses the working `Msg91SmsChannel`.
- **Email** sent via inline `SmtpClient` in `VerificationService` — no dedicated channel.

---

# 9. Gaps worth flagging (money flow)

- **Payout status depends on webhooks** — without `payout.processed/failed` (and `WebhookSecret` set),
  payouts stay stuck in `Processing`. No polling reconciliation job.
- **GST is collected but not persisted as its own column** — folded into `Payment.Amount`;
  reporting must recompute from `PlatformFee`.
- **Payout amount snapshotted at order time** — later commission changes don't affect paid orders (correct for partials).
- **"None" request → job can reach `Completed` with no captured payment** → no payout (guarded). Confirm this is the intended rule.
- **No `PaymentRequestId` FK on `Payment`** — fulfillment matches the newest `Pending` request positionally, not referentially.

---

<!-- _class: lead -->

# Summary

- **Thin mobile wrappers** over one Angular app; **layered ASP.NET Core** API with pluggable providers.
- **Three synchronized state machines** drive the job → bid → completion lifecycle.
- **Two-sided commission** split, prorated for partial payments; **payout** triggered on completion-verify, finalized by Razorpay webhooks.
- **Recently shipped:** bid-expiration sweep, file uploads (completion photos), consumer insurance purchase, checkout payment-method picker.
- **Highest-impact next steps:** route OTP SMS through Msg91, add payout reconciliation polling, move file storage to cloud (Blob/S3), back insurance with a real underwriter.

## Questions?
