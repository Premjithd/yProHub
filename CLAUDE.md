# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

yProHub is a professional services marketplace with four main components:
- **prohub-ui**: Angular 20 frontend application
- **ProHubAPI/ServiceProviderAPI**: ASP.NET Core 8 backend API
- **prohub-android**: Capacitor 8 Android wrapper (packages the Angular app as an APK)
- **prohub-ios**: Capacitor 8 iOS wrapper (packages the Angular app as an IPA — requires macOS to build)

The system connects users who post jobs with professionals (pros) who offer services, featuring authentication, verification, job bidding, messaging, and payment integration (Razorpay).

## Frontend (prohub-ui)

### Technology Stack
- **Framework**: Angular 20.2.x (standalone components and modules)
- **Language**: TypeScript 5.9 (strict mode enabled)
- **Styling**: SCSS with Angular Material Design
- **Authentication**: JWT-based with Azure MSAL (Microsoft authentication) support
- **Testing**: Karma/Jasmine with Chrome launcher
- **Build**: Angular CLI 20.2.2

### Commands

```bash
cd prohub-ui

npm install           # Install dependencies
npm start             # Dev server at http://localhost:4200
npm run build         # Production build
npm run watch         # Watch mode (rebuilds on changes)
npm test              # Run unit tests
ng test --code-coverage
```

### Architecture

**Routing**: The app uses a `MainLayout` wrapper for all authenticated routes. `accept-admin-invite` is outside the layout for public access. Most features lazy-load via `loadChildren`; root features use `loadComponent`.

**Auth**: Supports JWT and Azure MSAL. Auth state managed by `AuthService`; tokens stored in localStorage. HTTP interceptors attach JWT to all API requests.

**Forms**: Mix of template-driven and reactive forms with Angular Material inputs. Address autofill via backend proxy to Nominatim API.

**Key directories**:
- `src/app/auth/` — login, registration, verification flows
- `src/app/core/` — singleton services, guards, interceptors, shared models
- `src/app/features/` — lazy-loaded feature modules (home, profile, post-job, messages, payments, admin, etc.)
- `src/app/layout/` — main layout, navbar, footer, sidebar
- `src/app/services/` — job, material, payment services
- `src/environments/` — API URL config (dev: `http://localhost:5001/api`)

## Backend (ProHubAPI/ServiceProviderAPI)

### Technology Stack
- **Framework**: ASP.NET Core 8
- **Language**: C# 13 with nullable reference types and implicit usings
- **Database**: SQL Server + Entity Framework Core 8
- **Authentication**: JWT Bearer tokens
- **Payments**: Razorpay
- **SMS**: Msg91; **Email**: SMTP

### Commands

```bash
cd ProHubAPI/ServiceProviderAPI

dotnet restore
dotnet run                              # Dev server at https://localhost:7042
dotnet watch run                        # Watch mode
dotnet build --configuration Release
dotnet ef database update               # Apply pending migrations
dotnet ef migrations add MigrationName  # Add new migration (must build first)
dotnet test
```

Swagger UI: `https://localhost:7042/swagger`

> **Migration tip**: Always run `dotnet build` before `dotnet ef migrations add` — using `--no-build` with an outdated DLL generates empty Up/Down methods.

### Architecture

**Layered**: Controllers → Services → EF Core DbContext (`ApplicationDbContext`)

**Plugin abstractions** in `Services/Abstractions/`:
- `IPaymentProvider` — Razorpay implementation provided
- `INotificationChannel` — SMTP and Msg91 implementations
- `IInsuranceProvider`, `IFileStorageService` — placeholders for future

**Verification**: 6-digit codes with 15-min expiration; separate flows for Users and Professionals. Codes marked used after verification.

**Messaging**: `MessageIndex` tracks user-pair conversations; `Message` entities link to index.

**Email branding**: All outbound emails share one branded HTML shell built by `EmailTemplates` (`Services/EmailTemplates.cs`):
- `Wrap(innerHtml, tagline?)` renders the shell — centered yProHub logo header, a tagline line (defaults to "This is an official message from yProHub."), the inner content, and an automated-message/copyright footer.
- The **logo is embedded inline via a `cid:` content-id** (`BuildBrandedView` attaches `EmailAssets/logo-512.png` as a `LinkedResource`, `ContentId = "yprohub-logo"`), not referenced by URL — so it renders without depending on a publicly reachable image or the client un-blocking remote images. The `.csproj` copies `EmailAssets/**` to the build output (`AppContext.BaseDirectory/EmailAssets`); `BuildBrandedView` guards with `File.Exists`, so a missing asset degrades gracefully (email still sends, no logo).
- `TextToHtml(text)` converts plain-text notification bodies (escape + `nl2br`) for the shell.
- **Senders all route through the shell**: `SmtpEmailService.SendEmailAsync` (admin invite, team invite, and every `NotifyAsync` via `SmtpEmailChannel`) wraps the body in `BuildBrandedView`; `VerificationService` uses its own `SmtpClient` but the same `EmailTemplates` helper. So verification, password reset, admin/team invites, and all job notifications are consistently branded. Transactional emails with an action link (reset, team/admin invite) use a styled button (`#1d4ed8`) plus a copy-paste fallback URL; pass HTML (not plain text) to `SendEmailAsync` to get a real button rather than an escaped link.
- **SMS** can't carry a logo — branding there is just a `yProHub:` text prefix (see `NotificationService.BuildSmsText` and `VerificationService.SendVerificationSms`) plus the DLT-registered 6-char sender ID (`SMS:MSG91:SenderId`, e.g. `PROHUB`).

