# データベース設計書

## 1. 概要

本ドキュメントは「Trainer Desk」アプリケーションのCloud Firestoreデータモデルを定義します。

## 2. コレクション構成

```
firestore/
├── users/                    # ユーザー
├── contracts/                # トレーナー・トレーニー契約
├── reservations/             # 予約
├── trainingMenus/            # トレーニングメニュー
├── trainingRecords/          # トレーニング記録
├── mealRecords/              # 食事記録
├── weightRecords/            # 体重記録
├── chatRooms/                # チャットルーム
│   └── {chatRoomId}/messages # メッセージ
└── notifications/            # 通知
```

## 3. コレクション詳細

### 3.1 users（ユーザー）

ユーザー情報を管理するコレクション。

#### ドキュメントID
Firebase Authentication の UID

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| email | string | ○ | メールアドレス |
| displayName | string | ○ | 表示名 |
| photoUrl | string | - | プロフィール画像URL |
| role | string | ○ | ロール（trainer/trainee） |
| bio | string | - | 自己紹介 |
| fcmToken | string | - | FCMトークン |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### トレーナー専用フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| specialties | array<string> | - | 専門分野 |
| certifications | array<string> | - | 資格 |
| hourlyRate | number | - | 時間単価（円） |
| location | string | - | 活動エリア |
| businessHours | map | - | 営業時間設定 |

#### トレーニー専用フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| goalWeight | number | - | 目標体重（kg） |
| height | number | - | 身長（cm） |

#### businessHours構造

```json
{
  "monday": { "start": "09:00", "end": "21:00", "isOpen": true },
  "tuesday": { "start": "09:00", "end": "21:00", "isOpen": true },
  "wednesday": { "start": "09:00", "end": "21:00", "isOpen": true },
  "thursday": { "start": "09:00", "end": "21:00", "isOpen": true },
  "friday": { "start": "09:00", "end": "21:00", "isOpen": true },
  "saturday": { "start": "10:00", "end": "18:00", "isOpen": true },
  "sunday": { "start": null, "end": null, "isOpen": false },
  "sessionDuration": 60
}
```

#### サンプルデータ

