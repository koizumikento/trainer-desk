# 技術仕様書

## 1. 概要

本ドキュメントは「Trainer Desk」アプリケーションの技術仕様を定義します。

## 2. 技術スタック

### 2.1 フロントエンド

| 項目 | 技術 | バージョン |
|-----|------|----------|
| フレームワーク | Flutter | 3.x |
| 言語 | Dart | 3.x |
| 状態管理 | Riverpod | 2.x |
| ルーティング | go_router | 最新 |
| ローカルストレージ | shared_preferences | 最新 |

### 2.2 バックエンド（Firebase）

| サービス | 用途 |
|---------|------|
| Firebase Authentication | ユーザー認証 |
| Cloud Firestore | データベース |
| Cloud Functions | サーバーレス関数 |
| Cloud Storage | ファイルストレージ |
| Firebase Cloud Messaging | プッシュ通知 |
| Firebase Analytics | アナリティクス |
| Firebase Crashlytics | クラッシュレポート |

### 2.3 開発ツール

| 用途 | ツール |
|-----|-------|
| IDE | Android Studio / VS Code |
| バージョン管理 | Git |
| CI/CD | GitHub Actions |
| コード品質 | Flutter Analyze, Dart Format |
| テスト | Flutter Test, Integration Test |

## 3. アーキテクチャ

### 3.1 全体構成

```
┌─────────────────────────────────────────────────────────────┐
│                      Mobile App (Flutter)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │     UI      │  │   State     │  │  Repository │          │
│  │   (Widgets) │◄─┤ (Riverpod)  │◄─┤   Layer     │          │
│  └─────────────┘  └─────────────┘  └──────┬──────┘          │
└──────────────────────────────────────────┼──────────────────┘
                                           │
                    ┌──────────────────────┼──────────────────┐
                    │                Firebase                  │
                    │  ┌────────────┐  ┌───┴────┐  ┌────────┐ │
                    │  │    Auth    │  │Firestore│  │Storage │ │
                    │  └────────────┘  └────────┘  └────────┘ │
                    │  ┌────────────┐  ┌────────┐             │
                    │  │  Functions │  │  FCM   │             │
                    │  └────────────┘  └────────┘             │
                    └─────────────────────────────────────────┘
```

### 3.2 ディレクトリ構成

```
lib/
├── main.dart                 # エントリーポイント
├── app.dart                  # アプリケーション設定
├── firebase_options.dart     # Firebase設定
│
├── core/                     # コア機能
│   ├── constants/            # 定数
│   ├── exceptions/           # カスタム例外
│   ├── extensions/           # 拡張メソッド
│   ├── theme/                # テーマ設定
│   └── utils/                # ユーティリティ
│
├── data/                     # データ層
│   ├── models/               # データモデル
│   ├── repositories/         # リポジトリ実装
│   └── services/             # 外部サービス連携
│
├── domain/                   # ドメイン層
│   ├── entities/             # エンティティ
│   └── repositories/         # リポジトリインターフェース
│
├── presentation/             # プレゼンテーション層
│   ├── providers/            # Riverpodプロバイダー
│   ├── screens/              # 画面
│   ├── widgets/              # 共通ウィジェット
│   └── router/               # ルーティング設定
│
└── l10n/                     # 多言語対応
    └── app_ja.arb            # 日本語リソース
```

### 3.3 Cloud Functions 構成

```
functions/
├── src/
│   ├── index.ts              # エントリーポイント
│   ├── auth/                 # 認証関連トリガー
│   │   └── onUserCreated.ts
│   ├── reservations/         # 予約関連
│   │   ├── onCreate.ts
│   │   └── onUpdate.ts
│   ├── chat/                 # チャット関連
│   │   └── onMessageCreated.ts
│   ├── notifications/        # 通知関連
│   │   └── sendPushNotification.ts
│   └── scheduled/            # 定期実行
│       └── sendReminders.ts
├── package.json
└── tsconfig.json
```

## 4. 認証設計

### 4.1 認証フロー

