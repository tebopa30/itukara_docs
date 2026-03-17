# itukara - Step 5: Data Layer Implementation (SQLite)

Step 3.5で定義したインターフェース（Domain層）を満たす、具体的なデータアクセス層（Data層）の実装を行います。
ローカルファーストの要件を満たすため、Flutterの標準的なSQLiteパッケージである **`sqflite`** を用いた実装例となります。

## 1. データベース初期化と Provider (core/providers/database_provider.dart)

Sqfliteのデータベースインスタンスを管理・提供するProviderを作成します。

> **Note**: 実装コードは `frontend/app/lib/core/providers/database_provider.dart` に抽出済みです。

## 2. LocalTaskRepository 実装 (features/task/data/local_task_repository.dart)

`TaskRepository` の具象クラスです。
内部で `StreamController` を用いて変更をブロードキャストします。

> **Note**: 実装コードは `frontend/app/lib/features/task/data/local_task_repository.dart` に抽出済みです。

## 3. LocalTaskRecordRepository 実装 (features/task/data/local_task_record_repository.dart)

ここが肝となる修正点です。「TaskRecordの実績保存」と「TaskのlastRecordedAt更新」がSQLiteの **トランザクションとして一体化** されており、かつ **リアルタイムのStream配信** を行います。

> **Note**: 実装コードは `frontend/app/lib/features/task/data/local_task_record_repository.dart` に抽出済みです。

## 4. DI (main.dart での Provider オーバーライド)

Data層の依存関係（`LocalTaskRecordRepository` が `LocalTaskRepository` を求める構造）を ProviderScope で適切に依存注入します。

> **Note**: 実装でのDI構成は `frontend/app/lib/main.dart` に抽出済みです。

## 5. 設計解説（なぜこのような実装なのか）

### ① トランザクション設計の解説
SQLite(Sqflite)には Firebase の Batched Writes のような「複数コレクションにまたがるアトミックな書き込み」を透過的に行うレイヤーが標準ではありません。
そのため、`Database.transaction((txn) async { ... })` を使う必要があります。
このトランザクション処理を Application(UseCase) 層に記述してしまうと、UseCase が DBコネクションなどの Data層固有の知識を持ってしまい クリーンアーキテクチャに違反します。

そこで、**`LocalTaskRecordRepository.addRecord` の Data層実装内** にトランザクションを密封しました。Application 層（`TaskNotifier` など）からは `addRecord(record)` を呼ぶだけで「記録の保存と親Task日時更新の2つが必ず同時に成功する（失敗すれば共に破棄される）」ことが保証されます。

### ② Stream更新フローの説明
`sqflite` そのものは Firestore の `.snapshots()` のようなリアルタイム検知を持っていません。
それを補うため、Data層クラスの中に `StreamController.broadcast()` を持たせました。
トランザクションによる書き込みが成功した後のみ、以下のように **2つの Controller** へデータを流し、UIを同期させます。
1. `_notifyWatchersForTask`: このタスクの履歴リスト画面を開いているUIを更新。
2. `_taskRepo.notifyWatchers()`: ホームのタスク一覧画面の `lastRecordedAt` 表示を更新。

これにより、Flutterの Riverpod を用いた監視（`ref.watch`）がリアクティブに発火し、遅延や非同期ズレなく画面が再描画されます。