**Auth / Logout**: JWT tokens include a `jti` (JWT ID) claim. On logout, the `jti` is written to the `RevokedTokens` table (`ITokenBlacklistService`). An `OnTokenValidated` hook in `Program.cs` rejects any request whose token `jti` is in that table. The Angular `Auth.logoutOnServer()` calls `POST /api/auth/logout` before clearing localStorage — the client-side clear always happens even if the server call fails.

**Migrations**: Auto-applied on startup via `context.Database.MigrateAsync()`. Key migrations: `InitialCreate`, `AddUserTypeAndVerificationCodes`, `AddJobTable`, `AddAddressFieldsToUser/Pro`, `AddServiceCategoryTable`, `AddRevokedTokens`.

**Key directories**:
- `Controllers/` — Auth, Users, Pros, Jobs, JobCompletion, JobInsurance, Notifications, Messages, Payments, Materials, Services, Admin, Verification, Address
- `Models/` — EF entities (User, Pro, Job, JobBid, JobCompletion, JobInsurance, JobNotification, Service, Message, Payment, RevokedToken, etc.)
- `DTOs/` — request/response shapes
- `Services/` — business logic, JwtService, TokenBlacklistService, VerificationService, RateSplitService, SeedDataService

### Configuration

`appsettings.Development.json` overrides for local dev. Key sections in `appsettings.json`:
```json
{
  "ConnectionStrings": { "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ProhubDB;..." },
  "Jwt": { "Key": "...", "Issuer": "https://localhost:7042", "Audience": "https://localhost:7042" },
  "Email": { "SmtpServer": "...", "Port": 587, "Username": "...", "Password": "...", "From": "..." },
  "Payment": { "Razorpay": { "KeyId": "...", "KeySecret": "..." } }
}
```

## Payment Split & GST (India) — model + decision notes

How a job's money is split between **user (consumer)**, **pro (professional)**, and **platform**. Computed in `Services/RateSplitService.cs` (`CalculateSplitAsync` for the full bid; `ProrateForPrincipal` scales a full split down to a partial payment so a series of partials sums exactly to the full). Surfaced on `POST /api/payments/create-order` and persisted on the `Payment` row (`PlatformFee`, `ProPayout`).

### Current model (facilitation / two-sided commission)
The platform takes a cut from **both** sides of the agreed bid:
- **User pays** = bid + user commission + GST on that commission  → ~111.8% of bid at defaults
- **Pro receives** = bid − pro commission − GST on pro commission  → ~88.2% of bid at defaults
- **Platform earns** = user commission + pro commission (the GST is a pass-through to the government, not platform profit)

Config lives in the `AppSettings` table (admin-editable via `GET/PUT /api/admin/commission-config`); defaults in `RateSplitService.Defaults`:

| Key | Default | Meaning |
|---|---|---|
| `commission.user_percent` | 10 | user-side commission % (added on top of bid) |
| `commission.pro_percent` | 10 | pro-side commission % (deducted from payout) |
| `commission.gst_percent` | 18 | GST rate applied to commission(s) |
| `commission.min_fee` | 10 | floor (₹) on the user commission |
| `commission.max_percent` | 20 | cap (% of bid) on the user commission |
| `commission.pro_gst_applies` | 1 | **charge GST on the pro-side commission** (1=on, 0=off) |