```
┌─────────┐     ┌──────────────┐     ┌───────────────────┐
│  User   │────►│ Google OAuth │────►│ Firebase Auth     │
└─────────┘     └──────────────┘     └─────────┬─────────┘
                                               │
                                               ▼
                                     ┌─────────────────┐
                                     │  ID Token発行   │
                                     └─────────┬───────┘
                                               │
                                               ▼
                                     ┌─────────────────┐
                                     │ Firestoreアクセス│
                                     └─────────────────┘
```

### 4.2 Firebase Auth 設定

```dart
// lib/data/services/auth_service.dart
class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  final GoogleSignIn _googleSignIn = GoogleSignIn();

  Future<UserCredential> signInWithGoogle() async {
    final GoogleSignInAccount? googleUser = await _googleSignIn.signIn();
    final GoogleSignInAuthentication? googleAuth =
        await googleUser?.authentication;

    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth?.accessToken,
      idToken: googleAuth?.idToken,
    );

    return await _auth.signInWithCredential(credential);
  }

  Future<void> signOut() async {
    await _googleSignIn.signOut();
    await _auth.signOut();
  }
}
```

## 5. 状態管理

### 5.1 Riverpod プロバイダー設計

```dart
// 認証状態
final authStateProvider = StreamProvider<User?>((ref) {
  return FirebaseAuth.instance.authStateChanges();
});

// ユーザー情報
final userProvider = FutureProvider<AppUser?>((ref) async {
  final authState = ref.watch(authStateProvider);
  return authState.when(
    data: (user) => user != null
        ? ref.read(userRepositoryProvider).getUser(user.uid)
        : null,
    loading: () => null,
    error: (_, __) => null,
  );
});

// 予約一覧
final reservationsProvider = StreamProvider<List<Reservation>>((ref) {
  final user = ref.watch(userProvider);
  return user.when(
    data: (u) => u != null
        ? ref.read(reservationRepositoryProvider).watchReservations(u.id)
        : Stream.value([]),
    loading: () => Stream.value([]),
    error: (_, __) => Stream.value([]),
  );
});
```

## 6. プッシュ通知

### 6.1 FCM 設定

```dart
// lib/data/services/notification_service.dart
class NotificationService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // 通知権限リクエスト
    await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // FCMトークン取得
    final token = await _messaging.getToken();
    if (token != null) {
      await _saveTokenToFirestore(token);
    }

    // トークン更新監視
    _messaging.onTokenRefresh.listen(_saveTokenToFirestore);

    // フォアグラウンドメッセージハンドラ
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // バックグラウンドメッセージハンドラ
    FirebaseMessaging.onBackgroundMessage(_handleBackgroundMessage);
  }
}
```

### 6.2 Cloud Functions 通知送信

```typescript
// functions/src/notifications/sendPushNotification.ts
import * as admin from 'firebase-admin';

export async function sendPushNotification(
  userId: string,
  title: string,
  body: string,
  data?: Record<string, string>
) {
  const userDoc = await admin.firestore()
    .collection('users')
    .doc(userId)
    .get();

  const fcmToken = userDoc.data()?.fcmToken;
  if (!fcmToken) return;

  await admin.messaging().send({
    token: fcmToken,
    notification: { title, body },
    data,
    android: {
      priority: 'high',
      notification: {
        channelId: 'default',
      },
    },
    apns: {
      payload: {
        aps: {
          badge: 1,
          sound: 'default',
        },
      },
    },
  });
}
```

## 7. リアルタイム通信

### 7.1 チャット実装

```dart
// lib/data/repositories/chat_repository.dart
class ChatRepository {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  // メッセージ監視
  Stream<List<ChatMessage>> watchMessages(String chatRoomId) {
    return _firestore
        .collection('chatRooms')
        .doc(chatRoomId)
        .collection('messages')
        .orderBy('createdAt', descending: true)
        .limit(50)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => ChatMessage.fromFirestore(doc))
            .toList());
  }

  // メッセージ送信
  Future<void> sendMessage(String chatRoomId, ChatMessage message) async {
    final batch = _firestore.batch();

    // メッセージ追加
    final messageRef = _firestore
        .collection('chatRooms')
        .doc(chatRoomId)
        .collection('messages')
        .doc();
    batch.set(messageRef, message.toFirestore());

    // チャットルーム更新
    final chatRoomRef = _firestore.collection('chatRooms').doc(chatRoomId);
    batch.update(chatRoomRef, {
      'lastMessage': message.content,
      'lastMessageAt': FieldValue.serverTimestamp(),
    });

    await batch.commit();
  }
}
```

