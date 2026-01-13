# データモデル仕様書 - TrainerDesk

## 1. ER図（Entity Relationship Diagram）

![ER図](diagrams/er-diagram.puml)

---

## 2. Firestore コレクション構造

```
firestore/
├── users/                        # ユーザー情報
│   └── {userId}/
│       ├── (document fields)
│       └── fcmTokens/            # FCMトークン（サブコレクション）
├── trainerProfiles/              # トレーナープロフィール
│   └── {trainerId}/
├── traineeProfiles/              # トレーニープロフィール
│   └── {traineeId}/
├── bodyMeasurements/             # 体重・身長記録（時系列）
│   └── {measurementId}/
├── trainerTraineeLinks/          # トレーナー・トレーニー紐付け
│   └── {linkId}/
├── tickets/                      # 回数券
│   └── {ticketId}/
├── invitations/                  # 招待コード
│   └── {invitationId}/
├── reservations/                 # 予約
│   └── {reservationId}/
├── trainerSchedules/             # トレーナースケジュール設定
│   └── {trainerId}/
├── exerciseMaster/               # エクササイズマスターデータ（読み取り専用）
│   └── {exerciseId}/
├── trainingMenus/                # トレーニングメニューテンプレート
│   └── {menuId}/
├── trainingSessions/             # トレーニングセッション記録
│   └── {sessionId}/
├── meals/                        # 食事記録
│   └── {mealId}/
├── mealFeedbacks/                # 食事フィードバック
│   └── {feedbackId}/
├── chatRooms/                    # チャットルーム
│   └── {roomId}/
│       └── messages/             # メッセージ（サブコレクション）
│           └── {messageId}/
└── notifications/                # 通知履歴
    └── {notificationId}/
```

---

## 3. コレクション詳細

### 3.1 users（ユーザー）

基本的なユーザー情報を格納。認証後に作成される。

<details>
<summary>TypeScript型定義</summary>

```typescript
type UserRole = 'trainer' | 'trainee';

interface User {
  // ドキュメントID: Firebase Auth UID
  id: string;

  // 基本情報
  email: string;
  displayName: string;
  photoURL: string | null;

  // ユーザー種別（複数選択可）
  roles: UserRole[];                    // 持っている役割（1つ以上）
  activeRole: UserRole;                 // 現在アクティブな役割

  // ステータス
  isActive: boolean;

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
  lastLoginAt: Timestamp;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/user.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'user.freezed.dart';
part 'user.g.dart';

enum UserRole {
  @JsonValue('trainer')
  trainer,
  @JsonValue('trainee')
  trainee,
}

@freezed
class AppUser with _$AppUser {
  const AppUser._();

  const factory AppUser({
    required String id,
    required String email,
    required String displayName,
    String? photoURL,
    required List<UserRole> roles,        // 持っている役割（複数可）
    required UserRole activeRole,          // 現在アクティブな役割
    @Default(true) bool isActive,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
    @TimestampConverter() required DateTime lastLoginAt,
  }) = _AppUser;

  factory AppUser.fromJson(Map<String, dynamic> json) =>
      _$AppUserFromJson(json);

  factory AppUser.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return AppUser.fromJson({...data, 'id': doc.id});
  }

  // 便利ゲッター
  bool get isTrainer => roles.contains(UserRole.trainer);
  bool get isTrainee => roles.contains(UserRole.trainee);
  bool get hasBothRoles => isTrainer && isTrainee;
}

// Timestamp変換用コンバーター
class TimestampConverter implements JsonConverter<DateTime, Timestamp> {
  const TimestampConverter();

  @override
  DateTime fromJson(Timestamp timestamp) => timestamp.toDate();

  @override
  Timestamp toJson(DateTime date) => Timestamp.fromDate(date);
}
```
</details>

**サブコレクション: fcmTokens**

<details>
<summary>TypeScript型定義</summary>

```typescript
interface FcmToken {
  // ドキュメントID: 自動生成
  token: string;
  deviceType: 'ios' | 'android';
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/fcm_token.dart
@freezed
class FcmToken with _$FcmToken {
  const factory FcmToken({
    required String token,
    required DeviceType deviceType,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
  }) = _FcmToken;

  factory FcmToken.fromJson(Map<String, dynamic> json) =>
      _$FcmTokenFromJson(json);
}

enum DeviceType {
  @JsonValue('ios')
  ios,
  @JsonValue('android')
  android,
}
```
</details>

---

### 3.2 trainerProfiles（トレーナープロフィール）

