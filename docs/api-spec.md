# API仕様書

## 1. 概要

本ドキュメントは「Trainer Desk」アプリケーションのCloud Functions APIエンドポイントを定義します。

## 2. 共通仕様

### 2.1 ベースURL

```
https://{region}-{project-id}.cloudfunctions.net/
```

### 2.2 認証

すべてのAPIはFirebase Authentication のIDトークンによる認証が必要です。

```
Authorization: Bearer {idToken}
```

### 2.3 レスポンス形式

#### 成功時
```json
{
  "success": true,
  "data": { ... }
}
```

#### エラー時
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ"
  }
}
```

### 2.4 共通エラーコード

| コード | HTTPステータス | 説明 |
|-------|---------------|------|
| UNAUTHENTICATED | 401 | 認証エラー |
| PERMISSION_DENIED | 403 | 権限エラー |
| NOT_FOUND | 404 | リソースが見つからない |
| INVALID_ARGUMENT | 400 | パラメータエラー |
| INTERNAL | 500 | サーバーエラー |

---

## 3. Callable Functions

### 3.1 契約関連

#### 3.1.1 createContractRequest

契約リクエストを作成します。

**関数名**: `createContractRequest`

**リクエスト**:
```json
{
  "trainerId": "string",
  "message": "string (optional)"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "contractId": "string"
  }
}
```

**エラー**:
| コード | 説明 |
|-------|------|
| ALREADY_EXISTS | すでに契約リクエストが存在する |
| INVALID_ROLE | トレーニー以外からのリクエスト |

---

#### 3.1.2 respondToContractRequest

契約リクエストに応答します（承認/拒否）。

**関数名**: `respondToContractRequest`

**リクエスト**:
```json
{
  "contractId": "string",
  "approved": "boolean"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "contractId": "string",
    "status": "approved | rejected",
    "chatRoomId": "string (if approved)"
  }
}
```

**処理内容**:
- 契約ステータス更新
- 承認時: チャットルーム作成
- 通知送信

---

#### 3.1.3 cancelContract

契約を解除します。

**関数名**: `cancelContract`

**リクエスト**:
```json
{
  "contractId": "string",
  "reason": "string (optional)"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "contractId": "string",
    "cancelledReservations": ["string"]
  }
}
```

**処理内容**:
- 契約ステータスを cancelled に更新
- 未完了の予約をキャンセル
- 通知送信

---

### 3.2 予約関連

#### 3.2.1 createReservation

予約を作成します。

**関数名**: `createReservation`

**リクエスト**:
```json
{
  "trainerId": "string",
  "startTime": "ISO8601 string",
  "endTime": "ISO8601 string",
  "note": "string (optional)"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "reservationId": "string"
  }
}
```

**バリデーション**:
- 有効な契約が存在すること
- トレーナーの営業時間内であること
- 他の予約と重複しないこと
- 過去の日時でないこと

**エラー**:
| コード | 説明 |
|-------|------|
| NO_CONTRACT | 有効な契約がない |
| OUTSIDE_BUSINESS_HOURS | 営業時間外 |
| TIME_SLOT_UNAVAILABLE | 時間帯が利用不可 |
| PAST_DATE | 過去の日時 |

---

#### 3.2.2 respondToReservation

予約リクエストに応答します（承認/拒否）。

**関数名**: `respondToReservation`

**リクエスト**:
```json
{
  "reservationId": "string",
  "confirmed": "boolean",
  "rejectReason": "string (optional, if not confirmed)"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "reservationId": "string",
    "status": "confirmed | rejected"
  }
}
```

**処理内容**:
- 予約ステータス更新
- 通知送信

---

#### 3.2.3 cancelReservation

予約をキャンセルします。

**関数名**: `cancelReservation`

**リクエスト**:
```json
{
  "reservationId": "string",
  "reason": "string (optional)"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "reservationId": "string"
  }
}
```

**バリデーション**:
- 予約が pending または confirmed であること
- キャンセル期限（24時間前）を過ぎていないこと（設定による）

---

#### 3.2.4 getAvailableSlots

トレーナーの空き時間を取得します。

**関数名**: `getAvailableSlots`

**リクエスト**:
```json
{
  "trainerId": "string",
  "date": "YYYY-MM-DD"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "slots": [
      {
        "startTime": "ISO8601 string",
        "endTime": "ISO8601 string",
        "available": true
      }
    ]
  }
}
```

---

### 3.3 食事管理関連

#### 3.3.1 addMealFeedback

食事記録にフィードバックを追加します。

**関数名**: `addMealFeedback`

**リクエスト**:
```json
{
  "mealRecordId": "string",
  "comment": "string"
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "mealRecordId": "string"
  }
}
```

**バリデーション**:
- トレーナーからのリクエストであること
- 該当トレーニーと有効な契約があること

**処理内容**:
- 食事記録にフィードバック追加
- トレーニーへ通知送信

---

### 3.4 通知関連

#### 3.4.1 markNotificationsAsRead

通知を既読にします。

**関数名**: `markNotificationsAsRead`

**リクエスト**:
```json
{
  "notificationIds": ["string"]
}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "updatedCount": "number"
  }
}
```

---

### 3.5 ユーザー関連

#### 3.5.1 deleteAccount

アカウントを削除します。

**関数名**: `deleteAccount`

**リクエスト**:
```json
{}
```

**レスポンス**:
```json
{
  "success": true,
  "data": {
    "deletedAt": "ISO8601 string"
  }
}
```

**処理内容**:
- 有効な契約をすべてキャンセル
- 未完了の予約をすべてキャンセル
- ユーザードキュメント削除
- Firebase Auth ユーザー削除
- Storage 内のファイル削除

---

## 4. Firestore Triggers

### 4.1 認証トリガー

#### 4.1.1 onUserCreated

新規ユーザー作成時の処理。

**トリガー**: `functions.auth.user().onCreate()`

**処理内容**:
- users コレクションに初期ドキュメント作成
- ウェルカム通知の作成

---

### 4.2 予約トリガー

#### 4.2.1 onReservationCreated

予約作成時の処理。

**トリガー**: `functions.firestore.document('reservations/{reservationId}').onCreate()`

**処理内容**:
- トレーナーへプッシュ通知送信
- notifications コレクションへ通知追加

---

#### 4.2.2 onReservationUpdated

予約更新時の処理。

**トリガー**: `functions.firestore.document('reservations/{reservationId}').onUpdate()`

**処理内容**:
- ステータス変更時に相手へ通知送信
- confirmed → プッシュ通知 + アプリ内通知
- rejected → プッシュ通知 + アプリ内通知
- cancelled → プッシュ通知 + アプリ内通知

---

### 4.3 チャットトリガー

#### 4.3.1 onMessageCreated

メッセージ作成時の処理。

**トリガー**: `functions.firestore.document('chatRooms/{chatRoomId}/messages/{messageId}').onCreate()`

**処理内容**:
- 親ドキュメント（chatRooms）の lastMessage 更新
- 相手へプッシュ通知送信

---

### 4.4 食事記録トリガー

#### 4.4.1 onMealRecordUpdated

食事記録更新時の処理（フィードバック追加検知）。

**トリガー**: `functions.firestore.document('mealRecords/{mealRecordId}').onUpdate()`

**処理内容**:
- フィードバック追加時にトレーニーへ通知

---

## 5. Scheduled Functions

### 5.1 sendReservationReminders

予約リマインダーを送信します。

**スケジュール**: 毎時0分実行

**処理内容**:
1. 24時間後に予約がある場合
   - 両者へプッシュ通知送信
2. 1時間後に予約がある場合
   - 両者へプッシュ通知送信

```typescript
// functions/src/scheduled/sendReminders.ts
export const sendReservationReminders = functions.pubsub
  .schedule('0 * * * *')
  .timeZone('Asia/Tokyo')
  .onRun(async (context) => {
    const now = new Date();
    const in24Hours = new Date(now.getTime() + 24 * 60 * 60 * 1000);
    const in1Hour = new Date(now.getTime() + 60 * 60 * 1000);

    // 24時間前リマインダー
    await send24HourReminders(in24Hours);

    // 1時間前リマインダー
    await send1HourReminders(in1Hour);
  });
