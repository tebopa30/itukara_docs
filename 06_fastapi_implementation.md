# itukara - Step 6: FastAPI Implementation (価値重視MVP版)

Flutterアプリから送信された「タスクの実行履歴（タイムスタンプ配列）」を受け取り、"「平均とのズレ」と「直近の傾向」から「行動したくなる自然な日本語メッセージ」を生成する" APIの実装です。

将来のAIモデル差し替えや、ユーザー個別分析（Firebase連携）を見据え、1ファイルにすべての機能を含めつつも「ルーター(Routers)」「スキーマ(Schemas)」「サービス(Services)」のレイヤーを意識した構造にしています。

## 1. ディレクトリ構造 (backend/)
```text
backend/
├── app/
│   ├── main.py              # アプリケーション初期化、Router登録
│   ├── api/
│   │   └── routes/
│   │       └── analyze.py   # エンドポイント定義
│   ├── schemas/
│   │   └── analyze.py       # Pydantic リクエスト/レスポンスモデル
│   └── services/
│       └── analyzer_service.py # 分析・メッセージ生成ロジック
└── requirements.txt         # 依存パッケージ (fastapi, pydantic, uvicorn等)
```

## 2. API実装コード

### 2.1 `backend/requirements.txt`
```text
fastapi
uvicorn
pydantic
```

### 2.2 スキーマ定義 `backend/app/schemas/analyze.py`
入出力のデータ型を Pydantic で厳密に定義します。
```python
from pydantic import BaseModel
from typing import List, Literal
from datetime import datetime

class TaskRecordItem(BaseModel):
    recordedAt: datetime

class AnalyzeRequest(BaseModel):
    records: List[TaskRecordItem]

class AnalyzeResponse(BaseModel):
    # 既存の統計情報
    averageIntervalMinutes: float
    latestIntervalMinutes: float
    differencePercent: float
    status: Literal["short", "normal", "long"]
    trend: Literal["stable", "getting_longer", "getting_shorter"]
    
    # 新規：信頼性・緊急度・予測・メッセージ分離
    dataPoints: int
    confidence: Literal["low", "medium", "high"]
    urgency: Literal["low", "medium", "high"]
    estimatedNextMinutes: float
    primaryMessage: str
    secondaryMessage: str
```

### 2.3 分析ロジック `backend/app/services/analyzer_service.py`
計算と日本語生成のコアルーチンです。将来LLM(OpenAI等)に差し替える場合も、このモジュールだけを変更します。

```python
from datetime import datetime
from typing import List, Tuple
from app.schemas.analyze import AnalyzeResponse, TaskRecordItem

class AnalyzerService:
    @staticmethod
    def analyze_records(records: List[TaskRecordItem]) -> AnalyzeResponse:
        # データ件数に基づく信頼度判定
        data_points = len(records)
        if data_points < 4:
            confidence = "low"
        elif data_points < 8:
            confidence = "medium"
        else:
            confidence = "high"

        # データが2件未満（間隔が計算不能）の場合は初期値を返す
        if data_points < 2:
            return AnalyzeResponse(
                averageIntervalMinutes=0.0,
                latestIntervalMinutes=0.0,
                differencePercent=0.0,
                status="normal",
                trend="stable",
                dataPoints=data_points,
                confidence=confidence,
                urgency="low",
                estimatedNextMinutes=0.0,
                primaryMessage="記録を始めてみましょう。",
                secondaryMessage="数回記録すると、いつものペースやアドバイスが確認できるようになります。"
            )

        # 1. 時間順（古い順）にソート
        sorted_records = sorted(records, key=lambda x: x.recordedAt)
        
        # 2. 全ての間隔（分）を計算
        intervals = []
        for i in range(1, len(sorted_records)):
            diff = sorted_records[i].recordedAt - sorted_records[i-1].recordedAt
            intervals.append(diff.total_seconds() / 60.0)

        # 3. 平均と最新を取得
        avg_interval = sum(intervals) / len(intervals)
        latest_interval = intervals[-1]

        # 4. 判定 (differencePercent と status と urgency)
        diff_pct = ((latest_interval - avg_interval) / avg_interval) * 100 if avg_interval > 0 else 0
        
        if diff_pct >= 20.0:
            status = "long"
        elif diff_pct <= -20.0:
            status = "short"
        else:
            status = "normal"

        abs_diff = abs(diff_pct)
        if abs_diff < 20.0:
            urgency = "low"
        elif abs_diff < 40.0:
            urgency = "medium"
        else:
            urgency = "high"

        # 5. 傾向 (trend) - 直近3つの間隔を使用
        trend = "stable"
        if len(intervals) >= 3:
            recent_3 = intervals[-3:] # 古いものから順 [t-2, t-1, latest]
            if recent_3[0] < recent_3[1] < recent_3[2]:
                trend = "getting_longer"
            elif recent_3[0] > recent_3[1] > recent_3[2]:
                trend = "getting_shorter"

        # 6. 次回予測 (estimatedNextMinutes)
        estimated_next = avg_interval - (avg_interval - latest_interval) * 0.5
        if estimated_next < 0:
            estimated_next = avg_interval # 負の値になった場合のセーフガード

        # 7. メッセージ生成
        primary, secondary = AnalyzerService._generate_messages(status, trend, urgency, confidence)

        return AnalyzeResponse(
            averageIntervalMinutes=round(avg_interval, 1),
            latestIntervalMinutes=round(latest_interval, 1),
            differencePercent=round(diff_pct, 1),
            status=status,
            trend=trend,
            dataPoints=data_points,
            confidence=confidence,
            urgency=urgency,
            estimatedNextMinutes=round(estimated_next, 1),
            primaryMessage=primary,
            secondaryMessage=secondary
        )

    @staticmethod
    def _generate_messages(status: str, trend: str, urgency: str, confidence: str) -> Tuple[str, str]:
        primary = ""
        secondary = ""

        # Primary Message: 行動や結論を優しく促す
        if status == "long":
            if urgency == "high":
                primary = "そろそろ対応してあげると良さそうです。"
            else:
                primary = "いつもより少し間隔が空いています。"
        elif status == "short":
            if urgency == "high":
                primary = "いつもより早いペースで求めているかもしれません。"
            else:
                primary = "いつもより少し早いペースです。"
        else:
            primary = "いつも通りの良いペースです。"

        # Secondary Message: 傾向の補足や安心感を与える言葉
        if confidence == "low":
            secondary = "まだ記録が少ないため、普段のペースを学習中です"
        else:
            if trend == "getting_longer":
                secondary = "ここ数回、少しずつ間隔が長くなる傾向があります"
            elif trend == "getting_shorter":
                secondary = "ここ数回、少しずつ間隔が短くなる傾向があります"
            else:
                if status == "normal":
                    secondary = "今のリズムを大切にしましょう"
                else:
                    secondary = "一時的な変化かもしれないので、無理せず様子を見てくださいね"

        return primary, secondary
```

