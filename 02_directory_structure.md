# itukara - Step2: ディレクトリ構成設計

前回のアーキテクチャ設計（Flutter 4層クリーンアーキテクチャ、Riverpod、FastAPI、IAP対応、将来的なFirebase移行）に基づく、プロジェクトのディレクトリ構造設計です。

## 全体ツリー構造

```text
itukara/
├── docs/                      # ドキュメント群（設計書など）
├── app/                       # Flutterモバイルアプリ本体
│   ├── lib/
│   │   ├── core/              # アプリ全体で共通のレイヤー非依存要素
│   │   │   ├── constants/     # 定数（色、テーマ、環境変数、固定文字列など）
│   │   │   ├── errors/        # 共通例外クラス（Failure, Exception）
│   │   │   ├── providers/     # グローバルレベルのProvider（DB初期化など）
│   │   │   ├── routers/       # GoRouter等のルーティング定義
│   │   │   └── utils/         # 汎用ユーティリティ拡張（Extensionなど）
│   │   ├── features/          # 機能単位の分割 (Feature-first)
│   │   │   ├── auth/          # 認証・ユーザー管理機能
│   │   │   │   ├── application/
│   │   │   │   ├── domain/
│   │   │   │   ├── data/
│   │   │   │   └── presentation/
│   │   │   ├── task/          # タスク（カウントアップ）管理機能
│   │   │   │   ├── application/  # TaskNotifier / 状態管理とユースケース処理
│   │   │   │   ├── domain/       # TaskEntity / TaskRepository(interface)
│   │   │   │   ├── data/         # LocalTaskRepository / 外部API通信
│   │   │   │   └── presentation/ # View (Pages / Widgets)
│   │   │   ├── iap/           # 課金・プレミアム機能
│   │   │   │   ├── application/  # PremiumStatusNotifier
│   │   │   │   ├── domain/       # PremiumStatusEntity / BillingRepository(interface)
│   │   │   │   ├── data/         # in_app_purchaseを利用した実際の実装クラス
│   │   │   │   └── presentation/ # ペイウォール画面 / 購入ボタン
│   │   │   └── settings/      # 設定画面
│   │   └── main.dart          # エントリーポイント。ProviderScopeによりDIを起動
│   │
│   └── test/                  # モバイルアプリのテスト
│       ├── core/              # Core層の単体テスト
│       ├── features/          # 各機能のテスト
│       │   └── task/
│       │       ├── application/  # NotifierやUseCaseの単体テスト
│       │       ├── domain/       # Entityおよび純粋なドメインロジックのテスト
│       │       ├── data/         # RepositoryやDataSourceのテスト（Mock化）
│       │       └── presentation/ # Viewのウィジェットテストやゴールデンテスト
│       └── integration_test/  # 統合テスト（エンドツーエンド）
│
└── backend/                   # FastAPIサーバー (AI連携等)
    ├── app/                   # パッケージ本体
    │   ├── api/               # API層
    │   │   ├── routes/        # 各ルーターの定義 (auth.py, ai.py 等)
    │   │   └── dependencies.py# API共通依存（認証トークン検証やDBセッション）
    │   ├── core/              # アプリのコア設定
    │   │   ├── config.py      # 環境変数 / Pydantic BaseSettings
    │   │   └── security.py    # パスワードハッシュやJWT生成ロジック
    │   ├── models/            # データ層モデル (SQLAlchemy等)
    │   ├── schemas/           # Pydanticスキーマ (APIのリクエスト/レスポンス型定義)
    │   ├── services/          # ビジネスロジック・外部API呼び出し
    │   │   └── ai_predictor.py# AIによる解析やアドバイス生成ロジック
    │   └── main.py            # FastAPIエントリーポイント・ルーターの登録
    ├── tests/                 # バックエンドテスト
    │   ├── conftest.py        # Pytestのフィクスチャ設定 (DBモックなど)
    │   ├── api/               # エンドポイント結合テスト
    │   └── services/          # サービス・ビジネスロジックの単体テスト
    ├── requirements.txt       # 依存ライブラリ（pyproject.tomlも可）
    └── Dockerfile             # デプロイ用コンテナ設定
```

## 各フォルダの責務説明

### Flutter側 (`app/lib/`)
*   **core/**
    特有の機能（タスクやユーザーなど）に依存しない全体共通の処理を配置します。ルーティング設定やテーマ設定、再利用可能なユーティリティなどが含まれます。
*   **features/**
    機能群（タスク、課金、設定など）ごとにフォルダを切る「Feature-first」アプローチを採用します。特定の機能に関連するファイル群が一箇所にまとまり、スケール時にも影響範囲が明確になります。各機能内はクリーンアーキテクチャの4層に分割されます。
    *   **presentation/**: 画面（Page）と部分的なUIパーツ（Widget）。UI構築とユーザーアクションの受け付けのみ行います。ビジネスロジックは持ちません。
    *   **application/**: 状態管理とユースケースを担います。Riverpodの `Notifier` や `AsyncNotifier` がここに入り、UIからのアクションを受けてDomain/Data層を呼び出し、結果をStateとして保持します。
    *   **domain/**: ソフトウェアの核となるデータ用 Entity（クラス）と、Repositoryが満たすべき Interface（`abstract class`）を定義します。Flutter依存（UI）やDB依存を持たせない純粋なDartコードとします。
    *   **data/**: Domain層で定義した抽象インターフェースの実装コード（Concrete Class）を配置します。ローカルDB (SQLiteなど) や、外部連携ロジックを直に扱い、その結果をDomainのEntityにマッピングして返します。

### FastAPI側 (`backend/`)
*   **api/**: クライアントからのHTTPリクエストのルーティングを受け持ちます。FlutterでいうPresentation層に近い役割を持ちます。
*   **schemas/**: Pydanticによる厳密な型定義・バリデーションを行い、入出力のデータ構造を保証します。
*   **models/**: SQLAlchemyなどのデータベースとのマッピングを行うデータモデルを定義します。
*   **services/**: システムのビジネスロジック本体です。将来的なAI連携など、Flutter側で行うと重い処理や、バックエンドで隠蔽すべき知識をここに統合します。

## 将来拡張ポイントの明示

1.  **Firebaseへの差し替え (Flutter / Data Layer)**
    *   `app/lib/features/task/domain/task_repository.dart` (抽象インターフェース) に対して、初期段階では `data/local_task_repository.dart` (SQLite等) を作ります。
    *   将来のFirebase移行時は、既存のロジックに手を加えるのではなく `data/firebase_task_repository.dart` を新規実装します。
    *   Riverpodの「DI（依存性の注入）」定義部分で、返却するインスタンスを `FirebaseTaskRepository` に切り替えるだけで、Presentation/Application 層を一切変更せずにバックエンドを差し替え完了できる仕組みになっています。
2.  **IAP拡張 (Flutter / features/premium)**
    *   課金機能は完全に独立した `features/premium/` パッケージとして設けます。
    *   Riverpod を通じて `PremiumStatusNotifier` をグローバルに公開し、各機能 (例: task) の Presentation層で `ref.watch(premiumStatusProvider)` を監視することで、特定の機能にだけペイトウォールや制限をかけるといった制御が簡単に行えます。
3.  **AI連携・高度化 (FastAPI / services)**
    *   Flutter側に推論ロジックやAPIのシークレットキーを直書きせず、分析処理などを FastAPIの `services/ai_predictor.py` などで吸収します。将来的に ChatGPT や Gemini 等の様々なLLM利用やアルゴリズム変更に対しても、フロントエンドへ影響を与えずに柔軟な差し替え・高度化を実現します。
