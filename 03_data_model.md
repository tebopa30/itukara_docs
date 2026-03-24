# itukara - Step3: データモデル設計

クリーンアーキテクチャにおける Domain 層の Entity（データモデル）設計です。
不変性（Immutability）と null 安全性を確保するため、Flutter では "Freezed" パッケージ（および "json_serializable"）の導入を前提とします。これにより、"copyWith" 等の便利なメソッドが自動生成され、Riverpodを使った状態管理において安全かつ予測可能なオブジェクトの更新が可能になります。

## 1. 各Entityの定義と責務

### UserEntity（ユーザー情報・将来拡張用）
- "責務": アプリを利用するユーザー自身を表します。初期のローカル運用ではダミーデータで構いませんが、将来的にFirebase Authenticationへ移行する際の中核となります。
- "SQLite / Firebase 考慮事項": SQLiteでは単一レコード（ID固定）、Firebaseでは `uid` が入ります。

> **Note**: 実装コードは `frontend/app/lib/features/auth/domain/user_entity.dart` に抽出済みです。

### PremiumStatusEntity（課金情報）
- "責務": IAP（アプリ内課金）によるユーザーのプレミアム状態を表現します。
- "拡張ポイント": 単純な boolean ではなく、有効期限や購入したProductIDを持たせることで、サブスクリプションだけでなく買い切り型や複数プランにも柔軟に対応できます。

> **Note**: 実装コードは `frontend/app/lib/features/premium/domain/premium_status_entity.dart` に抽出済みです。

### TaskEntity（育児タスクカテゴリ/親定義）
- "責務": 「ミルク」「おむつ」「睡眠」などのタスクの種類そのものを定義します。
- "最新の実行日時キャッシュ": `lastRecordedAt` を持たせることで、一覧画面での「最後にやったのはいつか？」の表示や、通知判定を行う際に毎回 `TaskRecord` を検索するコストを省き、パフォーマンスを大幅に向上させます（TaskRecord保存時に合わせて更新する想定）。
- "SQLite / Firebase 考慮事項": 
  - SQLiteではAuto Incrementな `int` IDが一般的ですが、"UUID（String）" を採用します。これにより、ローカルで生成したIDを後からFirebaseにそのまま同期（アップサート）でき、ID衝突の問題が解決します。

> **Note**: 実装コードは `frontend/app/lib/features/task/domain/task_entity.dart` に抽出済みです。

### TaskRecordEntity（タスクの実行履歴・カウント用）
- "責務": 各タスクが「いつ」「どのように」実行されたかの1回分の履歴を保持します。AI異常検知のための基礎データ（時系列データ）となります。
- "拡張ポイント": AIが「普段と違う傾向（例: ミルクの間隔が通常より空いています。）」を検知するためには、タイムスタンプと「量・状態」の記録が必須です。分析に使えるよう定量的な値（ml、分など）も保持できるようにします。

> **Note**: 実装コードは `frontend/app/lib/features/task/domain/task_record_entity.dart` に抽出済みです。

## 2. SQLite / Firebase 両対応に向けた設計の注意点

1. "IDには `String (UUID)` を用いる"
   - SQLiteの `INTEGER PRIMARY KEY AUTOINCREMENT` は、Firebaseへデータを移行・同期する際に「ローカルでのID 1」と「サーバー上の別のデータID 1」が衝突する原因になります。
   - 解決策として、フロントエンド（Domain層のUseCase付近）で `uuid` パッケージ等を利用して作成時に一意な文字列を採番し、それを SQLite の主キーと Firebase の Document ID 両方として使用します。
2. "DateTimeの扱い"
   - SQLite には日付型がないため、`json_serializable` によって生成される `toIso8601String()` を活用し文字列か、または UNIX Timestamp (int) に変換して保存します。
   - Firebase (Firestore) には `Timestamp` 型があります。Data Layer (Repositoryの実装クラス内) でデータの保存/取得を行う際に、型の変換を吸収させ、Domain Entityとしては常にネイティブの `DateTime` 型を利用するようにします。
3. "論理削除の考慮 (`isActive`)"
   - ユーザーが特定の `TaskEntity`（例：ミルク）を削除した場合に、DBから物理削除してしまうと、過去残した `TaskRecordEntity` との紐付けが切れ、AI分析用の過去トラッキングができなくなります。そのため、`isActive = false` による論理削除設計としています。
