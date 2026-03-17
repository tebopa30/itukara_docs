# itukara - Step 7: Local Startup Guide

本ドキュメントは、itukaraプロジェクトのバックエンド（FastAPI）とフロントエンド（Flutter）をローカル環境で起動し、相互に通信させるための実践的な完全手順書です。初心者でも詰まらずに動作確認まで完了できるように設計されています。

## ① backend（FastAPI）の起動手順

まず、分析ロジックを提供するAPIサーバーを立ち上げます。ターミナル（Windowsの場合はPowerShellまたはコマンドプロンプト）を開いてください。

**1. backendディレクトリへ移動**
```bash
cd backend
```

**2. 仮想環境（venv）の作成と有効化**
プロジェクト固有のPythonパッケージを管理するために仮想環境を作成します。
*   **macOS / Linux:**
    ```bash
    python3 -m venv venv
    source venv/bin/activate
    ```
*   **Windows:**
    ```powershell
    python -m venv venv
    .\venv\Scripts\activate
    ```
*(ターミナルのプロンプトの先頭に `(venv)` と表示されれば有効化成功です)*

**3. 依存パッケージのインストール**
```bash
pip install -r requirements.txt
```

**4. サーバーの起動**
```bash
uvicorn app.main:app --reload
```
*(起動成功時: `Uvicorn running on http://127.0.0.1:8000` と表示されます)*

---

## ② API動作確認

サーバーが立ち上がったら、Flutterから繋ぐ前にAPI単体で正しく動くか確認します。

**ブラウザでの確認 (自動生成ドキュメント - Swagger UI)**
ブラウザで以下のURLにアクセスしてください。対話的なAPIドキュメントが表示されます。
*   http://localhost:8000/docs
*(ここで `/api/v1/analyze` メソッドを開き、「Try it out」から直接リクエストを送ってテストすることも可能です)*

**curlによるテスト**
別のターミナルを開き、以下のコマンドをそのままコピー＆ペーストして実行してください。

```bash
curl -X 'POST' \
  'http://localhost:8000/api/v1/analyze' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "records": [
    { "recordedAt": "2026-03-10T10:00:00" },
    { "recordedAt": "2026-03-10T12:00:00" },
    { "recordedAt": "2026-03-10T14:30:00" },
    { "recordedAt": "2026-03-10T17:15:00" },
    { "recordedAt": "2026-03-10T20:45:00" }
  ]
}'
```

**正常レスポンス例**
異常なデータがフィルタされず、正しく統計・分析された結果が以下のように返ってくれば成功です。
```json
{
  "averageIntervalMinutes": 161.25,
  "latestIntervalMinutes": 210.0,
  "differencePercent": 30.2,
  "status": "long",
  "trend": "getting_longer",
  "dataPoints": 4,
  "confidence": "medium",
  "urgency": "medium",
  "estimatedNextMinutes": 136.9,
  "primaryMessage": "いつもより少し間隔が空いています。",
  "secondaryMessage": "ここ数回、少しずつ間隔が長くなる傾向があります"
}
```

---

## ③ Flutterからの接続方法

FlutterアプリからローカルのFastAPIへリクエストを送信する際、使用するシミュレータ・エミュレータによって**接続先URL（Host）**が異なります。ここが一番のハマりポイントです。

**1. エンドポイントの動的設定**
Dartコード（例: `api_client.dart` や定数ファイル）で、`Platform` に応じてURLを切り替える実装を推奨します。

```dart
import 'dart:io' show Platform;
import 'package:flutter/foundation.dart' show kIsWeb;

String getApiBaseUrl() {
  if (kIsWeb) {
    return 'http://localhost:8000';
  }
  // ⚠ Androidエミュレータは自分自身(localhost)ではなく、PC本体のlocalhostを10.0.2.2で参照します
  if (Platform.isAndroid) {
    return 'http://10.0.2.2:8000';
  }
  // iOSシミュレータはMacのlocalhostをそのまま参照可能です
  return 'http://localhost:8000';
}
```

**2. 呼び出しの実装確認（httpパッケージの例）**
```dart
Future<AnalyzeResult> fetchAdviceFromRecords(List<TaskRecordEntity> records) async {
  final baseUrl = getApiBaseUrl();
  final url = Uri.parse('$baseUrl/api/v1/analyze');
  
  // 以降は Step 6 で実装した POST 処理と同じ
}
```

---

## ④ よくあるエラーと対処

*   **CORSエラー (Flutter Webでデバッグしている場合)**
    *   **症状:** Webブラウザで実行した際、`XMLHttpRequest error` や `CORS policy` といった赤いエラーが出る。
    *   **原因/対策:** ブラウザのセキュリティ制約です。`backend/app/main.py` に `CORSMiddleware` が設定されている必要があります。*(※今回の更新で main.py に設定済みですので、通常は発生しません)*
*   **接続できない (SocketException / Connection Refused)**
    *   **原因:** サーバーが起動していないか、指定しているURLが間違っています。
    *   **対策:** Androidエミュレータを使っている場合、必ず `10.0.2.2:8000` を呼び出しているか確認してください。もし **実機（iPhone/Android）** でテストする場合は、PCのローカルIPアドレス（例: `http://192.168.1.5:8000`）を指定し、FastAPI側を `uvicorn app.main:app --host 0.0.0.0 --reload` で起動する必要があります。
*   **JSONフォーマットエラー (422 Unprocessable Entity)**
    *   **原因:** Flutterから送信したJSONのキー名やデータ型が、FastAPI側のモデル（例: `recordedAt`）と一致していません。
    *   **対策:** 送信直前の `jsonEncode()` の結果を `print` し、Swagger UI (`/docs`) の成功するリクエストボディと見比べてください。

---

## ⑤ 動作確認チェックリスト

- [ ] ターミナルで `uvicorn` がエラーなく起動している
- [ ] ブラウザで `http://localhost:8000/docs` が開ける
- [ ] 別のターミナルから `curl` コマンドを叩き、正しいJSONレスポンスが返ってくる
- [ ] FlutterのUI上で「たった今」などの記録後に、APIから取得した『そろそろ対応してあげると良さそうです』等のメッセージが表示される

---

## ⑥ 開発効率を上げるTips

*   **`--reload` の恩恵:** 起動コマンドに付与した `--reload` は非常に強力です。Pythonのロジック（`analyzer_service.py` など）を編集して保存するだけで、自動的にサーバーが再起動します。ターミナルで手動で再起動する必要はありません。
*   **ログの見方:** FastAPIで何かエラー（コードの書き間違い等）が起きた場合、ターミナルに数行のエラーログ（スタックトレース）が出力されます。一番下のエラーメッセージ（例: `KeyError`, `TypeError`）を読めば原因の9割が分かります。
*   **FastAPIのSwagger UI (`/docs`):** 新しいフィールドを追加したり判定ロジックを変えた際、わざわざFlutterを動かしてテストするのは非効率です。**必ずブラウザ上の `/docs` でテストリクエストを送り、期待したレスポンスが返ってくるか確認してから Flutter側の開発に戻る** のが、バックエンド開発の鉄則です。