```json
{
  "email": "trainer@example.com",
  "displayName": "山田太郎",
  "photoUrl": "https://storage.googleapis.com/.../profile.jpg",
  "role": "trainer",
  "bio": "10年の指導経験を持つパーソナルトレーナーです。",
  "specialties": ["ダイエット", "筋力トレーニング", "姿勢改善"],
  "certifications": ["NSCA-CPT", "健康運動指導士"],
  "hourlyRate": 8000,
  "location": "東京都渋谷区",
  "businessHours": { ... },
  "fcmToken": "xxx",
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

---

### 3.2 contracts（契約）

トレーナーとトレーニーの契約関係を管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| trainerId | string | ○ | トレーナーのUID |
| traineeId | string | ○ | トレーニーのUID |
| status | string | ○ | ステータス |
| requestMessage | string | - | リクエストメッセージ |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |
| approvedAt | timestamp | - | 承認日時 |
| cancelledAt | timestamp | - | 解約日時 |

#### ステータス値

| 値 | 説明 |
|---|------|
| pending | 承認待ち |
| approved | 承認済み |
| rejected | 拒否 |
| cancelled | 解約済み |

#### インデックス

```
- trainerId + status (ASC)
- traineeId + status (ASC)
- trainerId + traineeId (複合)
```

---

### 3.3 reservations（予約）

トレーニング予約を管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| trainerId | string | ○ | トレーナーのUID |
| traineeId | string | ○ | トレーニーのUID |
| contractId | string | ○ | 契約ID |
| startTime | timestamp | ○ | 開始日時 |
| endTime | timestamp | ○ | 終了日時 |
| status | string | ○ | ステータス |
| note | string | - | 備考 |
| cancelReason | string | - | キャンセル理由 |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### ステータス値

| 値 | 説明 |
|---|------|
| pending | 承認待ち |
| confirmed | 確定 |
| rejected | 拒否 |
| cancelled | キャンセル |
| completed | 完了 |

#### インデックス

```
- trainerId + startTime (ASC)
- traineeId + startTime (ASC)
- trainerId + status + startTime (ASC)
- traineeId + status + startTime (ASC)
```

#### サンプルデータ

```json
{
  "trainerId": "trainer_uid_123",
  "traineeId": "trainee_uid_456",
  "contractId": "contract_id_789",
  "startTime": "2024-01-15T10:00:00Z",
  "endTime": "2024-01-15T11:00:00Z",
  "status": "confirmed",
  "note": "脚のトレーニング希望",
  "createdAt": "2024-01-10T00:00:00Z",
  "updatedAt": "2024-01-10T12:00:00Z"
}
```

---

### 3.4 trainingMenus（トレーニングメニュー）

トレーニングメニューテンプレートを管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| trainerId | string | ○ | 作成者（トレーナー）のUID |
| name | string | ○ | メニュー名 |
| description | string | - | 説明 |
| exercises | array<Exercise> | ○ | エクササイズリスト |
| isTemplate | boolean | ○ | テンプレートフラグ |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### Exercise構造

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| name | string | ○ | エクササイズ名 |
| sets | number | ○ | セット数 |
| reps | number | - | レップ数 |
| duration | number | - | 時間（秒） |
| restTime | number | - | 休憩時間（秒） |
| weight | number | - | 推奨重量（kg） |
| videoUrl | string | - | 参考動画URL |
| note | string | - | 備考 |

#### サンプルデータ

```json
{
  "trainerId": "trainer_uid_123",
  "name": "初心者向け全身トレーニング",
  "description": "初心者向けの基本的な全身トレーニングメニューです。",
  "exercises": [
    {
      "name": "スクワット",
      "sets": 3,
      "reps": 15,
      "restTime": 60,
      "note": "膝がつま先より前に出ないように注意"
    },
    {
      "name": "プランク",
      "sets": 3,
      "duration": 30,
      "restTime": 30,
      "note": "腰が落ちないように注意"
    }
  ],
  "isTemplate": true,
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

---

### 3.5 menuAssignments（メニュー割り当て）

トレーニーへのメニュー割り当てを管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| menuId | string | ○ | メニューID |
| trainerId | string | ○ | トレーナーのUID |
| traineeId | string | ○ | トレーニーのUID |
| startDate | timestamp | ○ | 開始日 |
| endDate | timestamp | - | 終了日 |
| frequency | number | - | 週の実施回数 |
| isActive | boolean | ○ | アクティブフラグ |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

---

### 3.6 trainingRecords（トレーニング記録）

トレーニング実績を記録するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| traineeId | string | ○ | トレーニーのUID |
| trainerId | string | - | トレーナーのUID（あれば） |
| menuId | string | - | 使用メニューID（あれば） |
| reservationId | string | - | 予約ID（あれば） |
| date | timestamp | ○ | 実施日 |
| exercises | array<ExerciseRecord> | ○ | エクササイズ記録 |
| condition | number | ○ | 体調（1-5） |
| note | string | - | メモ |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### ExerciseRecord構造

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| name | string | ○ | エクササイズ名 |
| sets | array<SetRecord> | ○ | セット記録 |

#### SetRecord構造

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| reps | number | - | 実施レップ数 |
| duration | number | - | 実施時間（秒） |
| weight | number | - | 使用重量（kg） |

#### インデックス

```
- traineeId + date (DESC)
- trainerId + traineeId + date (DESC)
```

---

### 3.7 mealRecords（食事記録）

食事記録を管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| traineeId | string | ○ | トレーニーのUID |
| date | timestamp | ○ | 食事日時 |
| mealType | string | ○ | 食事タイプ |
| content | string | ○ | 食事内容 |
| photoUrl | string | - | 写真URL |
| calories | number | - | カロリー（kcal） |
| protein | number | - | タンパク質（g） |
| fat | number | - | 脂質（g） |
| carbs | number | - | 炭水化物（g） |
| feedback | Feedback | - | トレーナーからのフィードバック |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### mealType値

| 値 | 説明 |
|---|------|
| breakfast | 朝食 |
| lunch | 昼食 |
| dinner | 夕食 |
| snack | 間食 |

#### Feedback構造

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| trainerId | string | ○ | トレーナーのUID |
| comment | string | ○ | コメント |
| createdAt | timestamp | ○ | 作成日時 |

#### インデックス

```
- traineeId + date (DESC)
- traineeId + mealType + date (DESC)
```

#### サンプルデータ

```json
{
  "traineeId": "trainee_uid_456",
  "date": "2024-01-15T08:00:00Z",
  "mealType": "breakfast",
  "content": "玄米、鶏むね肉のグリル、サラダ、味噌汁",
  "photoUrl": "https://storage.googleapis.com/.../meal.jpg",
  "calories": 450,
  "protein": 35,
  "fat": 10,
  "carbs": 55,
  "feedback": {
    "trainerId": "trainer_uid_123",
    "comment": "バランスの良い朝食ですね！タンパク質もしっかり摂れています。",
    "createdAt": "2024-01-15T10:00:00Z"
  },
  "createdAt": "2024-01-15T08:30:00Z",
  "updatedAt": "2024-01-15T10:00:00Z"
}
```

---

### 3.8 weightRecords（体重記録）

体重・体組成を記録するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| traineeId | string | ○ | トレーニーのUID |
| date | timestamp | ○ | 記録日 |
| weight | number | ○ | 体重（kg） |
| bodyFatPercentage | number | - | 体脂肪率（%） |
| muscleMass | number | - | 筋肉量（kg） |
| note | string | - | メモ |
| createdAt | timestamp | ○ | 作成日時 |

#### インデックス

```
- traineeId + date (DESC)
```

---

### 3.9 chatRooms（チャットルーム）

チャットルームを管理するコレクション。

#### ドキュメントID
自動生成（または `{trainerId}_{traineeId}` 形式）

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| participants | array<string> | ○ | 参加者のUID配列 |
| trainerId | string | ○ | トレーナーのUID |
| traineeId | string | ○ | トレーニーのUID |
| lastMessage | string | - | 最新メッセージ |
| lastMessageAt | timestamp | - | 最終メッセージ日時 |
| createdAt | timestamp | ○ | 作成日時 |
| updatedAt | timestamp | ○ | 更新日時 |

#### サブコレクション: messages

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| senderId | string | ○ | 送信者のUID |
| content | string | ○ | メッセージ内容 |
| type | string | ○ | メッセージタイプ |
| imageUrl | string | - | 画像URL |
| readBy | array<string> | ○ | 既読者のUID配列 |
| createdAt | timestamp | ○ | 送信日時 |

#### messageのtype値

| 値 | 説明 |
|---|------|
| text | テキストメッセージ |
| image | 画像メッセージ |

#### インデックス（messages）

```
- createdAt (DESC)
```

---

### 3.10 notifications（通知）

アプリ内通知を管理するコレクション。

#### ドキュメントID
自動生成

#### フィールド

| フィールド名 | 型 | 必須 | 説明 |
|------------|---|-----|------|
| userId | string | ○ | 通知先ユーザーのUID |
| type | string | ○ | 通知タイプ |
| title | string | ○ | タイトル |
| body | string | ○ | 本文 |
| data | map | - | 追加データ |
| isRead | boolean | ○ | 既読フラグ |
| createdAt | timestamp | ○ | 作成日時 |

#### type値

| 値 | 説明 |
|---|------|
| contract_request | 契約リクエスト |
| contract_approved | 契約承認 |
| contract_rejected | 契約拒否 |
| reservation_request | 予約リクエスト |
| reservation_confirmed | 予約確定 |
| reservation_rejected | 予約拒否 |
| reservation_cancelled | 予約キャンセル |
| reservation_reminder | 予約リマインダー |
| chat_message | チャットメッセージ |
| meal_feedback | 食事フィードバック |

#### インデックス

```
- userId + isRead + createdAt (DESC)
- userId + createdAt (DESC)
```

---

## 4. ER図

```
┌──────────────┐       ┌──────────────┐
│    users     │       │   contracts  │
│  (trainer)   │◄──────┤              │
└──────┬───────┘       │  trainerId   │
       │               │  traineeId   │
       │               │  status      │
       │               └──────┬───────┘
       │                      │
       │    ┌─────────────────┘
       │    │
       ▼    ▼
┌──────────────┐       ┌──────────────┐
│    users     │◄──────┤ reservations │
│  (trainee)   │       │              │
└──────┬───────┘       │  trainerId   │
       │               │  traineeId   │
       │               │  startTime   │
       │               └──────────────┘
       │
       ├──────────────────┐
       │                  │
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ mealRecords  │   │weightRecords │
│              │   │              │
│  traineeId   │   │  traineeId   │
│  date        │   │  date        │
│  content     │   │  weight      │
└──────────────┘   └──────────────┘

┌──────────────┐       ┌──────────────┐
│trainingMenus │◄──────┤menuAssignment│
│              │       │              │
│  trainerId   │       │  menuId      │
│  exercises   │       │  traineeId   │
└──────────────┘       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │trainingRecord│
                       │              │
                       │  traineeId   │
                       │  menuId      │
                       │  exercises   │
                       └──────────────┘

┌──────────────┐       ┌──────────────┐
│  chatRooms   │──────►│   messages   │
│              │       │ (subcollection)
│ participants │       │              │
│ lastMessage  │       │  senderId    │
└──────────────┘       │  content     │
                       └──────────────┘
```

## 5. データアクセスパターン

### 5.1 よく使用されるクエリ

#### トレーナーの予約一覧取得
```javascript
firestore
  .collection('reservations')
  .where('trainerId', '==', trainerId)
  .where('startTime', '>=', startOfDay)
  .where('startTime', '<=', endOfDay)
  .orderBy('startTime', 'asc')
```

#### トレーニーの食事記録一覧取得
```javascript
firestore
  .collection('mealRecords')
  .where('traineeId', '==', traineeId)
  .where('date', '>=', startOfWeek)
  .orderBy('date', 'desc')
  .limit(50)
```

#### 未読通知件数取得
```javascript
firestore
  .collection('notifications')
  .where('userId', '==', userId)
  .where('isRead', '==', false)
  .count()
```

### 5.2 リアルタイム監視が必要なコレクション

| コレクション | 用途 |
|------------|------|
| reservations | 予約ステータスの変更検知 |
| chatRooms/messages | 新着メッセージの受信 |
| notifications | 新着通知の受信 |

## 6. データ整合性

### 6.1 バッチ処理が必要な操作

1. **契約承認時**
   - contracts ステータス更新
   - chatRooms 作成
   - notifications 作成

2. **予約キャンセル時**
   - reservations ステータス更新
   - notifications 作成（相手への通知）

3. **アカウント削除時**
   - users 削除
   - 関連する contracts のステータス更新
   - 関連する reservations のキャンセル