Worked example (bid ₹1,500, defaults, pro GST on): user pays **₹1,677**, pro payout **₹1,323**, platform net commission **₹300**, GST remitted **₹54** (₹27 user-side + ₹27 pro-side). Admin UI: commission-config card in `admin-users` has a live preview + a "Charge GST on Pro Commission" toggle (button-toggle with `(click)`, not a native checkbox with `[(ngModel)]` — the app is zoneless and checkbox two-way binding doesn't reflect reliably).

### Why GST on the pro commission (decision)
The platform's commission is its **own taxable supply of service to the pro** (listing/lead-gen/payment handling), so 18% GST applies to it — separately from, and in addition to, any GST on the underlying job. This is a **distinct supply** from the job itself, so it holds in **both** GST regimes below. Precedent: a Swiggy/Zomato payout carries **5% GST (§9(5)) on the food value + 18% GST on the platform commission** simultaneously. Hence `pro_gst_applies` defaults **on**. Caveat: if the pro is **unregistered** (common under §9(5), see below), they **cannot claim ITC** on that commission GST, so it becomes a real cost that lowers their take-home.

### Open tax question — which GST regime applies (CONFIRM WITH A CA before launch)
yProHub's categories (plumbing, carpentry, painting, cleaning) are **"housekeeping"**, which is a **§9(5) notified service**. Two mutually-relevant regimes:

- **Model A — facilitation / marketplace** (what the code implements today): the **pro** is the supplier and handles GST on the job; the platform only charges **commission + 18% GST**, and additionally must collect **§52 TCS @ 0.5%** (reduced from 1% on 2024-07-10) on net supply value, filed in GSTR-8. *TCS is NOT modeled in code yet.*
- **Model B — §9(5) deemed supplier** (likely applies to home services): the **platform is treated as the supplier** and **pays GST on the full job value** to the government in **cash (no ITC)**; the **pro is exempt from GST registration even above the threshold**; **no §52 TCS**. The commission + 18% GST (Model A's pro-commission line) **still applies on top** as a separate supply.

So the pro-commission GST feature is correct in both; the **unresolved item** is whether the platform additionally owes **§9(5) GST on the full job value**. If so, `RateSplitService` needs a config-gated `gstMode` (`facilitation` | `deemed_supplier_9_5`) adding a platform full-value GST line. Also confirm with the CA: (a) the exact **GST rate per service category** (home-repair generally 18%; some §9(5) categories are 5% without ITC), and (b) whether both the §9(5) full-value GST and the commission GST co-apply.

### Market benchmark
Indian service marketplaces take **~15–35%**. Urban Company's all-in take is **~28%** of the pro's gross (commission + fees); customer pays service + convenience fee + GST; pro is paid after the customer rates the job. yProHub's default **20%** gross take (10% + 10%) is on the lower end.

## Android (prohub-android)

### Technology Stack
- **Type**: Capacitor 8 hybrid app — Android WebView wrapper around the Angular web app
- **App ID**: com.yprohub.android
- **Min SDK**: 24 (Android 7.0); **Target/Compile SDK**: 36 (Android 15)
- **Build**: Gradle 8.13.0, AGP 8.13.0, Java 21

### Architecture
The native layer is intentionally minimal — a single `MainActivity extends BridgeActivity`. All UI and business logic live in the Angular web app. Capacitor bridges native device APIs to the web layer.

**No native Kotlin/Java code** beyond the empty activity shell. UI is served from `www/` (built Angular output).

### UI changes and the Android app

The Android app serves a **static snapshot** of the Angular build stored in `www/`. Changes to `prohub-ui` do **not** apply automatically — you must manually rebuild and sync after any UI change:

```powershell
cd prohub-android
npm run web:sync   # rebuild Angular → copy to www/ → patch API URLs → cap sync
```

Then rebuild the APK in Android Studio. The web app (`localhost:4200`) and Android app are fully independent — the web app uses Angular's live dev server, Android uses the pre-built `www/` snapshot.

### Commands

```powershell
cd prohub-android

npm install                # Install Capacitor dependencies

# Full build pipeline (run in order):
npm run web:build          # Build Angular app (dev config)
npm run web:copy           # Copy Angular dist → www/
npm run web:patch-android-api  # Rewrite localhost URLs → 10.0.2.2:5101 for emulator
npm run cap:sync           # Sync www/ + plugins to Android native project

# Shortcut: all four steps at once
npm run web:sync

# Open in Android Studio
npm run cap:open           # Opens android/ in Android Studio
```

### API URL Patching
`web:patch-android-api` rewrites all occurrences of `localhost:7042` and `localhost:5001` to `10.0.2.2:5101` — the standard Android emulator address for the host machine. The backend must be running on port 5101 (or update the patch script) when testing on an emulator.

### Key Files
- `capacitor.config.json` — appId, webDir (`www/`), HTTP scheme settings
- `android/variables.gradle` — centralized SDK/library version pins
- `android/app/build.gradle` — app module config, dependencies
- `android/app/src/main/java/com/yprohub/android/MainActivity.java` — empty BridgeActivity shell
- `android/app/src/main/AndroidManifest.xml` — INTERNET permission, cleartext traffic allowed (dev)

## iOS (prohub-ios)

### Technology Stack
- **Type**: Capacitor 8 hybrid app — iOS WKWebView wrapper around the Angular web app
- **App ID**: com.yprohub.ios
- **Requires**: macOS + Xcode + CocoaPods to generate and build the native project

### Architecture
Mirrors `prohub-android` — minimal native shell, all UI/logic in the Angular web app. Unlike Android, iOS simulator can reach `localhost` directly so no API URL patching is needed.

### Commands (run from prohub-ios/)

```powershell
npm install        # Install Capacitor dependencies

# Sync web app into iOS project (run after any UI change)
npm run web:sync   # Angular build → copy to www/ → cap sync

# macOS only — complete native setup
npx cap add ios    # Generate ios/ Xcode project (first time, on macOS)
npm run cap:open   # Open in Xcode
```

> **macOS required**: `npx cap add ios` calls CocoaPods and Xcode tooling. The `ios/` native project directory must be generated on a Mac before building.

### Key Files
- `capacitor.config.json` — appId, webDir (`www/`)
- `package.json` — build pipeline scripts
- `ios/` — Xcode project (generated on macOS via `npx cap add ios`, not committed until then)

### UI changes and the iOS app
Same rule as Android — `www/` is a static snapshot. Run `npm run web:sync` from `prohub-ios/` after any UI changes, then rebuild in Xcode.

## Running Both Apps Locally

1. **Backend**: `cd ProHubAPI/ServiceProviderAPI && dotnet watch run` — runs at `https://localhost:7042`; migrations and seed data apply automatically on startup
2. **Frontend**: `cd prohub-ui && npm start` — runs at `http://localhost:4200`, configured to hit `http://localhost:5001/api` (update `src/environments/environment.ts` if backend port differs)
3. **Android** (emulator): ensure backend is on port 5101, then `cd prohub-android && npm run web:sync` then open in Android Studio and run on emulator
4. **iOS** (simulator, macOS only): `cd prohub-ios && npm run web:sync` then open in Xcode via `npm run cap:open` and run on simulator — no URL patching needed, simulator reaches `localhost` directly

## Frontend Architecture Notes

**Design tokens**: Global CSS custom properties defined in `src/styles.scss` — use `var(--color-primary)`, `var(--text-h1)`, etc. rather than hard-coded values.

**Bottom navigation**: `src/app/layout/bottom-nav/` — mobile-only (`display: none` on desktop), role-aware tabs. `--bottom-nav-height` CSS variable (60px on mobile, 0px on desktop) controls page padding.

**Snackbar panels**: `.snack-info`, `.snack-success`, `.snack-error` defined globally in `styles.scss`. Pass via `panelClass` on `MatSnackBar.open()`.

**Angular build budget**: `anyComponentStyle.maximumError` is set to `30kB` in `angular.json` (raised from default 20kB due to large detail-page stylesheets).

**Services browse search** (`features/services/services`): the search box filters on a **text query** plus an optional **PIN area** filter. Both are wired to `GET /api/services` (`BrowseServicesService`):
- `search` — tokenized on whitespace; a service matches if **any** term appears (OR semantics) as a **substring** in name/description/business name/pro name. So "plumb paint" matches both *Plumbing* and *Painting*, and partial words work ("plumb" → "plumber"). Implemented in `ServicesController.GetServices`.
- `pin` — exact match against the pro's `Address.ZipPostalCode`, layered on top of the text search to scope results to an area. (This is the pro's own postal code, not the admin `ServiceArea` reference table — there is no pro↔ServiceArea coverage link.)
- The home "Get Quotes" box always routes to `/services?search=&pin=` (for both logged-in and logged-out users); the services page reads those query params on load and applies the filter. Opening a service from the browse list requires login (redirects to `/auth/login?returnUrl=`).
- The sidebar "Services" link and the "What do you need help with?" category section are hidden for pros / hidden outright respectively — categories still load so deep-links (`?category=`) keep working.

## Important Notes

- **CORS**: Dev allows any origin (`AllowAnyOrigin()`). Restrict in production.
- **JWT secret**: Change the example value in `appsettings.json` for production.
- **JWT tokens**: Every token contains a `jti` claim. Logout revokes it server-side via the `RevokedTokens` table.
- **Email/SMS**: Console-only in dev; configure SMTP/Msg91 for production.
- **Address autofill**: Uses Nominatim (OpenStreetMap) via backend proxy. See `ADDRESS_AUTOFILL_GUIDE.md` for details.
- **Razorpay**: Test keys in `appsettings.Development.json`; swap for live keys in production.
- **Brand name**: Always "yProHub" (lowercase 'y'). The `y` prefix is intentional — never write "ProHub" or "YProHub".
