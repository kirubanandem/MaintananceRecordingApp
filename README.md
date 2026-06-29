# 📦 Kirubas MSR — Maintenance Accessories Stock Record
### Android Application — Complete Project Reference

**Deep Link / Web Download:** [https://msr-android.web.app/](https://msr-android.web.app/)

> **Version:** 1.7 &nbsp;|&nbsp; **Last updated:** July 2026  
> **Android Studio:** Quail 2026.1.1 Patch 2 &nbsp;|&nbsp; **AGP:** 9.2.0 &nbsp;|&nbsp; **Gradle:** 9.4.1

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
13. [PDF Export & Gmail Share](#13-pdf-export--gmail-share)
14. [Alert System](#14-alert-system)
15. [App Update System](#15-app-update-system)
16. [Firebase Hosting — Download Page](#16-firebase-hosting--download-page)
17. [Offline Support](#17-offline-support)
18. [Login — Biometric & Remember Me](#18-login--biometric--remember-me)
19. [Device Compatibility & 16KB Support](#19-device-compatibility--16kb-support)
20. [Gradle Dependencies (Exact Versions)](#20-gradle-dependencies-exact-versions)
21. [What Still Needs Implementation](#21-what-still-needs-implementation)
22. [Known TODOs in Generated Code](#22-known-todos-in-generated-code)
23. [Firestore Indexes Required](#23-firestore-indexes-required)
24. [Implementation Checklist](#24-implementation-checklist)
25. [Changelog](#25-changelog)

---

## 1. Project Overview

**App Name:** Kirubas MSR — Maintenance Accessories Stock Record  
**Package:** `com.kirubas.msr`  
**Platform:** Android (Java + XML Views)  
**Backend:** Firebase Firestore (Free Tier — Spark Plan)  

### What it does
A daily accessories stock monitoring system for physical inventory organized in a strict multi-tenant location hierarchy:

```
Company → Store → Wardrobe → Rack → Box → Particular (item)
```

Each day, stock is updated via Inward and Outward transactions. Closing stock auto-becomes the next day's opening balance. Admins are alerted when stock falls below a configurable minimum level.

### Key design decisions
- **Unified Navigation:** Single Drawer Menu + Bottom Navigation host all user roles and admin features.
- **Multi-tenant Hierarchy:** Physically nested Store-level isolation for Wardrobes, Racks, and Boxes under each Company.
- **Admin Correction Rights:** SuperAdmins and Admins can correct **Opening Balance** and **Minimum Qty** for existing items with auto-stock adjustment.
- **Intelligent CSV Import:** Store-aware bulk import with automatic hierarchy creation (Store -> Wardrobe -> Rack -> Box).
- **Hardened Security:** Server-side enforcement of user activation and privilege levels via strict Firestore Rules.
- **Smart Stock Reversion:** Deleting a ledger entry automatically reverses the stock change and re-evaluates low-stock alerts.
- **Offline-first:** Firestore offline persistence enabled by default.

---

## 2. Tech Stack & Versions

| Layer | Choice | Version |
|---|---|---|
| Language | Java | 17 (source/target) |
| UI | XML Layouts + ViewBinding | — |
| Android Gradle Plugin | AGP | 9.2.0 |
| Gradle | Gradle Wrapper | 9.4.1 |
| compileSdk / targetSdk | Android API | 36 |
| minSdk | Android 5.0 | API 21 |
| Firebase BOM | All Firebase libs | 33.1.0 |
| Firestore | firebase-firestore | via BOM |
| Auth | firebase-auth | via BOM |
| FCM | firebase-messaging | via BOM |
| google-services plugin | — | 4.4.2 |
| Material Design | material | 1.12.0 |
| PDF Export | iText 7 Community | 7.2.5 |
| Charts | MPAndroidChart | v3.1.0 (JitPack) |
| HTTP client | Retrofit2 + Gson | 2.11.0 |
| OkHttp logging | logging-interceptor | 4.12.0 |
| Image loading | Glide | 4.16.0 |
| Biometric | androidx.biometric | 1.2.0-alpha05 |
| Secure storage | security-crypto | 1.1.0-alpha06 |
| Desugaring | desugar_jdk_libs | 2.0.4 |
| MultiDex | multidex | 2.0.1 |
| Navigation | navigation-fragment/ui | 2.7.7 |
| ViewModel / LiveData | lifecycle | 2.8.3 |

---

## 7. Firestore Data Structure (v1.5 - Hierarchical)

```
/users/{uid}
    name, email, role, companyId, storeId, storeName, isActive, lastLoginAt

/companies/{companyId}
    name, address, isActive, createdAt

/companies/{companyId}/stores/{storeId}
    name, description, createdAt

/companies/{companyId}/stores/{storeId}/wardrobes/{wardrobeId}
    name, description, order, storeId, storeName, createdAt

/companies/{companyId}/stores/{storeId}/wardrobes/{wardrobeId}/racks/{rackId}
    name, order, wardrobeId, wardrobeName, storeId, storeName, createdAt

/companies/{companyId}/stores/{storeId}/wardrobes/{wardrobeId}/racks/{rackId}/boxes/{boxId}
    name, minStockQty, rackId, rackName, wardrobeId, wardrobeName, storeId, storeName

/companies/{companyId}/stores/{storeId}/particulars/{pid}
    name, unit, openingBalance, currentStock, minStockQty, active
    storeId, storeName, wardrobeId, wardrobeName, rackId, rackName, boxId, boxName

/companies/{companyId}/transactions/{txnId}
    particularId, particularName, type, qty, transactionDate, storeId, storeName, createdByName, createdByUid

/companies/{companyId}/stores/{storeId}/stockLedger/{dateStr}/entries/{pid}
    particularId, particularName, openingStock, totalInward, totalOutward, closingStock

/companies/{companyId}/stores/{storeId}/alerts/{pid}
    particularId, particularName, currentStock, minStockQty, isResolved, triggeredAt
```

---

## 8. Firestore Security Rules (Hardened v1.5)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isSignedIn() { return request.auth != null; }
    function userDoc() { return get(/databases/$(database)/documents/users/$(request.auth.uid)).data; }
    function isSuperAdmin() { return isSignedIn() && userDoc().get('role', 'viewer') == 'superadmin'; }
    function isUserActive() {
      let user = userDoc();
      return isSignedIn() && (user.get('active', true) == true || user.get('isActive', true) == true);
    }
    function userCompanyId() { return userDoc().get('companyId', ''); }
    function userStoreId() { return userDoc().get('storeId', ''); }
    function isAdminOrAbove() { let role = userDoc().get('role', 'viewer'); return role == 'superadmin' || role == 'admin' || role == 'manager'; }
    function isCompanyActive(cid) {
      let company = get(/databases/$(database)/documents/companies/$(cid)).data;
      return company.get('active', true) == true || company.get('isActive', true) == true;
    }

    function canAccessResource(resourceData) {
       return isSignedIn() && (isSuperAdmin() || (userCompanyId() == resourceData.get('companyId', '') && (isAdminOrAbove() || userStoreId() == '' || resourceData.get('storeId', '') == '' || resourceData.get('storeId', '') == userStoreId())));
    }

    match /users/{uid} {
      allow read: if isSignedIn() && (request.auth.uid == uid || isSuperAdmin());
      allow create: if isSignedIn() && isSuperAdmin();
      allow update: if isSignedIn() && (isSuperAdmin() || (request.auth.uid == uid && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['name', 'lastLoginAt'])));
      allow delete: if isSignedIn() && isSuperAdmin();
    }

    match /companies/{companyId} {
      allow read: if isSignedIn() && (isSuperAdmin() || userCompanyId() == companyId);
      allow create, delete: if isSignedIn() && isSuperAdmin();
      allow update: if isSignedIn() && (isSuperAdmin() || (userCompanyId() == companyId && isCompanyActive(companyId) && isAdminOrAbove()));

      match /stores/{storeId} {
        allow read, write: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && (isAdminOrAbove() || userStoreId() == storeId)));
        
        match /transactions/{id} {
          allow update: if isSignedIn() && isUserActive() && (
            isSuperAdmin() || (userCompanyId() == companyId && (
              isAdminOrAbove() || (request.auth.uid == resource.data.get('createdByUid', '') && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['dcNumber', 'note', 'partyId', 'partyName']))
            ))
          );
        }

        match /{allStoreSubcollections=**} {
          allow read: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && (isAdminOrAbove() || userStoreId() == storeId)));
          allow write: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && isCompanyActive(companyId) && (isAdminOrAbove() || userStoreId() == storeId)));
        }
      }

      match /transactions/{id} {
        allow read: if isSignedIn() && isUserActive() && (isSuperAdmin() || userCompanyId() == companyId);
        allow create: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && isCompanyActive(companyId)));
        allow update: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && isAdminOrAbove()));
        allow delete: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && isAdminOrAbove()));
      }
      
      match /{allOther=**} {
         allow read: if isSignedIn() && isUserActive() && (isSuperAdmin() || userCompanyId() == companyId);
         allow write: if isSignedIn() && isUserActive() && (isSuperAdmin() || (userCompanyId() == companyId && isCompanyActive(companyId) && isAdminOrAbove()));
      }
    }

    // Collection Group Protection
    match /{path=**}/particulars/{id} { allow read: if canAccessResource(resource.data); }
    match /{path=**}/transactions/{id} { allow read: if canAccessResource(resource.data); }
    match /{path=**}/alerts/{id} { allow read: if canAccessResource(resource.data); }
  }
}
```

---

## 9. User Roles & Permissions (v1.5)

| Feature | SuperAdmin | Admin | Manager | Viewer |
|---|:---:|:---:|:---:|:---:|
| Edit Opening Stock / Min Qty | ✅ | ✅ | ❌ | ❌ |
| Create User Accounts | ✅ | ✅ | ❌ | ❌ |
| Manage Stores | ✅ | ✅ | ❌ | ❌ |
| Switch Store Context | ✅ | ✅ | ❌ | ❌ |
| Manage Wardrobes / Racks / Boxes | ✅ | ✅ | ❌ | ❌ |
| Perform Transactions | ✅ | ✅ | ✅ | ❌ |
| Delete/Correct Transactions | ✅ | ✅ | ❌ | ❌ |
| View Help & Manuals | ✅ | ✅ | ✅ | ✅ |
| View Stock Reports | ✅ | ✅ | ✅ | ✅ |

---

## 23. Firestore Indexes Required (Hierarchy Optimized)

| Collection | Fields | Query location |
|---|---|---|
| `particulars` | `companyId ASC`, `storeId ASC`, `active ASC`, `name ASC` | Dashboard |
| `particulars` | `companyId ASC`, `currentStock ASC` | Dashboard Summary |
| `transactions` | `companyId ASC`, `transactionDate DESC`, `createdAt DESC` | Audit Log |
| `transactions` | `storeId ASC`, `createdAt DESC` | Audit Log (Store Filter) |
| `transactions` | `companyId ASC`, `storeId ASC`, `transactionDate DESC` | Ledger |
| `alerts` | `companyId ASC`, `storeId ASC`, `isResolved ASC`, `triggeredAt DESC` | Alerts |
| `wardrobes` | `companyId ASC`, `storeId ASC`, `order ASC` | Wardrobe List |
| `racks` | `companyId ASC`, `storeId ASC`, `wardrobeId ASC`, `order ASC` | Rack List |
| `boxes` | `companyId ASC`, `storeId ASC`, `rackId ASC`, `name ASC` | Box List |

---

## 24. Implementation Checklist

### ✅ Completed (Latest Build)
- [x] **Store Hierarchy**: Physical nesting under Stores implemented.
- [x] **Admin Corrections**: Editable Opening Stock and Min Qty for authorized roles.
- [x] **Security rules**: Privilege escalation protection and activation enforcement.
- [x] **Help & Manuals**: Integrated step-by-step guides for all features.
- [x] **UI Polish**: Version display in Profile; developer credits on Login.
- [x] **Manual Updates**: "Check for updates" button in Profile wired to UpdateChecker.
- [x] **Sync Manager**: Cascading updates for Store/Wardrobe renames.
- [x] **CSV Importer**: Robust regex-based 8-column store-aware import with caching.
- [x] **Unified Navigation**: Single Navigation Drawer for all roles.
- [x] **Ledger UX**: Expandable filters and optimized screen space.
- [x] **Smart Navigation**: Activity tracking via fragment backstack.
- [x] **Reliability**: Improved error dialogs with clickable links for Firestore indexing.

---

## 25. Changelog

### v1.7 — UX Refinement & Reliability (July 2026)
- **Smart Navigation**: Back button now tracks user activity through fragment backstack, returning to Dashboard before exit prompt.
- **User Management Fixes**: Fixed total user count display and improved "Add User" dialog with pre-loading states.
- **Enhanced Error Handling**: Long error messages (like missing Firestore indexes) now show in clickable dialogs instead of being truncated in Toasts.
- **Improved Icons**: Updated drawer menu icons for Logout, Help, and Setup items for better visual clarity.
- **Transaction UX**: Set **INWARD** as the default transaction type for faster data entry.
- **Navigation Shortcuts**: "Store Setup" menu item now navigates directly to the Store tab in the Setup fragment.

### v1.6 — Navigation Consolidation & Seeding (June 2026)
- **Unified Menu System**: Combined Options Menu and Admin Drawer into a single, role-aware Navigation Drawer in `MainActivity`.
- **Improved CSV Import**: Updated sample template and parser to include **Store** column, enabling multi-store bulk seeding.
- **Searchable Admin UI**: Added real-time search filters to Particulars and Box lists to handle large inventories.
- **Ledger UX**: Implemented expand/collapse filter section in Ledger view to maximize screen space for reports.
- **Layout Optimization**: Standardized `fragment_store_aware_list.xml` with integrated search functionality.

### v1.5 — Stock Integrity & UX (June 2026)
- Implemented **Automatic Stock Reversion** upon transaction deletion.
- Added **Immediate Alert Evaluation** on item creation and stock updates.
- Improved **Stock Report** with balance carry-forward for days without transactions.
- Optimized **Resume Experience**: App no longer "locks" or restarts when minimized.
- Hardened **Security Rules** for transaction updates and manager privileges.
- Restored **Firestore Single-field Overrides** for reliable `collectionGroup` filtering.

### v1.4 — Help Menus & UI Finalization (June 2026)
- Added **Help & Manuals** fragment with feature guides.
- Displayed **Application Version** in Profile page footer.
- Added **Manual Update Check** button in Profile.
- Added **Developer Credits** ("Developped By Kirubanandem.S") and hosting link to Login screens.
- Updated dates and documentation to June 2026.

### v1.0 — Initial release (June 2026)

---

*Kirubas MSR — Maintenance Accessories Stock Record*
*Package: `com.kirubas.msr` | minSdk: 21 | targetSdk: 36*
