# 📦 Kirubas MSR — Maintenance Accessories Stock Record
### Android Application — Complete Project Reference

**Deep Link / Web Download:** [https://msr-android.web.app/](https://msr-android.web.app/)

> **Version:** 10.0 &nbsp;|&nbsp; **Last updated:** 13 July 2026 12:00
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

---

## 3. Project File Inventory

### Core Source (`app/src/main/java/com/kirubas/msr/`)
- **`ui/`**: Fragments (Dashboard, Ledger, Alerts) and Activities (Report, Profile).
- **`data/`**: Models (`Particular`, `Transaction`, `StockLedger`, `Box`) and `SecurePrefsManager`.
- **`util/`**: `ExcelBuilder`, `PdfBuilder`, `DateUtils`, `SyncManager`.

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
- **State Management**: `SecurePrefsManager` (EncryptedSharedPreferences) stores session tokens, role info, and user preferences locally.
- **Navigation**: Uses a combination of `BottomNavigationView` for core pages and `NavigationView` (Drawer) for administrative tasks.

---

## 7. Firestore Data Structure (v2.0)

- `/users/{uid}`: Profiles with role-based access.
- `/registrationRequests/{id}`: Public status tracking for company registrations.
- `/companies/{cid}`: Organizational roots.
- `/companies/{cid}/reorderRequests/{id}`: Token-secured supplier orders with PO numbers.
- `/stores/{sid}/particulars/{pid}`: Core item data.
- `/stockLedger/{date}/entries/{pid}`: Aggregated daily snapshot data.

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

- **SuperAdmin**: Global access across all companies. Can manage any user or company.
- **Admin**: Full control within their own company. Can create users (defaults to **Viewer** for safety). Restricted from managing SuperAdmins.
- **Manager**: Operational access restricted to their assigned **Store**. Can manage inventory and view reports for that store only.
- **Viewer**: Read-only access to dashboards and reports. Blocked from all transaction and setup actions.

---

## 10. Screen & Navigation Map

- **Splash**: Check for updates and verify session.
- **Login**: Auth via Email or Biometrics.
- **Dashboard**: High-level counts + Movement Trend Chart + Top Movers.
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
- **Occupancy Tracking**: Prevents assigning multiple items to the same physical Box.

### 🔢 PO-Based Reordering (v10.0)
- **Unique PO Numbers**: Automatically generates numeric `PO#123456` for all supplier requests.
- **Attribution Tracking**: Records "Requested by" and "Updated by" user names for full accountability.
- **Secure Portal**: Suppliers update status via a tokenized web link; no MSR account required.
- **Action Notifications**: Formal WhatsApp/Email templates for new, resent, cancelled, and deleted requests.

### 📋 Registration Tracking
- **Public Reference**: Uses `REG-XXXXXX` codes for public status checks without login.
- **Direct Interaction**: One-click WhatsApp/Call actions for admins to contact applicants.

---

## 12. Stock Movement Logic

- **Initialization**: Opening Balance is set when a Particular is created.
- **Transactions**: 
  - `Inward`: `Current Stock + Quantity`
  - `Outward`: `Current Stock - Quantity` (Blocked if insufficient stock)
- **Aggregation**: `StockLedger` entries are updated in the same write-batch as transactions to ensure data consistency.
- **Recalculation**: Background task sums all historical transactions starting from the Opening Balance to correct potential drift.

---

## 13. PDF & Excel Export System

Users can now choose their preferred export format:
- **PDF**: Generates a clean, static report suitable for printing or record-keeping.
- **Excel (.xlsx)**: Generates a spreadsheet using Apache POI, allowing further calculations and sorting in external tools.
- **Universal Share**: Uses the system app chooser to send files via WhatsApp, Gmail, Telegram, etc.

---

## 14. Dashboard Analytics (Charts)

- **Line Chart Integration**: Uses `MPAndroidChart` with a unified scale for Inward, Outward, and Balance trends.
- **Performance Insights**: Dedicated cards showing "Top Inward Movers" and "Top Outward Movers" for the last 7 days.
- **Drill-down**: Tap on any chart or insight to view full-screen detailed analytics.
- **Data Integrity**: Automatic background stock recalculation triggers every 24 hours to ensure local current stock matches transaction history.

---

## 15. Alert System

- **Trigger**: Automatic notification generated when `currentStock <= minStockQty`.
- **Visibility**: Persistent badge count on the Bottom Navigation bar.
- **Resolution**: Alerts are automatically marked as "Resolved" when stock is replenished via an Inward transaction.

---

## 16. App Update System

- **Hosting**: App periodically checks `update.json` hosted on GitHub.
- **In-App Download**: If a new version is detected, users are prompted to download and install the APK directly within the app using `DownloadManager`.

---

## 17. Firebase Hosting — Web Portal & Status Checks

The project uses Firebase Hosting to serve a landing page and interactive portals:
- **Supplier Portal**: Securely view and update reorder status via token-based deep links.
- **Registration Check**: Publicly check the status of company requests using reference numbers.
- **App Download**: Staff landing page for the latest APK version.
- **URL**: [https://msr-android.web.app/](https://msr-android.web.app/)

---

## 18. Offline Support

- **Persistence**: Firestore disk persistence enabled by default.
- **Sync Status**: Dashboard and Ledger items show a "Pending Sync" icon when transactions are made while offline.
- **Conflict Resolution**: Last-write-wins strategy for multi-user edits on identical records.

---

## 19. Login — Biometric & Remember Me

- **Session Persistence**: "Remember Me" keeps users logged in using `SecurePrefsManager`.
- **Biometrics**: Integration with Android BiometricPrompt for fingerprint/face unlock.
- **Password Security**: Uses `EncryptedSharedPreferences` for storing sensitive tokens on-device.

---

## 20. Device Compatibility & 16KB Support

- **Min SDK**: API 26 (Android 8.0) for modern Java features.
- **Target SDK**: API 36 (Android 16 preview support).
- **16KB Page Alignment**: Built with AGP 9.2.0 for compatibility with modern ARM kernel page sizes.

---

## 21. Gradle Dependencies (Exact Versions)

```gradle
implementation 'androidx.appcompat:appcompat:1.7.0'
implementation 'com.google.android.material:material:1.12.0'
implementation 'com.google.firebase:firebase-bom:33.1.0'
implementation 'com.itextpdf:itext7-core:7.2.5'
implementation 'org.apache.poi:poi:5.2.5'
implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'
implementation 'com.github.bumptech.glide:glide:4.16.0'
```

---

## 22. What Still Needs Implementation

- **Multi-Language Support**: RTL and local language translations.
- **Deep Analytics**: Long-term stock usage forecasting using linear regression.
- **Supplier Portal v2**: Completed token-based tracking; future updates include multi-item batch orders.

---

## 23. Known TODOs in Generated Code

- **Backup**: Implement secondary database backup (Firebase to CSV export scheduled).
- **Image Upload**: Particular item image support in Firestore Storage.
- **Unit Conversion**: Better handling for fractional quantities (kgs, grams).

---

## 24. Firestore Indexes Required

Composite Indexes for `entries` (Collection Group):
- `companyId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, particularId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, storeId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, particularId ASC, storeId ASC, dateKey ASC, __name__ ASC`
- `companyId ASC, boxId ASC, dateKey ASC, __name__ ASC`

---

## 25. Changelog

### v10.0 — PO Tracking & Secure Attribution (July 2026)
- **PO Numbering**: Implemented unique numeric `PO#123456` system for all reorder requests.
- **Full Attribution**: Added tracking for "Requested by" and "Updated by" (User or Supplier) across all platforms.
- **Enhanced Notifications**: Formal messaging templates for New, Resent, Cancelled, and Deleted actions, including PO numbers and user names.
- **Web Portal Fixes**: Resolved permission errors for suppliers by embedding customer metadata in request documents.
- **Registration Ref-System**: Integrated public `REG-XXXXXX` status checks on the web portal.
- **Operational UX**: Forced browser-only link opening for mobile suppliers and restricted past-date selections for reorders.

### v9.0 — Reorder & Supplier Portal (July 2026)
- **Supplier Portal**: Launched web-based portal for suppliers to acknowledge and update reorder requests via secure tokens.
- **Reorder Workflow**: Integrated WhatsApp/Email messaging with auto-generated portal links.
- **Auto-Inward**: Added ability to pre-fill Inward Transactions directly from completed reorder requests.
- **Android 14 Compatibility**: Migrated all broadcast receivers to `ContextCompat` with `RECEIVER_NOT_EXPORTED` flags for enhanced security.
- **User Management Safety**: Prevented self-role demotion and defaulted new user creation to the "Viewer" role.
- **Administrative Isolation**: Restricted regular Admins from viewing or editing SuperAdmin accounts.
- **Pivot Report Optimization**: Resolved loading state hanging and moved Export actions to the Toolbar to increase viewable report space.
- **Chart Refinement**: Unified the Y-axis scale to prevent multi-axis confusion and added real-time "Top Movers" insights.
- **Indexing Safety**: Added proactive error handling for Firestore Index requirements with direct links for easy setup.

### v8.0 — Analytics & Advanced Export (July 2026)
- **Dashboard Charts**: Added a Line Chart at the top of the dashboard to show stock trends.
- **Excel Export**: Implemented full support for `.xlsx` exports in both Stock Reports and Ledger.
- **UI Consolidation**: Merged multiple export/share buttons into a single format-selector dialog.
- **Icon-Only Actions**: Converted "Generate", "Filter", and "Reset" buttons to modern icons for space efficiency.
- **Platform Upgrade**: Increased `minSdkVersion` to 26 to support modern Excel generation libraries.

---

*Kirubas MSR — Maintenance Accessories Stock Record*
*Package: `com.kirubas.msr` | minSdk: 26 | targetSdk: 36*
