# Trainer Desk - ドキュメント

パーソナルトレーナー向け顧客管理・トレーニング管理アプリケーションの仕様書です。

## ドキュメント一覧

| ドキュメント | 説明 |
|------------|------|
| [機能仕様書](./functional-spec.md) | アプリケーションの機能要件 |
| [技術仕様書](./technical-spec.md) | 技術スタック・アーキテクチャ |
| [データベース設計書](./database-schema.md) | Firestore データモデル |
| [API仕様書](./api-spec.md) | Cloud Functions API エンドポイント |

## プロジェクト概要

### 目的
パーソナルトレーナーとトレーニーをつなぎ、効率的なトレーニング管理・予約管理・食事管理を実現するモバイルアプリケーション。

### 対象プラットフォーム
- iOS
- Android

### ユーザータイプ
1. **トレーナー**: パーソナルトレーナー
2. **トレーニー**: トレーニングを受ける顧客

### 主要機能
- Google認証によるログイン
- 予約管理
- トレーニング内容管理
- 食事記録・管理
- チャット機能

### 技術スタック
- **フロントエンド**: Flutter
- **バックエンド**: Firebase
  - Authentication
  - Cloud Firestore
  - Cloud Functions
  - Cloud Storage
  - Cloud Messaging
