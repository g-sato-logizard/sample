# 倉庫業務工数管理システム 仕様書

## 1. システム概要

### 1.1 システム目的
倉庫業務における人的リソースの効率的な管理と可視化を実現し、生産性向上と業務改善の基盤を提供する。

### 1.2 システム範囲
- 倉庫内作業員の業務時間記録
- 作業実績の可視化・分析
- ロジザードZERO連携による実績データ取得
- 管理者向けダッシュボード機能

### 1.3 システム構成図

```
[倉庫内PC] ←QR/バーコード→ [作業員認証]
    ↓
[Webアプリケーション] ←API→ [ロジザードZERO]
    ↓
[データベース] → [ダッシュボード]
```

## 2. 機能要件

### 2.1 認証・認可機能

#### 2.1.1 作業員認証
- **入力方式**: QRコード、バーコード読み取り
- **認証情報**: 作業員ID、氏名、所属部署、権限レベル
- **セキュリティ**: トークンベース認証（JWT）
- **セッション管理**: 8時間自動タイムアウト

#### 2.1.2 管理者認証
- **入力方式**: ID/パスワード、2要素認証（TOTP）
- **権限レベル**: 閲覧者、管理者、システム管理者
- **アクセス制御**: RBAC（Role-Based Access Control）

### 2.2 作業時間記録機能

#### 2.2.1 作業開始・終了記録
```
作業記録 = {
  作業員ID: string,
  作業種別: enum[ピッキング, 梱包, 出荷, 検品, その他],
  開始時刻: timestamp,
  終了時刻: timestamp,
  作業時間: duration,
  作業場所: string,
  備考: string?
}
```

#### 2.2.2 作業種別定義
- **ピッキング**: 商品の取り出し作業
- **梱包**: 商品の包装・梱包作業
- **出荷**: 出荷準備・発送作業
- **検品**: 品質確認・検査作業
- **その他**: 清掃、会議、休憩等

#### 2.2.3 UI要件
- **操作性**: 大型ボタン（最小60px）、明確な色分け
- **応答性**: ボタン押下後1秒以内のフィードバック
- **エラーハンドリング**: 重複操作防止、ネットワーク断対応

### 2.3 外部API連携機能

#### 2.3.1 ロジザードZERO API連携
```
実績データ = {
  出荷日: date,
  出荷先: string,
  商品コード: string,
  商品名: string,
  出荷数量: number,
  梱包数: number,
  作業者: string?
}
```

#### 2.3.2 API通信仕様
- **プロトコル**: HTTPS/REST API
- **認証**: API Key + OAuth 2.0
- **データ形式**: JSON
- **取得頻度**: 1時間毎の自動同期
- **エラー処理**: 指数バックオフによるリトライ機能

### 2.4 ダッシュボード機能

#### 2.4.1 KPI指標
```
生産性指標 = {
  時間当たり処理数: 処理数 / 作業時間,
  作業効率: 実績時間 / 標準時間,
  稼働率: 作業時間 / 勤務時間,
  品質指標: 正確処理数 / 総処理数
}
```

#### 2.4.2 表示機能
- **リアルタイム監視**: 現在の作業状況
- **日次実績**: 作業者別・作業種別別集計
- **週次/月次トレンド**: 期間比較グラフ
- **アラート機能**: 異常値検知・通知

#### 2.4.3 レポート機能
- **CSV出力**: 詳細データエクスポート
- **グラフ表示**: Chart.js使用
- **フィルタリング**: 期間、作業者、作業種別

## 3. 非機能要件

### 3.1 性能要件
- **応答時間**: 画面表示2秒以内
- **同時利用者数**: 最大50名
- **データ保持期間**: 3年間
- **バックアップ**: 日次自動バックアップ

### 3.2 セキュリティ要件
- **データ暗号化**: AES-256（保存時）、TLS 1.3（通信時）
- **アクセスログ**: 全操作記録・監査証跡
- **脆弱性対策**: OWASP Top 10対応
- **個人情報保護**: GDPR準拠設計

### 3.3 可用性要件
- **稼働率**: 99.5%以上
- **障害復旧時間**: 4時間以内
- **メンテナンス**: 月1回、深夜時間帯

## 4. システム構成

### 4.1 技術スタック

#### 4.1.1 フロントエンド
- **フレームワーク**: React 18 + TypeScript
- **UI库**: Material-UI v5
- **状態管理**: Redux Toolkit
- **バーコード読取**: QuaggaJS

#### 4.1.2 バックエンド
- **実行環境**: Node.js 18 + Express
- **言語**: TypeScript
- **認証**: JWT + Passport.js
- **API仕様**: OpenAPI 3.0