```typescript
interface TrainerProfile {
  // ドキュメントID: userId と同じ
  userId: string;

  // プロフィール情報
  bio: string;                          // 自己紹介
  specialties: string[];                // 専門分野
  certifications: string[];             // 資格
  experienceYears: number;              // 経験年数

  // 料金情報
  pricing: {
    sessionDuration: number;            // セッション時間（分）
    pricePerSession: number;            // 1セッションあたり料金
    currency: 'JPY';
  }[];

  // 連絡先（任意公開）
  contactInfo: {
    phone?: string;
    website?: string;
    instagram?: string;
  };

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

---

### 3.3 traineeProfiles（トレーニープロフィール）

```typescript
interface TraineeProfile {
  // ドキュメントID: userId と同じ
  userId: string;

  // 身体情報
  height: number | null;                // 身長（cm）
  weight: number | null;                // 体重（kg）
  birthDate: Timestamp | null;          // 生年月日

  // 目標・備考
  goals: string[];                      // 目標
  medicalHistory: string;               // 既往歴・注意事項
  notes: string;                        // その他メモ

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

---

### 3.4 bodyMeasurements（体重・身長記録）

ユーザーの体重・身長を時系列で記録。手動入力またはHealth Connect / HealthKitから自動取得。

<details>
<summary>TypeScript型定義</summary>

```typescript
type MeasurementSource = 'manual' | 'health_connect' | 'healthkit';

interface BodyMeasurement {
  // ドキュメントID: 自動生成
  id: string;

  // 対象ユーザー
  userId: string;

  // 測定データ
  weight: number | null;              // 体重（kg）
  height: number | null;              // 身長（cm）
  bodyFatPercentage: number | null;   // 体脂肪率（%）
  muscleMass: number | null;          // 筋肉量（kg）
  bmi: number | null;                 // BMI（自動計算可）

  // メタ情報
  measuredAt: Timestamp;              // 測定日時
  source: MeasurementSource;          // データソース
  note: string;                       // メモ

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/body_measurement.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'body_measurement.freezed.dart';
part 'body_measurement.g.dart';

enum MeasurementSource {
  @JsonValue('manual')
  manual,
  @JsonValue('health_connect')
  healthConnect,
  @JsonValue('healthkit')
  healthkit,
}

@freezed
class BodyMeasurement with _$BodyMeasurement {
  const BodyMeasurement._();

  const factory BodyMeasurement({
    required String id,
    required String userId,
    double? weight,                    // 体重（kg）
    double? height,                    // 身長（cm）
    double? bodyFatPercentage,         // 体脂肪率（%）
    double? muscleMass,                // 筋肉量（kg）
    double? bmi,                       // BMI
    @TimestampConverter() required DateTime measuredAt,
    required MeasurementSource source,
    @Default('') String note,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
  }) = _BodyMeasurement;

  factory BodyMeasurement.fromJson(Map<String, dynamic> json) =>
      _$BodyMeasurementFromJson(json);

  factory BodyMeasurement.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return BodyMeasurement.fromJson({...data, 'id': doc.id});
  }

  // BMI自動計算（height, weightがある場合）
  double? get calculatedBmi {
    if (weight != null && height != null && height! > 0) {
      final heightM = height! / 100;
      return weight! / (heightM * heightM);
    }
    return null;
  }
}
```
</details>

**インデックス:**
- `userId` + `measuredAt`
- `userId` + `source` + `measuredAt`

---

### 3.5 trainerTraineeLinks（トレーナー・トレーニー紐付け）

```typescript
interface TrainerTraineeLink {
  // ドキュメントID: 自動生成
  trainerId: string;
  traineeId: string;

  // ステータス
  status: 'active' | 'inactive';

  // タイムスタンプ
  linkedAt: Timestamp;
  unlinkedAt: Timestamp | null;
}
```

**インデックス:**
- `trainerId` + `status`
- `traineeId` + `status`

---

### 3.6 tickets（回数券）

トレーナーがクライアントに発行する回数券。予約完了時に自動消化。

<details>
<summary>TypeScript型定義</summary>

```typescript
interface Ticket {
  // ドキュメントID: 自動生成
  id: string;

  // 関係者
  trainerId: string;
  traineeId: string;

  // 回数券情報
  totalCount: number;           // 購入枚数
  usedCount: number;            // 使用済み枚数
  pricePerSession: number;      // 1回あたり単価（円）
  totalPrice: number;           // 総額（円）

  // ステータス
  status: 'active' | 'exhausted' | 'expired';

  // 有効期限（任意）
  expiresAt: Timestamp | null;

  // メモ
  note: string;

  // タイムスタンプ
  purchasedAt: Timestamp;
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/ticket.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'ticket.freezed.dart';
part 'ticket.g.dart';

enum TicketStatus {
  @JsonValue('active')
  active,
  @JsonValue('exhausted')
  exhausted,
  @JsonValue('expired')
  expired,
}

@freezed
class Ticket with _$Ticket {
  const Ticket._();

  const factory Ticket({
    required String id,
    required String trainerId,
    required String traineeId,
    required int totalCount,
    @Default(0) int usedCount,
    required int pricePerSession,
    required int totalPrice,
    @Default(TicketStatus.active) TicketStatus status,
    @TimestampNullableConverter() DateTime? expiresAt,
    @Default('') String note,
    @TimestampConverter() required DateTime purchasedAt,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
  }) = _Ticket;

  factory Ticket.fromJson(Map<String, dynamic> json) =>
      _$TicketFromJson(json);

  factory Ticket.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Ticket.fromJson({...data, 'id': doc.id});
  }

  // 残り回数
  int get remainingCount => totalCount - usedCount;

  // 使い切ったか
  bool get isExhausted => remainingCount <= 0;

  // 有効期限切れか
  bool get isExpired => expiresAt != null && DateTime.now().isAfter(expiresAt!);

  // 使用可能か
  bool get isUsable => status == TicketStatus.active && !isExhausted && !isExpired;
}
```
</details>

**インデックス:**
- `trainerId` + `traineeId` + `status`
- `traineeId` + `status`

---

### 3.7 invitations（招待）

```typescript
interface Invitation {
  // ドキュメントID: 招待コード（6桁英数字）
  code: string;

  // 招待元
  trainerId: string;
  trainerName: string;

  // ステータス
  status: 'pending' | 'used' | 'expired';
  usedBy: string | null;                // 使用したトレーニーID

  // 有効期限
  expiresAt: Timestamp;

  // タイムスタンプ
  createdAt: Timestamp;
  usedAt: Timestamp | null;
}
```

---

### 3.8 trainerSchedules（トレーナースケジュール）

```typescript
interface TrainerSchedule {
  // ドキュメントID: trainerId
  trainerId: string;

  // 営業時間（曜日別）
  weeklySchedule: {
    [key in DayOfWeek]: {
      isAvailable: boolean;
      slots: {
        startTime: string;              // "09:00"
        endTime: string;                // "18:00"
      }[];
    };
  };

  // 定休日・休暇
  holidays: {
    date: Timestamp;
    reason?: string;
  }[];

  // セッション設定
  sessionDuration: number;              // デフォルトセッション時間（分）
  bufferTime: number;                   // セッション間のバッファ（分）

  // タイムスタンプ
  updatedAt: Timestamp;
}

type DayOfWeek = 'monday' | 'tuesday' | 'wednesday' | 'thursday' | 'friday' | 'saturday' | 'sunday';
```

---

### 3.9 exerciseMaster（エクササイズマスター）

アプリに組み込みのエクササイズ・マシンのマスターデータ。読み取り専用。

```typescript
type ExerciseCategory = 'chest' | 'back' | 'legs' | 'arms' | 'shoulders' | 'core' | 'cardio' | 'other';
type ExerciseType = 'machine' | 'free_weight' | 'bodyweight' | 'cable' | 'cardio';

interface ExerciseMaster {
  // ドキュメントID: 自動生成
  id: string;

  // 基本情報
  name: string;                           // 例: "チェストプレス"
  nameEn: string;                         // 例: "Chest Press"
  description: string;                    // 説明文

  // 分類
  category: ExerciseCategory;             // 部位カテゴリ
  type: ExerciseType;                     // 種別
  targetMuscles: string[];                // 対象筋肉群

  // 表示設定
  iconUrl: string | null;                 // アイコン画像
  sortOrder: number;                      // 表示順

  // メタ情報
  isPopular: boolean;                     // よく使われるか（優先表示用）
  gymBrands: string[];                    // 対応ジム（例: ["chocoZAP", "ANYTIME"]）
}
```

**プリセットデータ例:**

#### 胸（chest）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| チェストプレス | Chest Press | machine | chocoZAP, ANYTIME |
| ペックフライ | Pec Fly | machine | ANYTIME |
| ベンチプレス | Bench Press | free_weight | ANYTIME |
| ダンベルプレス | Dumbbell Press | free_weight | ANYTIME |
| ダンベルフライ | Dumbbell Fly | free_weight | ANYTIME |
| インクラインベンチプレス | Incline Bench Press | free_weight | ANYTIME |
| プッシュアップ | Push Up | bodyweight | - |
| ディップス | Dips | bodyweight | chocoZAP |
| ケーブルクロスオーバー | Cable Crossover | cable | ANYTIME |

#### 背中（back）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| ラットプルダウン | Lat Pulldown | machine | chocoZAP, ANYTIME |
| シーテッドロー | Seated Row | machine | ANYTIME |
| チンニング（懸垂） | Chin Up / Pull Up | bodyweight | ANYTIME |
| ベントオーバーロー | Bent Over Row | free_weight | ANYTIME |
| ワンハンドロー | One Hand Row | free_weight | ANYTIME |
| デッドリフト | Deadlift | free_weight | ANYTIME |
| ケーブルロー | Cable Row | cable | ANYTIME |

#### 脚（legs）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| レッグプレス | Leg Press | machine | chocoZAP, ANYTIME |
| レッグエクステンション | Leg Extension | machine | ANYTIME |
| レッグカール | Leg Curl | machine | ANYTIME |
| アダクション | Adduction | machine | chocoZAP |
| アブダクション | Abduction | machine | chocoZAP |
| ハックスクワット | Hack Squat | machine | ANYTIME |
| スクワット | Squat | free_weight | ANYTIME |
| ダンベルスクワット | Dumbbell Squat | free_weight | - |
| ランジ | Lunge | bodyweight | - |
| ブルガリアンスクワット | Bulgarian Squat | bodyweight | - |
| カーフレイズ | Calf Raise | bodyweight | - |

#### 肩（shoulders）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| ショルダープレス | Shoulder Press | machine | chocoZAP, ANYTIME |
| ダンベルショルダープレス | Dumbbell Shoulder Press | free_weight | ANYTIME |
| サイドレイズ | Side Raise | free_weight | - |
| フロントレイズ | Front Raise | free_weight | - |
| リアレイズ | Rear Raise | free_weight | - |
| アップライトロー | Upright Row | free_weight | ANYTIME |

#### 腕（arms）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| アームカール | Arm Curl | machine | chocoZAP |
| ケーブルカール | Cable Curl | cable | ANYTIME |
| ダンベルカール | Dumbbell Curl | free_weight | ANYTIME |
| バーベルカール | Barbell Curl | free_weight | ANYTIME |
| ハンマーカール | Hammer Curl | free_weight | - |
| トライセプスプレスダウン | Triceps Pressdown | cable | ANYTIME |
| トライセプスエクステンション | Triceps Extension | free_weight | ANYTIME |
| スカルクラッシャー | Skull Crusher | free_weight | ANYTIME |

#### 腹筋・体幹（core）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| アブドミナル | Abdominal | machine | chocoZAP |
| ケーブルクランチ | Cable Crunch | cable | ANYTIME |
| クランチ | Crunch | bodyweight | - |
| シットアップ | Sit Up | bodyweight | - |
| レッグレイズ | Leg Raise | bodyweight | - |
| プランク | Plank | bodyweight | - |
| サイドプランク | Side Plank | bodyweight | - |
| ロシアンツイスト | Russian Twist | bodyweight | - |

#### 有酸素（cardio）
| name | nameEn | type | gymBrands |
|------|--------|------|-----------|
| トレッドミル | Treadmill | cardio | chocoZAP, ANYTIME |
| エアロバイク | Stationary Bike | cardio | chocoZAP, ANYTIME |
| クロストレーナー | Cross Trainer | cardio | ANYTIME |
| ローイングマシン | Rowing Machine | cardio | ANYTIME |
| ステップマシン | Stair Climber | cardio | ANYTIME |

---

### 3.10 reservations（予約）

<details>
<summary>TypeScript型定義</summary>

```typescript
type PaymentType = 'ticket' | 'cash' | 'pending';

interface Reservation {
  // ドキュメントID: 自動生成
  trainerId: string;
  traineeId: string;

  // 予約情報
  scheduledAt: Timestamp;               // 予約日時
  duration: number;                     // 時間（分）
  location: string;                     // 場所

  // ステータス
  status: 'pending' | 'confirmed' | 'cancelled' | 'completed';
  cancelledBy: 'trainer' | 'trainee' | null;
  cancellationReason: string | null;

  // 支払い情報（completedの場合のみ有効）
  paymentType: PaymentType;             // ticket: 回数券消化 / cash: 現金等 / pending: 未払い
  ticketId: string | null;              // 使用した回数券ID（二重消化防止）
  paymentMethod: string | null;         // 現金/振込/PayPay等のメモ（任意）
  paidAt: Timestamp | null;             // 支払い確認日時

  // メモ
  trainerNote: string;
  traineeNote: string;

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
  confirmedAt: Timestamp | null;
  cancelledAt: Timestamp | null;
  completedAt: Timestamp | null;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/reservation.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'reservation.freezed.dart';
part 'reservation.g.dart';

enum ReservationStatus {
  @JsonValue('pending')
  pending,
  @JsonValue('confirmed')
  confirmed,
  @JsonValue('cancelled')
  cancelled,
  @JsonValue('completed')
  completed,
}

enum CancelledBy {
  @JsonValue('trainer')
  trainer,
  @JsonValue('trainee')
  trainee,
}

enum PaymentType {
  @JsonValue('ticket')
  ticket,      // 回数券消化
  @JsonValue('cash')
  cash,        // 現金等で支払い済み
  @JsonValue('pending')
  pending,     // 未払い
}

@freezed
class Reservation with _$Reservation {
  const Reservation._();

  const factory Reservation({
    required String id,
    required String trainerId,
    required String traineeId,
    @TimestampConverter() required DateTime scheduledAt,
    required int duration,
    required String location,
    required ReservationStatus status,
    CancelledBy? cancelledBy,
    String? cancellationReason,
    // 支払い情報
    @Default(PaymentType.pending) PaymentType paymentType,
    String? ticketId,                    // 使用した回数券ID
    String? paymentMethod,               // 現金/振込/PayPay等
    @TimestampNullableConverter() DateTime? paidAt,
    @Default('') String trainerNote,
    @Default('') String traineeNote,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
    @TimestampNullableConverter() DateTime? confirmedAt,
    @TimestampNullableConverter() DateTime? cancelledAt,
    @TimestampNullableConverter() DateTime? completedAt,
  }) = _Reservation;

  // 支払い済みか
  bool get isPaid => paymentType != PaymentType.pending;

  // 未払いか
  bool get isUnpaid => status == ReservationStatus.completed && paymentType == PaymentType.pending;

  factory Reservation.fromJson(Map<String, dynamic> json) =>
      _$ReservationFromJson(json);

  factory Reservation.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Reservation.fromJson({...data, 'id': doc.id});
  }
}

// Nullable Timestamp変換用コンバーター
class TimestampNullableConverter
    implements JsonConverter<DateTime?, Timestamp?> {
  const TimestampNullableConverter();

  @override
  DateTime? fromJson(Timestamp? timestamp) => timestamp?.toDate();

  @override
  Timestamp? toJson(DateTime? date) =>
      date != null ? Timestamp.fromDate(date) : null;
}
```
</details>

**インデックス:**
- `trainerId` + `scheduledAt`
- `traineeId` + `scheduledAt`
- `trainerId` + `status` + `scheduledAt`

---

### 3.11 trainingMenus（トレーニングメニュー）

```typescript
interface TrainingMenu {
  // ドキュメントID: 自動生成
  trainerId: string;

  // メニュー情報
  name: string;
  description: string;
  targetMuscles: string[];              // 対象部位
  difficulty: 'beginner' | 'intermediate' | 'advanced';

  // エクササイズ一覧
  exercises: {
    name: string;
    description: string;
    defaultSets: number;
    defaultReps: number;
    defaultWeight: number | null;       // kg
    restTime: number;                   // 秒
    notes: string;
  }[];

  // 公開設定
  isTemplate: boolean;                  // テンプレートとして保存

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

---

### 3.12 trainingSessions（トレーニングセッション）

```typescript
interface TrainingSession {
  // ドキュメントID: 自動生成
  trainerId: string;
  traineeId: string;
  reservationId: string | null;         // 紐付く予約（ない場合もある）

