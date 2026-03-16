# itukara - 全体アーキテクチャ設計

## 1. 概要
「itukara」は育児タスクのカウントアップを管理するアプリであり、将来的なスケーラビリティ（Firebase移行、AI連携、IAP課金）を考慮した商用レベルのアーキテクチャを採用します。

## 2. モバイルアプリ (Flutter) アーキテクチャ
保守性とテスト容易性を高めるため、クリーンアーキテクチャに基づく **4層レイヤーアーキテクチャ** を採用します。

### レイヤー分離
1. **Presentation Layer (UI/View)**
   - 画面、ウィジェット、状態の描画のみを責務とする。
   - `ConsumerWidget` を使用し、ViewModel (Notifier) の状態を監視。
2. **Application Layer (ViewModel/UseCase)**
   - UIからのイベントを受け取り、ビジネスロジック（Domain/Data層への指示）を実行。
   - 状態管理（StateNotifier / Notifier）を配置。
3. **Domain Layer (Entities/Interfaces)**
   - アプリケーションの中核となるデータモデル（Entity）と、RepositoryやServiceのインターフェース（抽象クラス）を定義。
   - 外部依存を持たない純粋なDartコード。
4. **Data Layer (Repositories/DataSources)**
   - Domain層で定義されたインターフェースの実装。
   - ローカルDB (SQLite), 外部API (FastAPI), IAP (課金), 将来のFirebase連携等の具体的なデータアクセス隠蔽。

### 状態管理の選定理由: **Riverpod (Notifier / AsyncNotifier)**
商用利用において、**Riverpod** を選定します。
- **選定理由**: 
  1. コンパイルセーフであり、Providerが見つからないエラーを防げる。
  2. `AsyncValue` を用いることで、ローディング・エラー・データ取得済みの状態管理が極めて簡潔かつ安全に書ける。
  3. テストが容易。
  4. グローバルに定義可能だが、各Provider間での依存関係解決（DI）が強力。

### Repositoryパターン & DI (Dependency Injection) 設計
- **Repositoryパターン**: `Data Layer` からのデータ取得・保存処理を共通のインターフェースにする。これにより、ローカルDB保存からFirebase保存への切り替えが、インターフェースの差し替えだけで可能になります。
- **DI設計**: Riverpod自体をDIコンテナとして活用します。将来のML差し替えやFirebase差し替え時には、Providerの返す実装クラスを変えるだけで済みます。

### 課金機能（IAP）の状態管理設計
- **PremiumStatus モデル**: ユーザーの課金状態（有効期限、サブスク開始日、フラグ）を保持する。
- **BillingRepository**: App Store と通信しレシート検証を行う責務。
- **PremiumStatusNotifier**: 非同期で課金状態を取得し状態を保持する。
- **アクセス制御**: 有料機能は、UI側で `ref.watch(premiumStatusProvider)` を監視し、機能制限やペイウォール画面へのルーティングを行います。

---

## 3. バックエンド (FastAPI) アーキテクチャ
Flutter側との疎結合を保ちつつ、AIなどの拡張機能を提供するマイクロサービス的な役割を持たせます。

### サーバーサイドアーキテクチャ
- **MVC/Layered Approach**: `routers` (APIエンドポイント), `services` (ビジネスロジック/AI連携), `schemas` (Pydanticモデル) に分割。
- **Strategy パターン (AI/異常検知)**: 通知機能は `AIPredictor` というインターフェースを定義し、当面はダミーを割り当て、将来的に `OpenAIProvider` を割り当てられるようにします。

## 4. 将来のFirebase移行に向けた布石
- Domain層で抽象化された `TaskRepository` の具象クラスとして、初期は `LocalTaskRepository` を利用。
- 移行時に `FirebaseTaskRepository` を実装し、Riverpodの DI 定義を切り替えます。
- ローカルからFirebaseへの「初回同期バッチ処理 (SyncService)」を用意できる設計にしておきます。
