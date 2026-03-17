# itukara - Step 4: Flutter Implementation (Domain + Application)

Step 3 / 3.5 の設計に基づく、Flutter側（Domain層とApplication層、最小限のPresentation層）の実装です。
Data層（SQLiteの具象クラス）は未実装のため、Providerを定義するだけとし、一時的にモック等で動かす（またはビルドエラーを放置する）状態とします。

## 1. Provider 設計 (core/providers)

Riverpodを用いてDomain層のRepositoryの依存関係をDI（Dependency Injection）します。

### `app/lib/core/providers/repository_providers.dart`
> **Note**: 実装コードは `frontend/app/lib/core/providers/repository_providers.dart` に抽出済みです。

## 2. Application 層 (Notifier / UseCase)

UIからのアクションを受け取り、Repositoryへ処理を委譲（オーケストレーション）し、結果をUIへ通知する役割を担います。

### `app/lib/features/iap/application/premium_status_notifier.dart`
課金状態を管理・提供するシンプルな Notifier です。

> **Note**: 実装コードは `frontend/app/lib/features/iap/application/premium_status_notifier.dart` に抽出済みです。

### `app/lib/features/task/application/task_notifier.dart`
Task一覧の状態管理と、TaskRecord追加時のトランザクション的な処理（UseCase）をオーケストレーションします。

> **Note**: 実装コードは `frontend/app/lib/features/task/application/task_notifier.dart` に抽出済みです。

## 3. Presentation 層 (UI / View)

UI は `ConsumerWidget` を用い、Application層の Notifier のみを監視・操作します。Data層やDomain層(Repository)へは直接触れません。

### `app/lib/features/task/presentation/task_list_page.dart`

> **Note**: 実装コードは `frontend/app/lib/features/task/presentation/task_list_page.dart` に抽出済みです。

## 4. 状態の流れ・クリーンアーキテクチャの解説

1. "ユーザーが行う操作": UI層（`TaskListPage` の `trailing: IconButton`）をタップする。
2. "UI -> Application": `ref.read(taskNotifierProvider.notifier).recordTaskExecution(task.id)` が呼ばれる。UIは「記録してほしい」と依頼するだけで、どのように保存されるかは一切知らない。
3. "Application層 (オーケストレーション)": `TaskNotifier` が現在時刻とUUIDを生成し、`TaskRecordEntity` を作成する。そして `taskRecordRepositoryProvider.addRecord` を呼び出す。
4. "Data層 (トランザクション実行)": Provider越しに注入された SQLite 用具象クラス（`LocalTaskRecordRepository` 等）が、実際の SQL トランザクションを発行してDBにレコードを保存し、同時に親Taskの時間を更新する。その後それぞれの Stream にデータ変更を通知する。
5. "Application -> UI": `build()` で返している `Stream` が、Data層からの通知を受け取り `TaskEntity` リストを検知し、`taskNotifierProvider` の状態が更新される。
6. "UIの再描画": UI層の `ref.watch` が状態の変更を受け取り、`lastRecordedAt` が自動的に新しい時間に切り替わって画面上に「たった今」と再描画される。（※将来的には、タイマーを用いて放置していても「1分前」と切り替わるコンポーネントへの改善が有効です）
