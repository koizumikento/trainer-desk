# 技術仕様書 - TrainerDesk

## 1. 技術スタック概要

### 1.1 フロントエンド（モバイルアプリ）
| 項目 | 技術 |
|------|------|
| フレームワーク | Flutter 3.x |
| 言語 | Dart 3.x |
| 状態管理 | Riverpod |
| ナビゲーション | GoRouter |
| UIライブラリ | Material 3 |
| フォーム管理 | flutter_form_builder + form_validators |
| アーキテクチャ | Clean Architecture + Feature-first |

### 1.2 バックエンド（Firebase）
| 項目 | サービス |
|------|----------|
| 認証 | Firebase Authentication |
| データベース | Cloud Firestore |
| ストレージ | Cloud Storage for Firebase |
| プッシュ通知 | Firebase Cloud Messaging (FCM) |
| サーバーレス関数 | Cloud Functions for Firebase |
| ホスティング | Firebase Hosting（Webダッシュボード用、将来対応） |

---

## 2. システムアーキテクチャ

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile App                                │
│                         (Flutter)                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │   Screens   │ │  Providers  │ │ Repositories│               │
│  │  (Widgets)  │ │  (Riverpod) │ │             │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FlutterFire SDK                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Firebase Services                           │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │    Auth      │ │  Firestore   │ │   Storage    │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
│  ┌──────────────┐ ┌──────────────┐                             │
│  │     FCM      │ │  Functions   │                             │
│  └──────────────┘ └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Firebase 設定

### 3.1 Firebase Authentication

```dart
// 認証プロバイダー設定
class AuthConfig {
  static const googleScopes = ['email', 'profile'];
}
```

### 3.2 Firestore セキュリティルール（概要）

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // ユーザーは自分のデータのみアクセス可能
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // トレーナーは自分のクライアントのデータにアクセス可能
    match /trainingSessions/{sessionId} {
      allow read: if isTrainerOrTrainee(resource.data);
      allow write: if isTrainer(resource.data.trainerId);
    }

    // チャットルームは参加者のみアクセス可能
    match /chatRooms/{roomId} {
      allow read, write: if request.auth.uid in resource.data.participants;
    }
  }
}
```

### 3.3 Cloud Storage セキュリティルール（概要）

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // プロフィール画像
    match /users/{userId}/profile/{fileName} {
      allow read: if request.auth != null;
      allow write: if request.auth.uid == userId;
    }

    // 食事画像
    match /meals/{mealId}/{fileName} {
      allow read: if isOwnerOrTrainer();
      allow write: if request.auth.uid == resource.metadata.userId;
    }

    // チャット画像
    match /chat/{roomId}/{fileName} {
      allow read, write: if isRoomParticipant(roomId);
    }
  }
}
```

---

## 4. API設計

### 4.1 Cloud Functions エンドポイント

| 関数名 | トリガー | 説明 |
|--------|----------|------|
| `onUserCreate` | Auth onCreate | ユーザー作成時の初期データ設定 |
| `sendReservationNotification` | Firestore onWrite | 予約ステータス変更時の通知送信 |
| `sendChatNotification` | Firestore onCreate | 新着メッセージ通知送信 |
| `sendMealFeedbackNotification` | Firestore onWrite | 食事フィードバック通知送信 |
| `generateInviteCode` | HTTPS callable | トレーナー招待コード生成 |
| `linkTrainerTrainee` | HTTPS callable | トレーナー・トレーニー紐付け |
| `cleanupExpiredInvites` | Scheduled | 期限切れ招待コード削除 |

### 4.2 リアルタイムリスナー

```dart
// チャットメッセージのリアルタイム監視
Stream<List<Message>> watchMessages(String roomId) {
  return FirebaseFirestore.instance
      .collection('chatRooms')
      .doc(roomId)
      .collection('messages')
      .orderBy('createdAt', descending: true)
      .limit(50)
      .snapshots()
      .map((snapshot) => snapshot.docs
          .map((doc) => Message.fromFirestore(doc))
          .toList());
}
```

---

## 5. プロジェクト構成