```

---

### 5.2 cleanupOldNotifications

古い通知を削除します。

**スケジュール**: 毎日午前3時実行

**処理内容**:
- 90日以上前の既読通知を削除

---

## 6. HTTP Functions（管理用）

### 6.1 healthCheck

ヘルスチェック用エンドポイント。

**エンドポイント**: `GET /healthCheck`

**認証**: 不要

**レスポンス**:
```json
{
  "status": "ok",
  "timestamp": "ISO8601 string"
}
```

---

## 7. 実装例

### 7.1 Callable Function 実装例

```typescript
// functions/src/reservations/createReservation.ts
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

interface CreateReservationData {
  trainerId: string;
  startTime: string;
  endTime: string;
  note?: string;
}

export const createReservation = functions.https.onCall(
  async (data: CreateReservationData, context) => {
    // 認証チェック
    if (!context.auth) {
      throw new functions.https.HttpsError(
        'unauthenticated',
        '認証が必要です'
      );
    }

    const traineeId = context.auth.uid;
    const { trainerId, startTime, endTime, note } = data;

    // バリデーション
    if (!trainerId || !startTime || !endTime) {
      throw new functions.https.HttpsError(
        'invalid-argument',
        '必須パラメータが不足しています'
      );
    }

    const db = admin.firestore();

    // 有効な契約確認
    const contractSnapshot = await db
      .collection('contracts')
      .where('trainerId', '==', trainerId)
      .where('traineeId', '==', traineeId)
      .where('status', '==', 'approved')
      .limit(1)
      .get();

    if (contractSnapshot.empty) {
      throw new functions.https.HttpsError(
        'failed-precondition',
        '有効な契約がありません'
      );
    }

    const contractId = contractSnapshot.docs[0].id;

    // 時間帯の重複チェック
    const overlappingReservations = await db
      .collection('reservations')
      .where('trainerId', '==', trainerId)
      .where('status', 'in', ['pending', 'confirmed'])
      .where('startTime', '<', new Date(endTime))
      .where('endTime', '>', new Date(startTime))
      .get();

    if (!overlappingReservations.empty) {
      throw new functions.https.HttpsError(
        'failed-precondition',
        'この時間帯はすでに予約があります'
      );
    }

    // 予約作成
    const reservationRef = db.collection('reservations').doc();
    await reservationRef.set({
      trainerId,
      traineeId,
      contractId,
      startTime: new Date(startTime),
      endTime: new Date(endTime),
      status: 'pending',
      note: note || null,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    return {
      success: true,
      data: {
        reservationId: reservationRef.id,
      },
    };
  }
);
```

### 7.2 Firestore Trigger 実装例

```typescript
// functions/src/chat/onMessageCreated.ts
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

export const onMessageCreated = functions.firestore
  .document('chatRooms/{chatRoomId}/messages/{messageId}')
  .onCreate(async (snapshot, context) => {
    const { chatRoomId } = context.params;
    const message = snapshot.data();
    const senderId = message.senderId;

    const db = admin.firestore();

    // チャットルーム情報取得
    const chatRoomDoc = await db
      .collection('chatRooms')
      .doc(chatRoomId)
      .get();

    if (!chatRoomDoc.exists) return;

    const chatRoom = chatRoomDoc.data()!;
    const recipientId = chatRoom.participants.find(
      (id: string) => id !== senderId
    );

    if (!recipientId) return;

    // チャットルームの lastMessage 更新
    await chatRoomDoc.ref.update({
      lastMessage: message.content,
      lastMessageAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp(),
    });

    // 受信者のFCMトークン取得
    const recipientDoc = await db
      .collection('users')
      .doc(recipientId)
      .get();

    const fcmToken = recipientDoc.data()?.fcmToken;
    if (!fcmToken) return;

    // 送信者情報取得
    const senderDoc = await db
      .collection('users')
      .doc(senderId)
      .get();

    const senderName = senderDoc.data()?.displayName || '不明';

    // プッシュ通知送信
    await admin.messaging().send({
      token: fcmToken,
      notification: {
        title: senderName,
        body: message.type === 'image' ? '画像を送信しました' : message.content,
      },
      data: {
        type: 'chat_message',
        chatRoomId,
      },
      android: {
        priority: 'high',
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
  });
```

---

## 8. クライアント側呼び出し例

### 8.1 Flutter での Callable Function 呼び出し

```dart
// lib/data/services/reservation_service.dart
import 'package:cloud_functions/cloud_functions.dart';

class ReservationService {
  final FirebaseFunctions _functions = FirebaseFunctions.instance;

  Future<String> createReservation({
    required String trainerId,
    required DateTime startTime,
    required DateTime endTime,
    String? note,
  }) async {
    final callable = _functions.httpsCallable('createReservation');

    try {
      final result = await callable.call({
        'trainerId': trainerId,
        'startTime': startTime.toIso8601String(),
        'endTime': endTime.toIso8601String(),
        'note': note,
      });

      final data = result.data as Map<String, dynamic>;
      if (data['success'] == true) {
        return data['data']['reservationId'];
      } else {
        throw Exception(data['error']['message']);
      }
    } on FirebaseFunctionsException catch (e) {
      throw Exception(e.message);
    }
  }
}
```
