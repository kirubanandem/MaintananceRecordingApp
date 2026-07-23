# 📦 Kirubas MSR — Maintenance Accessories Stock Record
### Android Application — Complete Project Reference

**Deep Link / Web Download:** [https://msr-android.web.app/](https://msr-android.web.app/)

> **Version:** 12.0 &nbsp;|&nbsp; **Last updated:** 23 July 2026 14:00
> **Android Studio:** Ladybug &nbsp;|&nbsp; **AGP:** 9.2.0 &nbsp;|&nbsp; **Gradle:** 9.4.1

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack & Versions](#2-tech-stack--versions)
3. [Project File Inventory](#3-project-file-inventory)
4. [How to Open in Android Studio](#4-how-to-open-in-android-studio)
5. [Firebase Setup (Step-by-Step)](#5-firebase-setup-step-by-step)
6. [App Architecture](#6-app-architecture)
7. [Firestore Data Structure](#7-firestore-data-structure)
8. [Firestore Security Rules](#8-firestore-security-rules)
9. [User Roles & Permissions](#9-user-roles--permissions)
10. [Screen & Navigation Map](#10-screen--navigation-map)
11. [Feature Specifications](#11-feature-specifications)
12. [Stock Movement Logic](#12-stock-movement-logic)
13. [PDF & Excel Export System](#13-pdf--excel-export-system)
14. [Dashboard Analytics (Charts)](#14-dashboard-analytics-charts)
15. [Alert System](#15-alert-system)
16. [App Update System](#16-app-update-system)
17. [Firebase Hosting — Download Page](#17-firebase-hosting--download-page)
18. [Offline Support](#18-offline-support)
19. [Login — Biometric & Remember Me](#19-login--biometric--remember-me)
20. [Device Compatibility & 16KB Support](#20-device-compatibility--16kb-support)
21. [Gradle Dependencies (Exact Versions)](#21-gradle-dependencies-exact-versions)
22. [What Still Needs Implementation](#22-what-still-needs-implementation)
23. [Known TODOs in Generated Code](#23-known-todos-in-generated-code)
24. [Firestore Indexes Required](#24-firestore-indexes-required)
25. [Changelog](#25-changelog)

---

## 1. Project Overview

**App Name:** Kirubas MSR — Maintenance Accessories Stock Record  
**Package:** `com.kirubas.msr`  
**Platform:** Android (Java + XML Views)  
**Backend:** Firebase Firestore (Free Tier — Spark Plan)  

### What it does
A daily accessories stock monitoring system for physical inventory organized in a strict multi-tenant location hierarchy:
`Company → Store → Wardrobe → Rack → Box → Particular (item)`

### Key design decisions
- **Visual Analytics:** Dashboard includes a Line Chart with a unified Y-axis showing inward/outward/stock trends for the last 7 days.
- **Multi-Format Export:** Consistently export reports to both PDF (iText) and Excel (Apache POI) via a centralized toolbar action.
- **Role Isolation:** Admins are strictly isolated to their company data and cannot view or manage SuperAdmin accounts.
- **Space-Optimized UI:** Maximized reporting space by moving action buttons (Export) to the top title bar and using a collapsible filter system.
- **Android 14 Ready:** Fully compatible with Android 14 (API 34) security requirements for background broadcasts.

---

## 2. Tech Stack & Versions

| Layer | Choice | Version |
|---|---|---|
| Language | Java | 17 |
| minSdk | Android 8.0 | API 26 (Required for POI) |
| Firebase BOM | All Firebase libs | 33.1.0 |
| PDF Export | iText 7 Community | 7.2.5 |
| Excel Export | Apache POI | 5.2.5 |
| Charts | MPAndroidChart | v3.1.0 |
| Local Cache | SQLite / Room | 2.6.1 |

---

## 3. Project File Inventory

### Core Source (`app/src/main/java/com/kirubas/msr/`)
- **`ui/`**: Fragments (Dashboard, Ledger, Alerts) and Activities (Report, Profile).
- **`data/`**: Models (`Particular`, `Transaction`, `StockLedger`, `Box`) and `SecurePrefsManager`.
- **`data/local/room/`**: Room Persistence Layer (`MSRDatabase`, `ParticularDao`, `ParticularEntity`).
- **`util/`**: `ExcelBuilder`, `PdfBuilder`, `DateUtils`, `SyncManager`, `BackupUtils`.

---

## 4. How to Open in Android Studio

1. **Clone the Repository**: Use `git clone` or download the ZIP.
2. **Open Project**: Launch Android Studio and select "Open" -> choose the project root folder.
3. **Gradle Sync**: Wait for the automatic Gradle synchronization to complete.
4. **Firebase Config**: Add your `google-services.json` to the `app/` folder (Download from Firebase Console).
5. **Run**: Select your device/emulator and press the "Run" button.

---

## 5. Firebase Setup (Step-by-Step)

1. **Create Project**: Start a new project in the [Firebase Console](https://console.firebase.google.com/).
2. **Add Android App**: Register `com.kirubas.msr` and download `google-services.json`.
3. **Authentication**: Enable **Email/Password** provider.
4. **Firestore Database**: Create in **Native Mode**. Start with test rules or use the provided snippets in Section 8.
5. **Hosting**: Initialize Firebase Hosting to deploy the [MSR Web Portal](https://msr-android.web.app/).

---

## 6. App Architecture

- **Pattern**: Clean Architecture principles using Activity/Fragment hierarchy.
- **Data Flow**: Fragments use `FirebaseFirestore` directly with SnapshotListeners for real-time UI updates.
- **Local Persistence**: **Room Database** acts as a caching layer for particulars to ensure instant UI loading and offline availability.
- **State Management**: `SecurePrefsManager` (EncryptedSharedPreferences) stores session tokens, role info, and user preferences locally.
- **Navigation**: Uses a combination of `BottomNavigationView` for core pages and `NavigationView` (Drawer) for administrative tasks.

---

## 7. Firestore Data Structure (v2.0)

- `/users/{uid}`: Profiles with role-based access.
- `/registrationRequests/{id}`: Public status tracking for company registrations.
- `/companies/{cid}`: Organizational roots.
- `/companies/{cid}/reorderRequests/{id}`: Token-secured supplier orders with sequential PO numbers.
- `/support_chats/{uid}/messages/{mid}`: Customer support chat history.
- `/broadcasts/{id}`: Global and targeted announcements (by role/status).
- `/stores/{sid}/particulars/{pid}`: Core item data.
- `/stockLedger/{date}/entries/{pid}`: Aggregated daily snapshot data.
- `/config/maintenance`: System-wide maintenance mode kill-switch.

---

## 8. Firestore Security Rules

```javascript
service cloud.firestore {
  match /databases/{database}/documents {
    // Helper to check user role
    function getRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    
    // User-specific profile access
    match /users/{userId} {
      allow read, write: if request.auth != null && (request.auth.uid == userId || getRole() == 'superadmin');
    }

    // Company data isolation
    match /{path=**}/companies/{companyId} {
      allow read: if request.auth != null && (getRole() == 'superadmin' || get(/databases/$(database)/documents/users/$(request.auth.uid)).data.companyId == companyId);
      allow write: if request.auth != null && (getRole() == 'superadmin' || (getRole() == 'admin' && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.companyId == companyId));
    }
  }
}
```

---

## 9. User Roles & Permissions

- **SuperAdmin**: Global access across all companies. Can manage any user or company. Can initiate support chats and toggle maintenance mode.
- **Admin**: Full control within their own company. Can create users (defaults to **Viewer** for safety). Restricted from managing SuperAdmins.
- **Manager**: Operational access restricted to their assigned **Store**. Can manage inventory and view reports for that store only.
- **Viewer**: Read-only access to dashboards and reports. Blocked from all transaction and setup actions.

---

## 10. Screen & Navigation Map

- **Splash**: Check for updates, maintenance mode, and verify session.
- **Login**: Auth via Email or Biometrics.
- **Dashboard**: High-level counts + Movement Trend Chart + Top Movers + Real-time support/registration alerts.
- **System Insights**: SuperAdmin only stats dashboard (Active companies, total users, trial health).
- **Transaction**: Dedicated screen for Inward/Outward stock entry.
- **Ledger**: Historical transaction log with multi-format export.
- **Pivot Report**: Complex cross-tabular reporting for date ranges.
- **Admin Panels**: Company, Store, Wardrobe, Rack, and Box configuration.

---

## 11. Feature Specifications

### 📊 Pivot Stock Report
- **Grid View**: A massive cross-tabular report showing Opening, Inward, Outward, and Closing stock for all items across selected dates.
- **Natural Sorting**: Inventory is organized by Box hierarchy using natural alphanumeric sorting (B1, B2, B10).
- **Infinite Scroll**: Supports horizontal and vertical scrolling for deep data exploration.

### 📝 Dynamic Stock Ledger
- **Transaction History**: Real-time list of all stock movements with DC number tracking.
- **Smart Filters**: Filter by Date Range, Particular, Party, or Document Number.
- **Live Sync**: Indicates "Pending Sync" status for offline transactions.

### 📦 Hierarchy Management
- **Logical Grouping**: Maintain items within a strict `Store → Wardrobe → Rack → Box` structure.
- **Safety Blocks**: Particulars tab now blocks item creation until a specific store is selected, and enforces mandatory box linkage.

### 🔢 Sequential PO Reordering
- **Sequential PO Numbers**: Automatically generates numeric `PO-00001` in sequence using Firestore transactions.
- **Attribution Tracking**: Records "Requested by" and "Updated by" user names.

### 💬 Support Chat 3.0 (v12.0)
- **Bidirectional Alerts**: Distinct unread counts for Admin vs User (`unreadCountForAdmin`, `unreadCountForUser`).
- **Dashboard Alerts**: Immediate visibility for new messages on the home dashboard.
- **Admin-Initiated**: Super Admins can now start chats with any user directly from User Management.

### 📋 Company Management (Super Admin)
- **Centralized Activation**: Activate or Deactivate companies directly from the global "All Companies" list.
- **System Insights**: Real-time health check showing active/total companies, expiring trials (7-day window), and global item counts.

---

## 12. Stock Movement Logic

- **Initialization**: Opening Balance is set when a Particular is created.
- **Transactions**: 
  - `Inward`: `Current Stock + Quantity`
  - `Outward`: `Current Stock - Quantity` (Blocked if insufficient stock)
- **Aggregation**: `StockLedger` entries are updated in the same write-batch as transactions.
- **Recalculation**: Background task sums all historical transactions starting from the Opening Balance to correct potential drift.

---

## 13. PDF & Excel Export System

Users can now choose their preferred export format:
- **PDF**: Generates a clean, static report suitable for printing.
- **Excel (.xlsx)**: Generates a spreadsheet using Apache POI.
- **Backup Options (v12.0)**: Selective export allows picking specific categories (Users, Parties, etc.) and date ranges for transactions/ledger to optimize data usage.

---

## 14. Dashboard Analytics (Charts)

- **Line Chart Integration**: Uses `MPAndroidChart` showing trends for the last 7 days.
- **Targeted Insights**: Charts automatically adjust based on the current list filters (Total/Low/Healthy).
- **Movers**: Real-time summary of top 5 items by inward/outward volume.

---

## 15. Alert System

- **Trigger**: Automatic notification when `currentStock <= minStockQty`.
- **Expiry Alerts (v12.0)**: Automatic 7-day warning popups for trial/grace period expirations (limited to once every 24 hours per user).
- **Admin Alerts**: Dashboard notifications for unread support chats and pending registrations.

---

## 16. App Update System (Mandatory)

- **Hosting**: App periodically checks `update.json` hosted on the MSR Web Portal.
- **Mandatory Enforcement**: Detects versions and blocks access until updated.

---

## 17. Firebase Hosting — Web Portal & Status Checks

The project uses Firebase Hosting to serve a landing page and interactive portals:
- **Supplier Portal**: Securely update reorder status via token-based deep links.
- **Registration Check**: Securely check request status.
- **App Download**: Staff landing page for the latest APK version.
- **URL**: [https://msr-android.web.app/](https://msr-android.web.app/)

---

## 18. Offline Support (Enhanced v12.0)

- **Room Persistence**: The Particulars list is now cached in a local SQLite database using Room.
- **Instant Load**: Screen displays cached data immediately while syncing Firestore updates in the background.
- **Resilience**: Full inventory viewing is available without any internet connection.

---

## 19. Login — Biometric & Remember Me

- **Session Persistence**: "Remember Me" keeps users logged in using `SecurePrefsManager`.
- **Biometrics**: Integration with Android BiometricPrompt.

---

## 20. Device Compatibility & 16KB Support

- **Min SDK**: API 26 (Android 8.0).
- **Target SDK**: API 36.
- **16KB Page Alignment**: Built with AGP 9.2.0 for modern ARM kernel compatibility.

---

## 21. Gradle Dependencies (Exact Versions)

```gradle
implementation 'androidx.appcompat:appcompat:1.7.0'
implementation 'com.google.android.material:material:1.12.0'
implementation 'com.google.firebase:firebase-bom:33.1.0'
implementation 'androidx.room:room-runtime:2.6.1'
implementation 'com.itextpdf:itext7-core:7.2.5'
implementation 'org.apache.poi:poi:5.2.5'
implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'
```

---

## 22. What Still Needs Implementation

- **Barcode Scanning**: Integrated QR/Barcode support for quick item lookup.
- **Multi-Language Support**: RTL and local language translations.
- **Transaction Attachments**: Supporting photo uploads for physical DC records.

---

## 23. Known TODOs in Generated Code

- **Image Storage**: Scaling storage for particular item thumbnails.
- **Unit Conversion**: Better handling for fractional quantities (kgs, grams).

---

## 24. Firestore Indexes Required

Composite Indexes for `entries` (Collection Group):
- `companyId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, particularId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, storeId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, particularId ASC, storeId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, boxId ASC, dateKey ASC, __name__ ASC`

New Indexes (v12.0):
- `parties: companyId ASC` (Collection Group)
- `stores: companyId ASC` (Collection Group)

---

## 25. Changelog

### v12.0 — Local Cache & Advanced Administration (July 2026)
- **Room Cache Implementation:** Integrated Room database to cache Particulars. Inventory now loads instantly and works fully offline.
- **Selective Backup:** Enhanced the export system with category checkboxes and date range filters to minimize Firestore read costs.
- **SuperAdmin Insights:** Launched a new dashboard for SuperAdmins showing real-time global health stats (Active companies, expiring trials, user counts).
- **Maintenance Mode:** Added a system-wide "Kill Switch" allowing SuperAdmins to block app access with custom messages during maintenance.
- **Targeted Broadcasts:** Announcements can now be filtered by User Role (Admin/Manager/Viewer) and Company Status (Trial/Active/Grace).
- **Support Chat 3.0:** Implemented bidirectional unread tracking and added the ability for SuperAdmins to initiate chats directly from User Management.
- **Safety Validations:** Forced store selection before adding particulars and enforced mandatory box linkage to prevent "orphan" items.
- **Proactive Notifications:** Implemented a 24-hour frequency-limited alert for upcoming company expirations (Trial/Grace).
- **Global Management:** Companies can now be Activated/Deactivated directly from the "All Companies" management list.

### v11.0 — UI UX & Edge-to-Edge Optimization (July 2026)
- **Bottom Navigation Placement:** Fixed positioning of `BottomNavigationView` for modern Edge-to-Edge displays.
- **Window Insets Refinement:** Optimized handling of system bars and keyboard (IME) transitions.

### v10.0 — PO Tracking & Secure Attribution & Atomic Integrity (July 2026)
- **Atomic Stock Deletions:** Migrated transaction deletions to `db.runTransaction` blocks.
- **Sequential PO Numbers:** Replaced random PO numbers with transaction-safe sequential `PO-XXXXX` system.
- **Support Chat 2.0:** Implemented Bubble UI with relative timestamps.
- **Full Company Backup:** Added initial "Full Data Backup" to Excel.

### v9.0 — Reorder & Supplier Portal (July 2026)
- **Supplier Portal**: Launched web-based portal for suppliers via secure tokens.
- **Reorder Workflow**: Integrated WhatsApp/Email messaging with auto-generated portal links.

---

*Kirubas MSR — Maintenance Accessories Stock Record*
*Package: `com.kirubas.msr` | minSdk: 26 | targetSdk: 36*