```
trainer_desk/
├── lib/
│   ├── main.dart                     # エントリーポイント
│   ├── app.dart                      # App Widget
│   │
│   ├── core/                         # コア機能
│   │   ├── constants/                # 定数
│   │   │   ├── app_colors.dart
│   │   │   ├── app_text_styles.dart
│   │   │   └── app_constants.dart
│   │   ├── extensions/               # 拡張メソッド
│   │   │   ├── context_extension.dart
│   │   │   └── datetime_extension.dart
│   │   ├── router/                   # ルーティング
│   │   │   └── app_router.dart
│   │   ├── theme/                    # テーマ設定
│   │   │   └── app_theme.dart
│   │   └── utils/                    # ユーティリティ
│   │       ├── validators.dart
│   │       └── formatters.dart
│   │
│   ├── data/                         # データ層
│   │   ├── models/                   # データモデル
│   │   │   ├── user_model.dart
│   │   │   ├── reservation_model.dart
│   │   │   ├── training_session_model.dart
│   │   │   ├── meal_model.dart
│   │   │   └── message_model.dart
│   │   ├── repositories/             # リポジトリ実装
│   │   │   ├── auth_repository.dart
│   │   │   ├── user_repository.dart
│   │   │   ├── reservation_repository.dart
│   │   │   ├── training_repository.dart
│   │   │   ├── meal_repository.dart
│   │   │   └── chat_repository.dart
│   │   └── services/                 # 外部サービス
│   │       ├── firebase_service.dart
│   │       ├── storage_service.dart
│   │       └── notification_service.dart
│   │
│   ├── domain/                       # ドメイン層
│   │   ├── entities/                 # エンティティ
│   │   │   ├── user.dart
│   │   │   ├── reservation.dart
│   │   │   ├── training_session.dart
│   │   │   ├── meal.dart
│   │   │   └── message.dart
│   │   └── repositories/             # リポジトリインターフェース
│   │       ├── i_auth_repository.dart
│   │       ├── i_user_repository.dart
│   │       └── ...
│   │
│   ├── presentation/                 # プレゼンテーション層
│   │   ├── providers/                # Riverpod Providers
│   │   │   ├── auth_provider.dart
│   │   │   ├── user_provider.dart
│   │   │   ├── reservation_provider.dart
│   │   │   ├── training_provider.dart
│   │   │   ├── meal_provider.dart
│   │   │   └── chat_provider.dart
│   │   ├── widgets/                  # 共通ウィジェット
│   │   │   ├── common/
│   │   │   │   ├── loading_widget.dart
│   │   │   │   ├── error_widget.dart
│   │   │   │   └── empty_state_widget.dart
│   │   │   ├── forms/
│   │   │   │   └── custom_text_field.dart
│   │   │   └── cards/
│   │   │       ├── reservation_card.dart
│   │   │       └── meal_card.dart
│   │   └── screens/                  # 画面
│   │       ├── auth/                 # 認証画面
│   │       │   ├── login_screen.dart
│   │       │   └── select_role_screen.dart
│   │       ├── trainer/              # トレーナー画面
│   │       │   ├── dashboard/
│   │       │   │   └── trainer_dashboard_screen.dart
│   │       │   ├── reservations/
│   │       │   │   ├── reservation_list_screen.dart
│   │       │   │   └── reservation_detail_screen.dart
│   │       │   ├── clients/
│   │       │   │   ├── client_list_screen.dart
│   │       │   │   └── client_detail_screen.dart
│   │       │   ├── training/
│   │       │   │   ├── training_menu_list_screen.dart
│   │       │   │   └── session_record_screen.dart
│   │       │   └── settings/
│   │       │       └── trainer_settings_screen.dart
│   │       ├── trainee/              # トレーニー画面
│   │       │   ├── dashboard/
│   │       │   │   └── trainee_dashboard_screen.dart
│   │       │   ├── reservations/
│   │       │   │   ├── booking_screen.dart
│   │       │   │   └── my_reservations_screen.dart
│   │       │   ├── training/
│   │       │   │   └── training_history_screen.dart
│   │       │   ├── meals/
│   │       │   │   ├── meal_record_screen.dart
│   │       │   │   └── meal_history_screen.dart
│   │       │   └── settings/
│   │       │       └── trainee_settings_screen.dart
│   │       └── common/               # 共通画面
│   │           ├── chat/
│   │           │   ├── chat_list_screen.dart
│   │           │   └── chat_room_screen.dart
│   │           └── notifications/
│   │               └── notification_list_screen.dart
│   │
│   └── firebase_options.dart         # Firebase設定（自動生成）
│
├── functions/                        # Cloud Functions
│   ├── src/
│   │   ├── index.ts
│   │   ├── auth/
│   │   │   └── on_user_create.ts
│   │   ├── notifications/
│   │   │   ├── reservation_notification.ts
│   │   │   ├── chat_notification.ts
│   │   │   └── meal_feedback_notification.ts
│   │   └── invitations/
│   │       ├── generate_invite_code.ts
│   │       └── link_trainer_trainee.ts
│   ├── package.json
│   └── tsconfig.json
│
├── assets/                           # アセット
│   ├── images/
│   ├── icons/
│   └── fonts/
│
├── test/                             # テスト
│   ├── unit/
│   ├── widget/
│   └── integration/
│
├── android/                          # Android固有設定
├── ios/                              # iOS固有設定
│
├── pubspec.yaml                      # パッケージ設定
├── analysis_options.yaml             # Lint設定
├── firebase.json                     # Firebase設定
├── firestore.rules                   # Firestoreルール
├── firestore.indexes.json            # Firestoreインデックス
├── storage.rules                     # Storageルール
└── docs/                             # ドキュメント
    ├── functional-spec.md
    ├── technical-spec.md
    └── data-model.md
```