## 8. ファイルストレージ

### 8.1 画像アップロード

```dart
// lib/data/services/storage_service.dart
class StorageService {
  final FirebaseStorage _storage = FirebaseStorage.instance;

  Future<String> uploadImage({
    required String path,
    required File file,
  }) async {
    final ref = _storage.ref().child(path);

    // 画像圧縮
    final compressedFile = await _compressImage(file);

    // アップロード
    final uploadTask = ref.putFile(
      compressedFile,
      SettableMetadata(contentType: 'image/jpeg'),
    );

    final snapshot = await uploadTask;
    return await snapshot.ref.getDownloadURL();
  }

  Future<File> _compressImage(File file) async {
    // 画像圧縮処理
    // flutter_image_compress パッケージ使用
  }
}
```

### 8.2 Storage構成

```
storage/
├── users/
│   └── {userId}/
│       └── profile.jpg
├── meals/
│   └── {userId}/
│       └── {mealId}.jpg
└── chat/
    └── {chatRoomId}/
        └── {messageId}.jpg
```

## 9. セキュリティ

### 9.1 Firestore Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // ユーザー自身のドキュメントのみ読み書き可能
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }

    // 予約は関係者のみアクセス可能
    match /reservations/{reservationId} {
      allow read: if request.auth != null &&
        (request.auth.uid == resource.data.trainerId ||
         request.auth.uid == resource.data.traineeId);
      allow create: if request.auth != null;
      allow update: if request.auth != null &&
        (request.auth.uid == resource.data.trainerId ||
         request.auth.uid == resource.data.traineeId);
    }

    // チャットルームは参加者のみアクセス可能
    match /chatRooms/{chatRoomId} {
      allow read, write: if request.auth != null &&
        request.auth.uid in resource.data.participants;

      match /messages/{messageId} {
        allow read, write: if request.auth != null &&
          request.auth.uid in get(/databases/$(database)/documents/chatRooms/$(chatRoomId)).data.participants;
      }
    }
  }
}
```

### 9.2 Storage Security Rules

```javascript
// storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // ユーザーは自分のフォルダにのみ書き込み可能
    match /users/{userId}/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }

    match /meals/{userId}/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }
  }
}
```

## 10. テスト戦略

### 10.1 テスト種別

| 種別 | 対象 | ツール |
|-----|------|-------|
| 単体テスト | ビジネスロジック | flutter_test |
| ウィジェットテスト | UIコンポーネント | flutter_test |
| 統合テスト | E2Eフロー | integration_test |
| Firebaseテスト | Security Rules | @firebase/rules-unit-testing |

### 10.2 テスト構成

```
test/
├── unit/                    # 単体テスト
│   ├── models/
│   ├── repositories/
│   └── services/
├── widget/                  # ウィジェットテスト
│   └── screens/
└── integration/             # 統合テスト
    └── flows/

functions/
└── test/                    # Cloud Functions テスト
    ├── rules.test.ts
    └── functions.test.ts
```

## 11. CI/CD

### 11.1 GitHub Actions ワークフロー

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.x'
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test

  build-android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build apk --release

  build-ios:
    needs: test
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
      - run: flutter build ios --release --no-codesign
```

## 12. 環境設定

### 12.1 環境変数

```dart
// lib/core/constants/env.dart
enum Environment { dev, stg, prod }

class EnvConfig {
  static Environment current = Environment.dev;

  static String get firebaseProjectId {
    switch (current) {
      case Environment.dev:
        return 'trainer-desk-dev';
      case Environment.stg:
        return 'trainer-desk-stg';
      case Environment.prod:
        return 'trainer-desk-prod';
    }
  }
}
```

### 12.2 Flavor設定（Android）

```groovy
// android/app/build.gradle
android {
    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            resValue "string", "app_name", "Trainer Desk Dev"
        }
        stg {
            dimension "environment"
            applicationIdSuffix ".stg"
            resValue "string", "app_name", "Trainer Desk Stg"
        }
        prod {
            dimension "environment"
            resValue "string", "app_name", "Trainer Desk"
        }
    }
}
```
