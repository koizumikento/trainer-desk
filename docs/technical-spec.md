# 技術仕様書 - TrainerDesk

## 1. 技術スタック概要

### 1.1 フロントエンド（モバイルアプリ）
| 項目 | 技術 |
|------|------|
| フレームワーク | React Native (Expo) |
| 言語 | TypeScript |
| 状態管理 | Zustand |
| ナビゲーション | React Navigation v6 |
| UIライブラリ | React Native Paper / NativeBase |
| フォーム管理 | React Hook Form + Zod |

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
│                    (React Native + Expo)                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │   Screens   │ │   Hooks     │ │   Stores    │               │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Firebase SDK                                │
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

```typescript
// 認証プロバイダー設定
const authProviders = {
  google: {
    enabled: true,
    scopes: ['email', 'profile']
  }
};
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

```typescript
// チャットメッセージのリアルタイム監視
const subscribeToMessages = (roomId: string, callback: (messages: Message[]) => void) => {
  return firestore()
    .collection('chatRooms')
    .doc(roomId)
    .collection('messages')
    .orderBy('createdAt', 'desc')
    .limit(50)
    .onSnapshot(snapshot => {
      const messages = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      callback(messages);
    });
};
```

---

## 5. プロジェクト構成

```
trainer-desk/
├── app/                          # Expo Router のページ
│   ├── (auth)/                   # 認証関連画面
│   │   ├── login.tsx
│   │   └── select-role.tsx
│   ├── (trainer)/                # トレーナー専用画面
│   │   ├── dashboard.tsx
│   │   ├── reservations/
│   │   ├── clients/
│   │   ├── training/
│   │   └── settings.tsx
│   ├── (trainee)/                # トレーニー専用画面
│   │   ├── dashboard.tsx
│   │   ├── reservations/
│   │   ├── training/
│   │   ├── meals/
│   │   └── settings.tsx
│   ├── (common)/                 # 共通画面
│   │   ├── chat/
│   │   └── notifications.tsx
│   └── _layout.tsx
├── src/
│   ├── components/               # 共通コンポーネント
│   │   ├── ui/                   # 基本UIコンポーネント
│   │   ├── forms/                # フォームコンポーネント
│   │   └── cards/                # カード系コンポーネント
│   ├── hooks/                    # カスタムフック
│   │   ├── useAuth.ts
│   │   ├── useReservations.ts
│   │   ├── useTraining.ts
│   │   ├── useMeals.ts
│   │   └── useChat.ts
│   ├── stores/                   # Zustand ストア
│   │   ├── authStore.ts
│   │   ├── reservationStore.ts
│   │   └── chatStore.ts
│   ├── services/                 # Firebase サービス
│   │   ├── firebase.ts           # Firebase 初期化
│   │   ├── auth.ts               # 認証サービス
│   │   ├── firestore.ts          # Firestore操作
│   │   ├── storage.ts            # Storage操作
│   │   └── messaging.ts          # FCM設定
│   ├── types/                    # TypeScript 型定義
│   │   ├── user.ts
│   │   ├── reservation.ts
│   │   ├── training.ts
│   │   ├── meal.ts
│   │   └── chat.ts
│   ├── utils/                    # ユーティリティ関数
│   │   ├── date.ts
│   │   ├── validation.ts
│   │   └── format.ts
│   └── constants/                # 定数定義
│       ├── config.ts
│       └── theme.ts
├── functions/                    # Cloud Functions
│   ├── src/
│   │   ├── index.ts
│   │   ├── auth/
│   │   ├── notifications/
│   │   └── invitations/
│   ├── package.json
│   └── tsconfig.json
├── assets/                       # 画像・フォント等
├── app.json                      # Expo設定
├── package.json
├── tsconfig.json
├── firebase.json                 # Firebase設定
├── firestore.rules               # Firestoreルール
├── storage.rules                 # Storageルール
└── docs/                         # ドキュメント
    ├── functional-spec.md
    ├── technical-spec.md
    └── data-model.md
```

---

## 6. 主要ライブラリ

### 6.1 フロントエンド依存関係

```json
{
  "dependencies": {
    "expo": "~50.0.0",
    "expo-router": "~3.4.0",
    "react": "18.2.0",
    "react-native": "0.73.0",
    "@react-native-firebase/app": "^18.0.0",
    "@react-native-firebase/auth": "^18.0.0",
    "@react-native-firebase/firestore": "^18.0.0",
    "@react-native-firebase/storage": "^18.0.0",
    "@react-native-firebase/messaging": "^18.0.0",
    "@react-native-google-signin/google-signin": "^11.0.0",
    "@react-navigation/native": "^6.1.0",
    "zustand": "^4.5.0",
    "react-hook-form": "^7.50.0",
    "zod": "^3.22.0",
    "date-fns": "^3.3.0",
    "react-native-calendars": "^1.1300.0",
    "react-native-image-picker": "^7.1.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "~18.2.0",
    "eslint": "^8.56.0",
    "prettier": "^3.2.0"
  }
}
```

### 6.2 Cloud Functions 依存関係

```json
{
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^4.5.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "firebase-functions-test": "^3.1.0"
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

---

## 8. プッシュ通知設計

### 8.1 通知トピック

| トピック | 対象 | 説明 |
|----------|------|------|
| `user_{userId}` | 個人 | ユーザー個別通知 |
| `trainer_{trainerId}_clients` | グループ | トレーナーの全クライアント |

### 8.2 通知ペイロード構造

```typescript
interface NotificationPayload {
  notification: {
    title: string;
    body: string;
  };
  data: {
    type: 'reservation' | 'chat' | 'meal_feedback' | 'reminder';
    targetId: string;
    action: string;
  };
}
```

---

## 9. エラーハンドリング

### 9.1 エラーコード

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

### 9.2 グローバルエラーハンドラー

```typescript
const handleError = (error: AppError) => {
  // エラーログ送信（Firebase Crashlytics）
  crashlytics().recordError(error);

  // ユーザーへの通知
  showErrorToast(getErrorMessage(error.code));
};
```

---

## 10. テスト戦略

### 10.1 テストレベル

| レベル | ツール | 対象 |
|--------|--------|------|
| ユニットテスト | Jest | Hooks, Utils, Stores |
| コンポーネントテスト | React Native Testing Library | UIコンポーネント |
| E2Eテスト | Detox | ユーザーフロー |

### 10.2 Cloud Functions テスト

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

## 11. デプロイメント

### 11.1 環境

| 環境 | 用途 |
|------|------|
| development | 開発・デバッグ |
| staging | テスト・QA |
| production | 本番 |

### 11.2 CI/CD パイプライン

```yaml
# GitHub Actions 例
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
      - run: npm ci
      - run: npm run build
      - run: firebase deploy --only functions

  build-app:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: expo/expo-github-action@v8
      - run: eas build --platform all --non-interactive
```

---

## 12. 監視・ログ

### 12.1 Firebase サービス

| サービス | 用途 |
|----------|------|
| Firebase Analytics | ユーザー行動分析 |
| Firebase Crashlytics | クラッシュレポート |
| Firebase Performance | パフォーマンス監視 |
| Cloud Logging | サーバーログ |

### 12.2 カスタムイベント

```typescript
// 重要なユーザーアクションをトラッキング
analytics().logEvent('reservation_created', {
  trainer_id: trainerId,
  date: reservationDate,
});
```