---

## 6. 主要パッケージ

### 6.1 パッケージ一覧と説明

#### Firebase関連
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [firebase_core](https://pub.dev/packages/firebase_core) | ^3.9.0 | Firebase初期化に必須のコアパッケージ |
| [firebase_auth](https://pub.dev/packages/firebase_auth) | ^5.5.0 | Firebase認証（Google Sign-In等） |
| [cloud_firestore](https://pub.dev/packages/cloud_firestore) | ^5.6.0 | Firestoreデータベース操作 |
| [firebase_storage](https://pub.dev/packages/firebase_storage) | ^12.4.0 | Cloud Storage（画像アップロード等） |
| [firebase_messaging](https://pub.dev/packages/firebase_messaging) | ^15.2.0 | プッシュ通知（FCM） |
| [firebase_analytics](https://pub.dev/packages/firebase_analytics) | ^11.4.0 | ユーザー行動分析 |
| [firebase_crashlytics](https://pub.dev/packages/firebase_crashlytics) | ^4.3.0 | クラッシュレポート |

> **Note**: FlutterFireパッケージは互換性のあるバージョンを使用する必要があります。
> [VERSIONS.md](https://github.com/firebase/flutterfire/blob/main/VERSIONS.md) で確認してください。

#### 認証
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [google_sign_in](https://pub.dev/packages/google_sign_in) | ^6.2.2 | Googleアカウントでのサインイン |

#### 状態管理
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [flutter_riverpod](https://pub.dev/packages/flutter_riverpod) | ^3.0.0 | リアクティブな状態管理フレームワーク |
| [riverpod_annotation](https://pub.dev/packages/riverpod_annotation) | ^3.0.0 | Riverpodのアノテーション定義 |

> **Riverpod 3.0の主な変更点**:
> - StateProvider/StateNotifierProviderは`legacy.dart`に移動
> - オフライン永続化の実験的機能追加
> - 失敗したProviderの自動リトライ機能
> - 詳細: [What's new in Riverpod 3.0](https://riverpod.dev/docs/whats_new)

#### ナビゲーション
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [go_router](https://pub.dev/packages/go_router) | ^14.8.0 | 宣言的ルーティング（Navigation 2ベース） |

> **go_routerの特徴**:
> - ディープリンク対応
> - ShellRouteによる複数Navigator対応
> - URLベースのナビゲーション
> - Flutter公式推奨パッケージ

#### UI・表示
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [flutter_hooks](https://pub.dev/packages/flutter_hooks) | ^0.21.3 | Reactライクなフック（useState, useEffect等） |
| [cached_network_image](https://pub.dev/packages/cached_network_image) | ^3.4.1 | 画像のキャッシュ・表示 |
| [shimmer](https://pub.dev/packages/shimmer) | ^3.0.0 | ローディング時のシマーエフェクト |

#### フォーム
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [flutter_form_builder](https://pub.dev/packages/flutter_form_builder) | ^9.7.0 | フォーム構築の簡素化 |
| [form_builder_validators](https://pub.dev/packages/form_builder_validators) | ^11.1.0 | フォームバリデーション |

> **flutter_form_builderの主なウィジェット**:
> - FormBuilderTextField, FormBuilderDropdown
> - FormBuilderDateTimePicker, FormBuilderDateRangePicker
> - FormBuilderRadioGroup, FormBuilderChoiceChips
> - FormBuilderSlider, FormBuilderRangeSlider

#### 日付・カレンダー
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [intl](https://pub.dev/packages/intl) | ^0.20.2 | 国際化・日付フォーマット |
| [table_calendar](https://pub.dev/packages/table_calendar) | ^3.1.3 | 高機能カレンダーウィジェット |

> **table_calendarの特徴**:
> - 単一/複数/範囲選択対応
> - 横・縦スクロール対応
> - イベント表示・カスタムスタイリング
> - 多言語対応

#### 画像
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [image_picker](https://pub.dev/packages/image_picker) | ^1.1.2 | カメラ・ギャラリーから画像選択 |
| [image_cropper](https://pub.dev/packages/image_cropper) | ^8.0.2 | 画像のトリミング |

#### ローカルストレージ
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [shared_preferences](https://pub.dev/packages/shared_preferences) | ^2.5.2 | 簡易的なキー・バリューストレージ |
| [flutter_secure_storage](https://pub.dev/packages/flutter_secure_storage) | ^9.2.4 | 暗号化されたセキュアストレージ |

#### コード生成・ユーティリティ
| パッケージ | バージョン | 説明 |
|-----------|-----------|------|
| [freezed](https://pub.dev/packages/freezed) | ^3.0.5 | イミュータブルクラス・Union型生成 |
| [freezed_annotation](https://pub.dev/packages/freezed_annotation) | ^3.0.0 | freezedのアノテーション |
| [json_serializable](https://pub.dev/packages/json_serializable) | ^6.9.5 | JSONシリアライズ/デシリアライズ生成 |
| [json_annotation](https://pub.dev/packages/json_annotation) | ^4.9.0 | json_serializableのアノテーション |
| [build_runner](https://pub.dev/packages/build_runner) | ^2.4.15 | コード生成実行 |
| [uuid](https://pub.dev/packages/uuid) | ^4.5.1 | UUID生成 |

### 6.2 pubspec.yaml

```yaml
name: trainer_desk
description: Personal trainer management app
version: 1.0.0+1

environment:
  sdk: '>=3.6.0 <4.0.0'
  flutter: '>=3.27.0'

dependencies:
  flutter:
    sdk: flutter

  # Firebase
  firebase_core: ^3.9.0
  firebase_auth: ^5.5.0
  cloud_firestore: ^5.6.0
  firebase_storage: ^12.4.0
  firebase_messaging: ^15.2.0
  firebase_analytics: ^11.4.0
  firebase_crashlytics: ^4.3.0

  # Google Sign In
  google_sign_in: ^6.2.2

  # State Management
  flutter_riverpod: ^3.0.0
  riverpod_annotation: ^3.0.0

  # Navigation
  go_router: ^14.8.0

  # UI
  flutter_hooks: ^0.21.3
  cached_network_image: ^3.4.1
  shimmer: ^3.0.0

  # Forms
  flutter_form_builder: ^9.7.0
  form_builder_validators: ^11.1.0

  # Date & Time
  intl: ^0.20.2
  table_calendar: ^3.1.3

  # Image
  image_picker: ^1.1.2
  image_cropper: ^8.0.2

  # Local Storage
  shared_preferences: ^2.5.2
  flutter_secure_storage: ^9.2.4

  # Utilities
  freezed_annotation: ^3.0.0
  json_annotation: ^4.9.0
  uuid: ^4.5.1
  collection: ^1.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

  # Code Generation
  build_runner: ^2.4.15
  freezed: ^3.0.5
  json_serializable: ^6.9.5
  riverpod_generator: ^3.0.0

  # Testing
  mockito: ^5.4.5
  integration_test:
    sdk: flutter

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/icons/
```

### 6.3 analysis_options.yaml

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  errors:
    invalid_annotation_target: ignore  # freezed + json_serializable用
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"

linter:
  rules:
    prefer_single_quotes: true
    require_trailing_commas: true
    sort_pub_dependencies: true
```

### 6.4 Cloud Functions 依存関係

```json
{
  "dependencies": {
    "firebase-admin": "^13.0.0",
    "firebase-functions": "^6.3.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "firebase-functions-test": "^3.4.0"
  }
}
```

---

## 7. 認証フロー

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    User      │     │   App        │     │   Firebase   │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       │  1. Tap Login      │                    │
       │───────────────────>│                    │
       │                    │                    │
       │                    │ 2. Google Sign-In  │
       │                    │───────────────────>│
       │                    │                    │
       │  3. Google Auth    │                    │
       │<───────────────────│                    │
       │                    │                    │
       │  4. Grant          │                    │
       │───────────────────>│                    │
       │                    │                    │
       │                    │ 5. Verify Token    │
       │                    │───────────────────>│
       │                    │                    │
       │                    │ 6. Create/Get User │
       │                    │<───────────────────│
       │                    │                    │
       │                    │ 7. Check userType  │
       │                    │───────────────────>│
       │                    │                    │
       │  8. Role Select    │                    │
       │    (if new user)   │                    │
       │<───────────────────│                    │
       │                    │                    │
       │  9. Select Role    │                    │
       │───────────────────>│                    │
       │                    │                    │
       │                    │ 10. Save userType  │
       │                    │───────────────────>│
       │                    │                    │
       │  11. Dashboard     │                    │
       │<───────────────────│                    │
       │                    │                    │
```

### 7.1 認証実装例

```dart
// lib/data/repositories/auth_repository.dart
class AuthRepository implements IAuthRepository {
  final FirebaseAuth _auth;
  final GoogleSignIn _googleSignIn;
  final FirebaseFirestore _firestore;

  AuthRepository({
    FirebaseAuth? auth,
    GoogleSignIn? googleSignIn,
    FirebaseFirestore? firestore,
  })  : _auth = auth ?? FirebaseAuth.instance,
        _googleSignIn = googleSignIn ?? GoogleSignIn(),
        _firestore = firestore ?? FirebaseFirestore.instance;

  @override
  Stream<User?> get authStateChanges => _auth.authStateChanges();

  @override
  Future<UserCredential> signInWithGoogle() async {
    final googleUser = await _googleSignIn.signIn();
    if (googleUser == null) {
      throw AuthException('Google sign in cancelled');
    }

    final googleAuth = await googleUser.authentication;
    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );

    return await _auth.signInWithCredential(credential);
  }

  @override
  Future<void> signOut() async {
    await Future.wait([
      _auth.signOut(),
      _googleSignIn.signOut(),
    ]);
  }
}
```

---

## 8. 状態管理（Riverpod）

### 8.1 Provider 構成

```dart
// lib/presentation/providers/auth_provider.dart
@riverpod
class Auth extends _$Auth {
  @override
  Stream<User?> build() {
    return ref.watch(authRepositoryProvider).authStateChanges;
  }

  Future<void> signInWithGoogle() async {
    await ref.read(authRepositoryProvider).signInWithGoogle();
  }

  Future<void> signOut() async {
    await ref.read(authRepositoryProvider).signOut();
  }
}

// lib/presentation/providers/user_provider.dart
@riverpod
Future<AppUser?> currentUser(CurrentUserRef ref) async {
  final authUser = ref.watch(authProvider).value;
  if (authUser == null) return null;

  return ref.watch(userRepositoryProvider).getUser(authUser.uid);
}
```

### 8.2 AsyncValue パターン

```dart
// 画面でのデータ表示
class ReservationListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final reservationsAsync = ref.watch(reservationsProvider);

    return reservationsAsync.when(
      data: (reservations) => ListView.builder(
        itemCount: reservations.length,
        itemBuilder: (context, index) => ReservationCard(
          reservation: reservations[index],
        ),
      ),
      loading: () => const LoadingWidget(),
      error: (error, stack) => ErrorWidget(message: error.toString()),
    );
  }
}
```

---

## 9. ルーティング（GoRouter）

```dart
// lib/core/router/app_router.dart
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    initialLocation: '/login',
    redirect: (context, state) {
      final isLoggedIn = authState.value != null;
      final isLoginRoute = state.matchedLocation == '/login';
      final isSelectRoleRoute = state.matchedLocation == '/select-role';

      if (!isLoggedIn && !isLoginRoute) {
        return '/login';
      }

      if (isLoggedIn && isLoginRoute) {
        final user = ref.read(currentUserProvider).value;
        if (user == null || user.userType == null) {
          return '/select-role';
        }
        return user.userType == UserType.trainer
            ? '/trainer/dashboard'
            : '/trainee/dashboard';
      }

      return null;
    },
    routes: [
      // 認証
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/select-role',
        builder: (context, state) => const SelectRoleScreen(),
      ),

      // トレーナー
      ShellRoute(
        builder: (context, state, child) => TrainerShell(child: child),
        routes: [
          GoRoute(
            path: '/trainer/dashboard',
            builder: (context, state) => const TrainerDashboardScreen(),
          ),
          GoRoute(
            path: '/trainer/reservations',
            builder: (context, state) => const ReservationListScreen(),
          ),
          GoRoute(
            path: '/trainer/clients',
            builder: (context, state) => const ClientListScreen(),
          ),
          GoRoute(
            path: '/trainer/clients/:id',
            builder: (context, state) => ClientDetailScreen(
              clientId: state.pathParameters['id']!,
            ),
          ),
          // ... その他のルート
        ],
      ),

      // トレーニー
      ShellRoute(
        builder: (context, state, child) => TraineeShell(child: child),
        routes: [
          GoRoute(
            path: '/trainee/dashboard',
            builder: (context, state) => const TraineeDashboardScreen(),
          ),
          GoRoute(
            path: '/trainee/reservations',
            builder: (context, state) => const MyReservationsScreen(),
          ),
          GoRoute(
            path: '/trainee/meals',
            builder: (context, state) => const MealHistoryScreen(),
          ),
          // ... その他のルート
        ],
      ),

      // 共通
      GoRoute(
        path: '/chat',
        builder: (context, state) => const ChatListScreen(),
      ),
      GoRoute(
        path: '/chat/:roomId',
        builder: (context, state) => ChatRoomScreen(
          roomId: state.pathParameters['roomId']!,
        ),
      ),
    ],
  );
});
```

---

## 10. プッシュ通知設計

### 10.1 通知トピック

| トピック | 対象 | 説明 |
|----------|------|------|
| `user_{userId}` | 個人 | ユーザー個別通知 |
| `trainer_{trainerId}_clients` | グループ | トレーナーの全クライアント |

### 10.2 通知ペイロード構造

```dart
class NotificationPayload {
  final String title;
  final String body;
  final NotificationType type;
  final String targetId;
  final String action;

  NotificationPayload({
    required this.title,
    required this.body,
    required this.type,
    required this.targetId,
    required this.action,
  });
}

enum NotificationType {
  reservation,
  chat,
  mealFeedback,
  reminder,
}
```

### 10.3 FCM 初期化

```dart
// lib/data/services/notification_service.dart
class NotificationService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // 権限リクエスト
    await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // フォアグラウンド通知設定
    await FirebaseMessaging.instance.setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );

    // トークン取得・保存
    final token = await _messaging.getToken();
    if (token != null) {
      await _saveToken(token);
    }

    // トークン更新監視
    _messaging.onTokenRefresh.listen(_saveToken);
  }

  Future<void> _saveToken(String token) async {
    // Firestoreにトークンを保存
  }
}
```

---

## 11. エラーハンドリング

### 11.1 エラーコード

| コード | 説明 |
|--------|------|
| `AUTH_001` | 認証エラー |
| `AUTH_002` | セッション期限切れ |
| `DATA_001` | データ取得エラー |
| `DATA_002` | データ保存エラー |
| `UPLOAD_001` | ファイルアップロードエラー |
| `NETWORK_001` | ネットワークエラー |
| `INVITE_001` | 招待コード無効 |
| `INVITE_002` | 招待コード期限切れ |

### 11.2 例外クラス

```dart
// lib/core/exceptions/app_exception.dart
sealed class AppException implements Exception {
  final String code;
  final String message;

  AppException(this.code, this.message);
}

class AuthException extends AppException {
  AuthException(String message) : super('AUTH_001', message);
}

class DataException extends AppException {
  DataException(String message) : super('DATA_001', message);
}

class NetworkException extends AppException {
  NetworkException() : super('NETWORK_001', 'ネットワークエラーが発生しました');
}
```

### 11.3 グローバルエラーハンドラー

```dart
// lib/core/utils/error_handler.dart
class ErrorHandler {
  static void handle(Object error, StackTrace stackTrace) {
    // Crashlyticsにログ送信
    FirebaseCrashlytics.instance.recordError(error, stackTrace);

    // ユーザーへの通知
    if (error is AppException) {
      _showErrorSnackBar(error.message);
    } else {
      _showErrorSnackBar('予期せぬエラーが発生しました');
    }
  }

  static void _showErrorSnackBar(String message) {
    // SnackBar表示
  }
}
```

---

## 12. テスト戦略

### 12.1 テストレベル

| レベル | ツール | 対象 |
|--------|--------|------|
| ユニットテスト | flutter_test | Repositories, Providers, Utils |
| ウィジェットテスト | flutter_test | UIコンポーネント |
| インテグレーションテスト | integration_test | ユーザーフロー |

### 12.2 テスト例

```dart
// test/unit/repositories/auth_repository_test.dart
void main() {
  late AuthRepository authRepository;
  late MockFirebaseAuth mockFirebaseAuth;
  late MockGoogleSignIn mockGoogleSignIn;

  setUp(() {
    mockFirebaseAuth = MockFirebaseAuth();
    mockGoogleSignIn = MockGoogleSignIn();
    authRepository = AuthRepository(
      auth: mockFirebaseAuth,
      googleSignIn: mockGoogleSignIn,
    );
  });

  group('signInWithGoogle', () {
    test('should return UserCredential on success', () async {
      // Arrange
      when(mockGoogleSignIn.signIn()).thenAnswer(
        (_) async => MockGoogleSignInAccount(),
      );

      // Act
      final result = await authRepository.signInWithGoogle();

      // Assert
      expect(result, isA<UserCredential>());
    });
  });
}
```

### 12.3 Cloud Functions テスト

```typescript
// Firebase Functions テスト例
import * as test from 'firebase-functions-test';

const testEnv = test();

describe('onUserCreate', () => {
  it('should create initial user data', async () => {
    const user = { uid: 'test-uid', email: 'test@example.com' };
    const wrapped = testEnv.wrap(onUserCreate);
    await wrapped(user);
    // Firestoreにデータが作成されたことを確認
  });
});
```

---

## 13. デプロイメント

### 13.1 環境

| 環境 | 用途 | Firebase Project |
|------|------|------------------|
| development | 開発・デバッグ | trainer-desk-dev |
| staging | テスト・QA | trainer-desk-stg |
| production | 本番 | trainer-desk-prod |

### 13.2 Flavor 設定

```dart
// lib/main_development.dart
void main() {
  runApp(const App(flavor: Flavor.development));
}

// lib/main_staging.dart
void main() {
  runApp(const App(flavor: Flavor.staging));
}

// lib/main_production.dart
void main() {
  runApp(const App(flavor: Flavor.production));
}
```

### 13.3 CI/CD パイプライン

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy-functions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: cd functions && npm ci
      - run: cd functions && npm run build
      - run: firebase deploy --only functions

  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
      - run: flutter pub get
      - run: flutter build appbundle --release

  build-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
      - run: flutter pub get
      - run: flutter build ios --release --no-codesign
```

---

## 14. 監視・ログ

### 14.1 Firebase サービス

| サービス | 用途 |
|----------|------|
| Firebase Analytics | ユーザー行動分析 |
| Firebase Crashlytics | クラッシュレポート |
| Firebase Performance | パフォーマンス監視 |
| Cloud Logging | サーバーログ |

### 14.2 カスタムイベント

```dart
// lib/core/utils/analytics_service.dart
class AnalyticsService {
  final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;

  Future<void> logReservationCreated({
    required String trainerId,
    required DateTime date,
  }) async {
    await _analytics.logEvent(
      name: 'reservation_created',
      parameters: {
        'trainer_id': trainerId,
        'date': date.toIso8601String(),
      },
    );
  }

  Future<void> logMealRecorded({
    required String mealType,
  }) async {
    await _analytics.logEvent(
      name: 'meal_recorded',
      parameters: {
        'meal_type': mealType,
      },
    );
  }
}
```