#### 4.1.3 データベース
- **主DB**: PostgreSQL 14
- **キャッシュ**: Redis 7
- **データ移行**: Prisma ORM

#### 4.1.4 インフラ
- **コンテナ**: Docker + Docker Compose
- **Webサーバー**: Nginx
- **監視**: Prometheus + Grafana

### 4.2 アーキテクチャ図

```
┌─────────────────┐    ┌─────────────────┐
│   Frontend      │    │   Backend       │
│   (React)       │◄──►│   (Express)     │
└─────────────────┘    └─────────────────┘
                                │
                       ┌─────────────────┐
                       │   Database      │
                       │   (PostgreSQL)  │
                       └─────────────────┘
                                │
                       ┌─────────────────┐
                       │   External API  │
                       │   (ロジザードZERO)│
                       └─────────────────┘
```

## 5. データベース設計

### 5.1 テーブル構成

#### 5.1.1 作業員マスタ（users）
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  employee_id VARCHAR(20) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  department VARCHAR(50),
  role VARCHAR(20) DEFAULT 'worker',
  qr_code TEXT UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 5.1.2 作業記録（work_records）
```sql
CREATE TABLE work_records (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  work_type VARCHAR(20) NOT NULL,
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  duration INTEGER, -- 秒単位
  location VARCHAR(50),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 5.1.3 実績データ（shipment_records）
```sql
CREATE TABLE shipment_records (
  id SERIAL PRIMARY KEY,
  shipment_date DATE NOT NULL,
  destination VARCHAR(200),
  product_code VARCHAR(50),
  product_name VARCHAR(200),
  quantity INTEGER,
  package_count INTEGER,
  worker_id INTEGER REFERENCES users(id),
  logizard_id VARCHAR(50) UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 5.2 インデックス設計
```sql
-- パフォーマンス最適化用インデックス
CREATE INDEX idx_work_records_user_date ON work_records(user_id, DATE(start_time));
CREATE INDEX idx_work_records_type_date ON work_records(work_type, DATE(start_time));
CREATE INDEX idx_shipment_records_date ON shipment_records(shipment_date);
```

## 6. セキュリティ仕様

### 6.1 認証・認可
- **JWT有効期限**: 8時間
- **リフレッシュトークン**: 7日間
- **パスワード要件**: 8文字以上、大小英数字+記号
- **失敗制限**: 5回連続失敗でアカウントロック

### 6.2 データ保護
- **暗号化**: 個人情報はAES-256で暗号化
- **ログ管理**: 操作ログ・エラーログの長期保存
- **アクセス制御**: IPアドレス制限、操作権限管理

## 7. 運用・保守仕様

### 7.1 監視・アラート
- **システム監視**: CPU、メモリ、ディスク使用率
- **アプリケーション監視**: レスポンス時間、エラー率
- **ビジネス監視**: 作業時間異常、データ欠損

### 7.2 バックアップ・復旧
- **データベース**: 日次フルバックアップ + 時間毎差分
- **ファイル**: 日次バックアップ
- **復旧手順**: RTO 4時間、RPO 1時間

## 8. 開発・運用環境

### 8.1 開発環境
- **OS**: Windows 11
- **コンテナ**: Docker Desktop
- **IDE**: Visual Studio Code
- **バージョン管理**: Git

### 8.2 本番環境
- **クラウド**: AWS ECS Fargate
- **データベース**: Amazon RDS (PostgreSQL)
- **ファイル**: Amazon S3
- **CDN**: Amazon CloudFront

## 9. 段階的実装計画

### Phase 1: 基本機能（4週間）
1. 基本認証機能
2. 作業時間記録機能
3. 基本的なデータ表示

### Phase 2: 外部連携（3週間）
1. ロジザードZERO API連携
2. データ同期機能
3. エラーハンドリング強化

### Phase 3: 分析機能（3週間）
1. ダッシュボード構築
2. レポート機能
3. アラート機能

### Phase 4: 運用最適化（2週間）
1. パフォーマンス最適化
2. セキュリティ強化
3. 運用監視機能

## 10. 品質保証

### 10.1 テスト戦略
- **単体テスト**: Jest + React Testing Library
- **統合テスト**: Supertest + Testcontainers
- **E2Eテスト**: Playwright
- **パフォーマンステスト**: Artillery

### 10.2 品質指標
- **コードカバレッジ**: 80%以上
- **脆弱性スキャン**: Snyk使用
- **コード品質**: SonarQube使用

---

**文書バージョン**: 1.0
**作成日**: 2025-06-13
**承認者**: システム責任者
**次回レビュー**: 2025-07-13
