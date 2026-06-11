# yProHub

A professional services marketplace connecting users who post jobs with verified professionals (pros) who offer services. The platform features authentication, two-step pro registration, job bidding, messaging, payments (Razorpay/UPI), notifications, refund/dispute management, and service area filtering.

## Repository Structure

| Directory | Description |
|---|---|
| `prohub-ui/` | Angular 20 frontend (web app) |
| `ProHubAPI/ServiceProviderAPI/` | ASP.NET Core 8 backend API |
| `prohub-android/` | Capacitor 8 Android APK wrapper |
| `prohub-ios/` | Capacitor 8 iOS IPA wrapper |

Each subdirectory is a git submodule. After cloning, run:

```bash
git submodule update --init --recursive
```

---

## Quick Start

### 1. Backend

```bash
cd ProHubAPI/ServiceProviderAPI
dotnet restore
dotnet watch run          # https://localhost:7042 — migrations auto-apply on startup
```

Swagger UI: `https://localhost:7042/swagger`

### 2. Frontend

```bash
cd prohub-ui
npm install
npm start                 # http://localhost:4200
```

> The frontend expects the API at `http://localhost:5001/api`. Update `src/environments/environment.ts` if the backend port differs.

### 3. Android (emulator)

```powershell
cd prohub-android
npm install
npm run web:sync          # Build Angular → copy to www/ → patch API URLs → cap sync
# Then open android/ in Android Studio and run on emulator
```

### 4. iOS (simulator — macOS only)

```bash
cd prohub-ios
npm install
npm run web:sync          # Build Angular → copy to www/ → cap sync
npm run cap:open          # Open in Xcode
```

---

## Features

### Authentication & Registration
- Email/password login for users and professionals
- Two-step pro registration: account creation then profile completion
- Email and phone verification (6-digit codes, 15-min expiry, one-time use)
- JWT-based auth with `jti` claim; logout revokes token server-side via `RevokedTokens` table
- Azure MSAL (Microsoft) authentication support
- Protected routes with role-aware auth guards

### User & Professional Profiles
- Redesigned profile page with role section (user vs. pro)
- Address autofill via backend proxy to Nominatim (OpenStreetMap)
- Pro profile includes service categories, service areas, and verification status

### Service Categories
- Admin-managed service categories with CRUD in the admin dashboard
- Category pro counts displayed on browse/search pages
- Seeded default categories on startup via `SeedDataService`
- Services hero section with layout polish

### Service Area Filtering
- Country → State → District → PIN hierarchy
- Admin CRUD for service areas (in admin-users page)
- Pros and jobs validated against registered service areas
- Find-a-pro filtered by service area; seeded with Trivandrum data

### Job Posting & Bidding
- Users post jobs; pros browse available jobs filtered by service area and category
- Pros bid on jobs; users review and accept bids
- Job status lifecycle: open → in-progress → completed

### Job Completion & Insurance
- Job completion flow with sign-off
- Job notifications sent to involved parties
- Job insurance tracking per job

### Payments
- Razorpay integration for job payments
- UPI payment pre-fill for faster checkout
- Admin payments tab with refund action
- Refund tracking and admin dispute UI

### Messaging
- Direct messaging between users and pros
- `MessageIndex` tracks user-pair conversations

### Notifications
- User notifications for job activity (bids, status changes, completions)
- Notification bell in the UI; notification list page

### Admin Dashboard
- Category management (create, edit, delete, view pro counts)
- Service area management
- User and pro management
- Payments tab with refund action and dispute resolution
- Settings toggle for platform-level flags

### Mobile Apps (Android & iOS)
- Capacitor 8 WebView wrappers — all UI/logic lives in the Angular app
- Android: API URLs patched from `localhost` to `10.0.2.2:5101` for emulator
- iOS: no URL patching needed; simulator reaches `localhost` directly
- `www/` is a static snapshot — run `npm run web:sync` after any UI change

---

## Technology Stack

### Frontend (`prohub-ui`)
- Angular 20.2.x (standalone components)
- Angular Material Design
- TypeScript 5.9 (strict mode)
- SCSS with CSS custom properties (`var(--color-primary)`, `var(--text-h1)`, etc.)
- JWT + Azure MSAL authentication
- Lazy-loaded feature modules

### Backend (`ProHubAPI/ServiceProviderAPI`)
- ASP.NET Core 8, C# 13
- Entity Framework Core 8 + SQL Server
- JWT Bearer authentication with token blacklisting
- Razorpay payment integration
- Msg91 SMS, SMTP email (console-only in dev)
- Nominatim proxy for address autofill

### Mobile
- Capacitor 8 (Android + iOS)
- Android: min SDK 24, target SDK 36, Gradle 8.13.0
- iOS: requires macOS + Xcode + CocoaPods

---

## Configuration

### Backend (`appsettings.json`)

```json
{
  "ConnectionStrings": { "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ProhubDB;..." },
  "Jwt": { "Key": "...", "Issuer": "https://localhost:7042", "Audience": "https://localhost:7042" },
  "Email": { "SmtpServer": "...", "Port": 587, "Username": "...", "Password": "...", "From": "..." },
  "Payment": { "Razorpay": { "KeyId": "...", "KeySecret": "..." } }
}
```

### Frontend (`src/environments/environment.ts`)

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5001/api'
};
```

---

## Key Notes

- **Brand**: Always "yProHub" — lowercase `y` is intentional.
- **CORS**: Dev allows any origin. Restrict in production.
- **JWT secret**: Replace the example value before deploying.
- **Email/SMS**: Console-only in dev; configure SMTP and Msg91 for production.
- **Razorpay**: Test keys in `appsettings.Development.json`; swap live keys for production.
- **Migrations**: Auto-applied on startup. Add new ones with `dotnet ef migrations add <Name>` (run `dotnet build` first).
- **Android sync**: `www/` is not auto-updated — run `npm run web:sync` from `prohub-android/` after every UI change.