  // セッション情報
  sessionDate: Timestamp;
  duration: number;                     // 実施時間（分）

  // 実施内容
  exercises: {
    name: string;
    sets: {
      setNumber: number;
      reps: number;
      weight: number | null;            // kg
      completed: boolean;
    }[];
    notes: string;
  }[];

  // 全体メモ
  trainerNotes: string;                 // トレーナーのメモ
  traineeNotes: string;                 // トレーニーのメモ

  // 身体測定（任意）
  measurements: {
    weight: number | null;
    bodyFatPercentage: number | null;
  } | null;

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**インデックス:**
- `traineeId` + `sessionDate`
- `trainerId` + `sessionDate`

---

### 3.13 meals（食事記録）

<details>
<summary>TypeScript型定義</summary>

```typescript
interface Meal {
  // ドキュメントID: 自動生成
  traineeId: string;
  trainerId: string | null;             // 紐付くトレーナー

  // 食事情報
  mealType: 'breakfast' | 'lunch' | 'dinner' | 'snack';
  mealDate: Timestamp;
  description: string;                  // 食事内容テキスト

  // 画像
  images: {
    url: string;
    path: string;                       // Storage パス
  }[];

  // 栄養情報（任意）
  nutrition: {
    calories: number | null;
    protein: number | null;             // g
    carbs: number | null;               // g
    fat: number | null;                 // g
  } | null;

  // フィードバック状態
  hasFeedback: boolean;

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```
</details>

<details>
<summary>Dart型定義（freezed）</summary>

```dart
// lib/domain/entities/meal.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

part 'meal.freezed.dart';
part 'meal.g.dart';

enum MealType {
  @JsonValue('breakfast')
  breakfast,
  @JsonValue('lunch')
  lunch,
  @JsonValue('dinner')
  dinner,
  @JsonValue('snack')
  snack,
}

@freezed
class MealImage with _$MealImage {
  const factory MealImage({
    required String url,
    required String path,
  }) = _MealImage;

  factory MealImage.fromJson(Map<String, dynamic> json) =>
      _$MealImageFromJson(json);
}

@freezed
class Nutrition with _$Nutrition {
  const factory Nutrition({
    int? calories,
    double? protein,
    double? carbs,
    double? fat,
  }) = _Nutrition;

  factory Nutrition.fromJson(Map<String, dynamic> json) =>
      _$NutritionFromJson(json);
}

@freezed
class Meal with _$Meal {
  const factory Meal({
    required String id,
    required String traineeId,
    String? trainerId,
    required MealType mealType,
    @TimestampConverter() required DateTime mealDate,
    required String description,
    @Default([]) List<MealImage> images,
    Nutrition? nutrition,
    @Default(false) bool hasFeedback,
    @TimestampConverter() required DateTime createdAt,
    @TimestampConverter() required DateTime updatedAt,
  }) = _Meal;

  factory Meal.fromJson(Map<String, dynamic> json) => _$MealFromJson(json);

  factory Meal.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Meal.fromJson({...data, 'id': doc.id});
  }
}
```
</details>

**インデックス:**
- `traineeId` + `mealDate`
- `trainerId` + `mealDate`
- `traineeId` + `hasFeedback`

---

### 3.14 mealFeedbacks（食事フィードバック）

```typescript
interface MealFeedback {
  // ドキュメントID: 自動生成
  mealId: string;
  trainerId: string;
  traineeId: string;

  // フィードバック内容
  rating: 1 | 2 | 3 | 4 | 5;           // 評価
  comment: string;                      // コメント
  suggestions: string;                  // 改善提案

  // 既読状態
  isRead: boolean;

  // タイムスタンプ
  createdAt: Timestamp;
  readAt: Timestamp | null;
}
```

---

### 3.15 chatRooms（チャットルーム）

```typescript
interface ChatRoom {
  // ドキュメントID: `${trainerId}_${traineeId}` 形式
  trainerId: string;
  traineeId: string;
  participants: string[];               // [trainerId, traineeId]

  // 最新メッセージ情報
  lastMessage: {
    text: string;
    senderId: string;
    sentAt: Timestamp;
  } | null;

  // 未読カウント
  unreadCount: {
    [userId: string]: number;
  };

  // タイムスタンプ
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**サブコレクション: messages**
```typescript
interface Message {
  // ドキュメントID: 自動生成
  senderId: string;
  text: string;