### 2.4 ルーター定義 `backend/app/api/routes/analyze.py`
APIエンドポイントを定義します。

```python
from fastapi import APIRouter
from app.schemas.analyze import AnalyzeRequest, AnalyzeResponse
from app.services.analyzer_service import AnalyzerService

router = APIRouter()

@router.post("/analyze", response_model=AnalyzeResponse)
async def analyze_task_records(request: AnalyzeRequest):
    """
    タスクの実行履歴配列を受け取り、間隔の統計と自然言語メッセージを返します。
    """
    return AnalyzerService.analyze_records(request.records)
```

### 2.5 メインエントリ `backend/app/main.py`
アプリケーションを組み上げます。

```python
from fastapi import FastAPI
from app.api.routes import analyze

app = FastAPI(
    title="itukara Analytics API",
    description="育児タスクの傾向分析とアドバイス生成API",
    version="1.0.0"
)

# ルーターの登録
app.include_router(analyze.router, prefix="/api/v1", tags=["analyze"])

@app.get("/")
async def root():
    return {"status": "ok", "message": "itukara API is running."}
```

## 3. ロジックと構造の解説

### 関数の役割
*   `analyze_records`: Flutterから送られてきた生のタイムスタンプ配列を処理可能な形にソートし、それぞれの間隔（分）を算出します。全体の平均と最新の間隔を比較し、ルールベースで `status` や `trend` だけでなく、プロダクト品質となる **`confidence`（信頼性）、`urgency`（緊急度）、`estimatedNextMinutes`（予測）** を決定し、メッセージ生成へ引き継ぎます。
*   `_generate_messages`: 行動を促す結論（`primary`）と、不安を和らげたり状況を補足するメッセージ（`secondary`）を分離して生成し、育児アプリとして優しいトーンにします。

### 価値重視と意思決定支援への進化
このAPIは単なる統計ではありません。件数による信頼性評価(`confidence`)や緊急度の閾値(`urgency`)を持つことで、「いつものペース」のデータ基盤を構築するとともに、次回のアクションを予測(`estimatedNextMinutes`)して、より具体的で優しい言葉（「そろそろ対応してあげると良さそうです」）をFlutter UIへ提供します。

### 将来拡張への布石
*   Firebase / Auth 対応: 現状の `AnalyzeRequest` に `userId` 文字列を持たせ、`AnalyzerService` が Firestore SDK を呼び出して過去数ヶ月のデータを引いてくるよう変更すれば、すぐにユーザーごとの高度な分析に切り替えられます。
*   AIモデル差し替え: 現在は `_generate_message` 内で `if` 文によるルールベースとしていますが、将来 GPT-4o-mini 等に移行する際も、この関数の中を「プロンプト生成とOpenAI APIコール」に書き換えるだけで完結します。APIのインターフェース（入出力）は全く変わりません。

## 4. Flutterからの呼び出し例
Flutterの Data層（`task_record_repository.dart` 等外部API通信特化のリポジトリ）にて、`http` や `dio` パッケージで呼び出します。

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../domain/task_record_entity.dart';

class AnalyzeResult {
  final String primaryMessage;
  final String secondaryMessage;
  final double estimatedNextMinutes;

  AnalyzeResult({
    required this.primaryMessage,
    required this.secondaryMessage,
    required this.estimatedNextMinutes,
  });
}

Future<AnalyzeResult> fetchAdviceFromRecords(List<TaskRecordEntity> records) async {
  final url = Uri.parse('http://localhost:8000/api/v1/analyze');
  
  // APIスキーマに合わせたJSON構築
  final requestBody = {
    'records': records.map((r) => {
      'recordedAt': r.recordedAt.toIso8601String()
    }).toList(),
  };

  final response = await http.post(
    url,
    headers: {"Content-Type": "application/json"},
    body: jsonEncode(requestBody),
  );

  if (response.statusCode == 200) {
    final data = jsonDecode(response.body);
    return AnalyzeResult(
      primaryMessage: data['primaryMessage'] as String,
      secondaryMessage: data['secondaryMessage'] as String,
      estimatedNextMinutes: (data['estimatedNextMinutes'] as num).toDouble(),
    );
  } else {
    throw Exception('APIエラー: ${response.statusCode}');
  }
}
```
