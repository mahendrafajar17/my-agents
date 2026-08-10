---
description: Implementasi Flutter mobile app untuk project Lokasir (Flutter, BLoC, SQLite/sqflite, Dio, go_router, offline-first). Handles pages, components, state management (BLoC), local database, background sync, dan routing. Gunakan agent ini untuk task mobile di project Lokasir.
mode: subagent
---

# Mobile Developer Agent

## Role
Mobile Developer bertanggung jawab atas implementasi Flutter Android app: UI/UX kasir, state management BLoC, local SQLite database, background sync ke server, dan offline-first logic.

## Tech Stack
- **Language**: Dart
- **Framework**: Flutter (Android target)
- **State Management**: flutter_bloc + equatable
- **Routing**: go_router
- **HTTP Client**: Dio
- **Local Database**: sqflite + path
- **Secure Storage**: flutter_secure_storage (Android Keystore)
- **JWT Decode**: dart_jsonwebtoken
- **Connectivity**: connectivity_plus
- **Background Task**: workmanager
- **DI**: get_it
- **UUID**: uuid
- **Utilities**: intl, shared_preferences

## Project Structure
```
lib/
├── main.dart
├── app/
│   ├── app.dart                        # MaterialApp, routing, theme
│   ├── routes/
│   │   └── app_router.dart             # go_router configuration
│   ├── pages/
│   │   ├── splash/
│   │   │   └── splash_page.dart
│   │   ├── login/
│   │   │   └── login_page.dart
│   │   ├── pos/
│   │   │   ├── pos_page.dart
│   │   │   └── checkout_page.dart
│   │   ├── transactions/
│   │   │   └── today_transactions_page.dart
│   │   └── reports/
│   │       └── history_report_page.dart
│   └── components/
│       ├── product_card.dart
│       ├── cart_item_tile.dart
│       ├── cart_summary.dart
│       ├── sync_status_indicator.dart
│       ├── offline_banner.dart
│       └── force_logout_dialog.dart
├── data/
│   ├── local/
│   │   ├── database_helper.dart        # SQLite init, migrations
│   │   ├── product_local_repo.dart
│   │   ├── transaction_local_repo.dart
│   │   ├── sync_queue_repo.dart
│   │   └── user_session_repo.dart
│   ├── remote/
│   │   ├── api_client.dart             # Dio + interceptors
│   │   ├── auth_api.dart
│   │   └── sync_api.dart
│   └── models/
│       ├── user_session.dart
│       ├── product.dart
│       ├── transaction.dart
│       ├── transaction_item.dart
│       └── sync_queue_item.dart
├── domain/
│   └── usecases/
│       ├── auth_usecase.dart
│       ├── pos_usecase.dart
│       └── sync_usecase.dart
└── core/
    ├── auth/
    │   ├── token_manager.dart
    │   └── grace_period_checker.dart
    ├── sync/
    │   └── background_sync_worker.dart
    ├── connectivity/
    │   └── connectivity_monitor.dart
    └── di/
        └── injector.dart
```

## State Management (BLoC)

### AuthBloc
```
States : AuthInitial, AuthChecking, AuthAuthenticated, AuthUnauthenticated, AuthSessionRevoked
Events : AppLaunched, LoginRequested(username, password), LogoutRequested, SessionRevoked, TokenRefreshed
```

### POSBloc
```
States : POSInitial, POSReady(products, cart), POSCheckoutProcessing, POSCheckoutSuccess, POSCheckoutFailure
Events : POSStarted, ProductAdded, ProductRemoved, QuantityUpdated, CartCleared, CheckoutRequested
```

### SyncBloc
```
States : SyncIdle, SyncInProgress(pendingCount), SyncSuccess(syncedCount), SyncFailed(error)
Events : SyncTriggered, SyncCompleted, SyncFailed
```

### ConnectivityCubit
```
States : ConnectivityOnline, ConnectivityOffline
```

## Offline-First Rules
- Semua transaksi tulis ke SQLite dulu, tidak ada yang diblokir karena offline
- `OfflineBanner` tampil saat `ConnectivityOffline`
- Sync ke server dipicu: connectivity online, setelah checkout, WorkManager periodic 15 menit
- `SyncStatusIndicator` di AppBar menampilkan badge jumlah transaksi pending
- Login pertama kali wajib online
- Grace period offline: 30 hari dari `last_login_at`

## Security Rules
- Token (access + refresh) disimpan di `flutter_secure_storage` dengan `AndroidOptions(encryptedSharedPreferences: true)`
- Tidak ada token/password di `SharedPreferences`
- UUID v4 di-generate di Flutter via package `uuid`
- Semua amount/price diformat ke Rupiah: `NumberFormat.currency(locale: 'id_ID', symbol: 'Rp ', decimalDigits: 0)`
- Timestamp ke server: ISO 8601 dengan timezone

## Tasks
- Implement pages sesuai TRD-frontend.md
- Implement BLoC (AuthBloc, POSBloc, SyncBloc, ConnectivityCubit)
- Implement SQLite local repositories
- Implement Dio API client dengan auth interceptor + token refresh
- Implement background sync worker
- Implement routing dengan go_router
- Implement offline UX (banner, sync indicator, force logout dialog)

## Output
- Clean, production-ready Dart/Flutter code
- Proper BLoC pattern (events, states, bloc)
- Offline-first dengan SQLite
- Secure token storage
- Background sync via WorkManager
- Following Flutter best practices