  // 画像（任意）
  image: {
    url: string;
    path: string;
  } | null;

  // 既読情報
  readBy: string[];                     // 既読したユーザーID配列

  // タイムスタンプ
  createdAt: Timestamp;
}
```

**インデックス (chatRooms):**
- `participants` (配列含む)
- `updatedAt`

**インデックス (messages):**
- `createdAt`

---

### 3.16 notifications（通知）

```typescript
interface Notification {
  // ドキュメントID: 自動生成
  userId: string;                       // 受信者

  // 通知内容
  type: 'reservation_created' | 'reservation_confirmed' | 'reservation_cancelled' |
        'chat_message' | 'meal_feedback' | 'reminder';
  title: string;
  body: string;

  // リンク先
  targetType: 'reservation' | 'chat' | 'meal' | 'training';
  targetId: string;

  // 状態
  isRead: boolean;

  // タイムスタンプ
  createdAt: Timestamp;
  readAt: Timestamp | null;
}
```

**インデックス:**
- `userId` + `createdAt`
- `userId` + `isRead` + `createdAt`

---

## 4. Cloud Storage 構造

```
storage/
├── users/
│   └── {userId}/
│       └── profile/
│           └── avatar.jpg              # プロフィール画像
├── meals/
│   └── {mealId}/
│       └── {imageId}.jpg               # 食事画像
└── chat/
    └── {roomId}/
        └── {messageId}/
            └── {imageId}.jpg           # チャット画像
```

### 4.1 ファイル命名規則

| 種別 | パス | 命名規則 |
|------|------|----------|
| プロフィール画像 | `/users/{userId}/profile/` | `avatar_{timestamp}.jpg` |
| 食事画像 | `/meals/{mealId}/` | `{uuid}.jpg` |
| チャット画像 | `/chat/{roomId}/{messageId}/` | `{uuid}.jpg` |

### 4.2 画像サイズ制限

| 種別 | 最大サイズ | リサイズ |
|------|-----------|----------|
| プロフィール画像 | 5MB | 400x400px |
| 食事画像 | 10MB | 1200x1200px |
| チャット画像 | 10MB | 1200x1200px |

---

## 5. セキュリティルール

### 5.1 Firestore Rules

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ============ ヘルパー関数 ============

    // ログイン済みか
    function isAuthenticated() {
      return request.auth != null;
    }

    // 本人か
    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    // トレーナー権限を持っているか
    function isTrainer() {
      return isAuthenticated() &&
             'trainer' in get(/databases/$(database)/documents/users/$(request.auth.uid)).data.roles;
    }

    // トレーニー権限を持っているか
    function isTrainee() {
      return isAuthenticated() &&
             'trainee' in get(/databases/$(database)/documents/users/$(request.auth.uid)).data.roles;
    }

    // 現在アクティブな役割がトレーナーか
    function isActiveTrainer() {
      return isAuthenticated() &&
             get(/databases/$(database)/documents/users/$(request.auth.uid)).data.activeRole == 'trainer';
    }

    // 現在アクティブな役割がトレーニーか
    function isActiveTrainee() {
      return isAuthenticated() &&
             get(/databases/$(database)/documents/users/$(request.auth.uid)).data.activeRole == 'trainee';
    }

    // 紐付いているか確認
    function isLinked(trainerId, traineeId) {
      return exists(/databases/$(database)/documents/trainerTraineeLinks/$(trainerId + '_' + traineeId)) &&
             get(/databases/$(database)/documents/trainerTraineeLinks/$(trainerId + '_' + traineeId)).data.status == 'active';
    }

    // ============ コレクションルール ============

    // users
    match /users/{userId} {
      allow read: if isAuthenticated();
      allow create: if isOwner(userId);
      allow update: if isOwner(userId);
      allow delete: if false;

      match /fcmTokens/{tokenId} {
        allow read, write: if isOwner(userId);
      }
    }

    // trainerProfiles
    match /trainerProfiles/{trainerId} {
      allow read: if isAuthenticated();
      allow write: if isOwner(trainerId) && isTrainer();
    }

    // traineeProfiles
    match /traineeProfiles/{traineeId} {
      allow read: if isOwner(traineeId) ||
                    (isTrainer() && isLinked(request.auth.uid, traineeId));
      allow write: if isOwner(traineeId) && isTrainee();
    }

    // bodyMeasurements（体重・身長記録）
    match /bodyMeasurements/{measurementId} {
      allow read: if isOwner(resource.data.userId) ||
                    (isTrainer() && isLinked(request.auth.uid, resource.data.userId));
      allow create: if isAuthenticated() &&
                      request.resource.data.userId == request.auth.uid;
      allow update: if isOwner(resource.data.userId);
      allow delete: if isOwner(resource.data.userId);
    }

    // trainerTraineeLinks
    match /trainerTraineeLinks/{linkId} {
      allow read: if isAuthenticated() &&
                    (resource.data.trainerId == request.auth.uid ||
                     resource.data.traineeId == request.auth.uid);
      allow create: if isAuthenticated();
      allow update: if resource.data.trainerId == request.auth.uid ||
                      resource.data.traineeId == request.auth.uid;
    }

    // tickets（回数券）
    match /tickets/{ticketId} {
      allow read: if isAuthenticated() &&
                    (resource.data.trainerId == request.auth.uid ||
                     resource.data.traineeId == request.auth.uid);
      allow create: if isTrainer() &&
                      request.resource.data.trainerId == request.auth.uid;
      allow update: if isTrainer() &&
                      resource.data.trainerId == request.auth.uid;
      allow delete: if false; // 削除不可（履歴保持）
    }

    // invitations
    match /invitations/{invitationId} {
      allow read: if isAuthenticated();
      allow create: if isTrainer();
      allow update: if isTrainer() && resource.data.trainerId == request.auth.uid ||
                      isTrainee();
    }

    // trainerSchedules
    match /trainerSchedules/{trainerId} {
      allow read: if isAuthenticated();
      allow write: if isOwner(trainerId) && isTrainer();
    }

    // exerciseMaster（読み取り専用マスターデータ）
    match /exerciseMaster/{exerciseId} {
      allow read: if isAuthenticated();
      allow write: if false; // 管理者のみFirebaseコンソールから更新
    }

    // reservations
    match /reservations/{reservationId} {
      allow read: if isAuthenticated() &&
                    (resource.data.trainerId == request.auth.uid ||
                     resource.data.traineeId == request.auth.uid);
      allow create: if isAuthenticated() &&
                      (request.resource.data.trainerId == request.auth.uid ||
                       request.resource.data.traineeId == request.auth.uid);
      allow update: if resource.data.trainerId == request.auth.uid ||
                      resource.data.traineeId == request.auth.uid;
    }

    // trainingMenus
    match /trainingMenus/{menuId} {
      allow read: if isAuthenticated() &&
                    resource.data.trainerId == request.auth.uid;
      allow write: if isTrainer() &&
                     request.resource.data.trainerId == request.auth.uid;
    }

    // trainingSessions
    match /trainingSessions/{sessionId} {
      allow read: if isAuthenticated() &&
                    (resource.data.trainerId == request.auth.uid ||
                     resource.data.traineeId == request.auth.uid);
      allow create: if isTrainer();
      allow update: if resource.data.trainerId == request.auth.uid ||
                      resource.data.traineeId == request.auth.uid;
    }

    // meals
    match /meals/{mealId} {
      allow read: if isAuthenticated() &&
                    (resource.data.traineeId == request.auth.uid ||
                     (resource.data.trainerId == request.auth.uid && isTrainer()));
      allow create: if isTrainee() &&
                      request.resource.data.traineeId == request.auth.uid;
      allow update: if resource.data.traineeId == request.auth.uid;
    }

    // mealFeedbacks
    match /mealFeedbacks/{feedbackId} {
      allow read: if isAuthenticated() &&
                    (resource.data.trainerId == request.auth.uid ||
                     resource.data.traineeId == request.auth.uid);
      allow create: if isTrainer();
      allow update: if resource.data.traineeId == request.auth.uid;
    }

    // chatRooms
    match /chatRooms/{roomId} {
      allow read, write: if isAuthenticated() &&
                           request.auth.uid in resource.data.participants;

      match /messages/{messageId} {
        allow read: if isAuthenticated() &&
                      request.auth.uid in get(/databases/$(database)/documents/chatRooms/$(roomId)).data.participants;
        allow create: if isAuthenticated() &&
                        request.auth.uid in get(/databases/$(database)/documents/chatRooms/$(roomId)).data.participants &&
                        request.resource.data.senderId == request.auth.uid;
      }
    }

    // notifications
    match /notifications/{notificationId} {
      allow read: if isOwner(resource.data.userId);
      allow update: if isOwner(resource.data.userId);
      allow create: if false; // Cloud Functionsからのみ作成
    }
  }
}
```

### 5.2 Storage Rules

```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return request.auth.uid == userId;
    }

    function isValidImage() {
      return request.resource.contentType.matches('image/.*') &&
             request.resource.size < 10 * 1024 * 1024; // 10MB
    }

    // プロフィール画像
    match /users/{userId}/profile/{fileName} {
      allow read: if isAuthenticated();
      allow write: if isOwner(userId) && isValidImage();
    }

    // 食事画像
    match /meals/{mealId}/{fileName} {
      allow read: if isAuthenticated();
      allow write: if isAuthenticated() && isValidImage();
    }

    // チャット画像
    match /chat/{roomId}/{allPaths=**} {
      allow read: if isAuthenticated();
      allow write: if isAuthenticated() && isValidImage();
    }
  }
}
```

---

## 6. インデックス設定

Firestore で必要な複合インデックス:

```json
{
  "indexes": [
    {
      "collectionGroup": "bodyMeasurements",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "measuredAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "tickets",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "trainerId", "order": "ASCENDING" },
        { "fieldPath": "traineeId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "tickets",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "traineeId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "reservations",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "trainerId", "order": "ASCENDING" },
        { "fieldPath": "scheduledAt", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "reservations",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "traineeId", "order": "ASCENDING" },
        { "fieldPath": "scheduledAt", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "reservations",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "trainerId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "scheduledAt", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "reservations",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "trainerId", "order": "ASCENDING" },
        { "fieldPath": "paymentType", "order": "ASCENDING" },
        { "fieldPath": "scheduledAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "trainingSessions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "traineeId", "order": "ASCENDING" },
        { "fieldPath": "sessionDate", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "meals",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "traineeId", "order": "ASCENDING" },
        { "fieldPath": "mealDate", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "notifications",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "isRead", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "messages",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

---

## 7. データマイグレーション戦略

### 7.1 バージョニング

各ドキュメントに `schemaVersion` フィールドを追加し、スキーマ変更時に対応。

```typescript
interface VersionedDocument {
  schemaVersion: number;
  // ... other fields
}
```

### 7.2 マイグレーション手順

1. 新しいスキーマ定義を作成
2. Cloud Functions でマイグレーションスクリプト作成
3. バッチ処理で既存ドキュメントを更新
4. アプリ側で両バージョン対応のロジックを実装
5. 完全移行後、古いスキーマ対応コードを削除
