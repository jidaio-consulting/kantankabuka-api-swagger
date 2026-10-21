# OpenAPI仕様とバックエンド実装の乖離調査レポート

- 調査日: 2026-10-21
- 対象仕様: `openapi.yml`（version 1.0.1）
- 対象実装: `kantankabuka-backend` リポジトリ `external-main` ブランチ
  - `cmd/external-api/main.go`（ルーティング）
  - `handler/external/`（ハンドラ）
  - `domain/model/external/`（リクエスト/レスポンス構造体）
  - `util/external/validator.go`（バリデーション）
  - `util/external/errors.go` / `util/errors.go`（エラーレスポンス）
  - `cmd/external-api/middleware/`（JWT認証）

## サマリ

| 項目 | 結果 |
| --- | --- |
| エンドポイント（POST 3本） | 一致 |
| transition系レスポンススキーマ | 一致 |
| calculation系レスポンススキーマ | 乖離あり（フィールド名・型・未記載フィールド） |
| リクエストスキーマ | 乖離あり（financial欠落・制約未記載） |
| エラーレスポンス | 乖離あり（401/500のメッセージ） |

一致している点:

- `POST /v1/calculation`、`POST /v1/check_required_prev3`、`POST /v1/transition` の存在・メソッド
- `transitionResult`（8フィールド・型）、`stockHolderForTransitionResponse`、`future` のスキーマ
- `response200_calculation` / `response200_transition` のトップレベル構造（`calculationResult` / `transitionResult`、`stockHolders`、`simulatorIndustryPeriod`、`inActiveCompanyReasons`）
- `client_id` ヘッダ + Bearer JWT という認証方式

## 重要度: 高（クライアント実装に影響）

### 1. 比準要素のフィールド名が大文字/小文字で不一致

- 仕様: `DividendRatio` / `ProfitRatio` / `NetAssetRatio`（openapi.yml:343-351）
- 実装: JSONキーは `dividendRatio` / `profitRatio` / `netAssetRatio`（domain/model/calculation.go:305-307）

仕様通りにパースすると値が取得できない。**仕様側を小文字始まりに修正すべき**。

### 2. 401エラーメッセージの不一致

- 仕様: 「一律『invalid authentication』を返却します」（openapi.yml:459）
- 実装: `{"error":"invalid token"}`（cmd/external-api/middleware/auth.go）

さらに仕様の401レスポンスの `example` が `'invalid primaryIndustry'` のままになっている（openapi.yml:467、400からのコピペミス）。

### 3. 500エラーメッセージの不一致

- 仕様: 「一律『internal server error』を返却します」（openapi.yml:469）
- 実装:
  - `calculation` / `transition`: 内部エラーメッセージをそのまま返す（`util.NewCustomError(500, err.Error())` → `{"error":"<内部エラー文言>"}`）
  - `check_required_prev3`: ボディなしの500（`c.Status(500)` のみ、handler/external/calculation.go:73）

内部エラー文字列の外部露出は情報漏えいの観点でも実装側の見直し候補。どちらに合わせるか要判断。

### 4. check_required_prev3 のリクエストスキーマに `financial` がない

- 仕様: `body_check_required_prev3` のスキーマは `company` と `settlements` のみ（openapi.yml:81-93）。ただしexampleには `financial` が含まれている
- 実装: `financial` は `binding:"required"` で必須（domain/model/external/calculation.go:38）

スキーマに `financial` を追加すべき。

### 5. `evaluationTypeText` のenumが壊れている

- 仕様: `[原則（相続税評価額), 原則（相続税評価額), 特例（配当還元価額）]`（openapi.yml:245）— 重複・括弧不整合・「原則（法人税評価額）」欠落
- 実装の実際の値（domain/model/external/calculation.go:171-205）:
  - `原則（相続税評価額）`（evaluationType: A）
  - `原則（法人税評価額）`（evaluationType: B）
  - `特例（配当還元価額）`（evaluationType: C）

## 重要度: 中（ドキュメント不足）

### 6. required・バリデーション制約の記載が仕様に一切ない

実装（util/external/validator.go、各構造体の `binding` タグ）には以下の制約があるが、仕様には `required` 指定や制約の記載がない:

- ほぼ全フィールドが必須（`company`、`settlements`、`financial` など）
- `settlements` の要素数: calculation / transition は **2〜3期**、check_required_prev3 は**厳密に2期**
- `future.incomes` / `future.measures` は **9要素固定**
- `stockHolders` は calculation で最低1件（`min=1`）
- `stockNumber ≤ allStock - treasuryStock`
- `allStock > treasuryStock`、`allStock ≥ 1`
- `capital` は0不可（`not_zero`）、`income` / `profitReserve` は負値許容
- `foundedYear` / `financial.year` は4桁、`month` は1〜12
- `primaryIndustry` は必須、`secondaryIndustry` / `tertiaryIndustry` は任意（早見表の値のみ）
- transition の `calculationResult` はほぼ全フィールド必須（`required,gte=0` など）

### 7. 医療法人の制約が未記載

`primaryIndustry='U'`（医療法人）のとき `evaluationType='C'`（配当還元方式）は400エラー（util/external/errors.go `ValidateMedicalCorporationStockHolder`）。また医療法人の場合 `dividendDiscountOne` / `dividendDiscountAll` は0が返る（usecase/external/calculation.go:64-68）。

### 8. calculationレスポンスの未記載フィールド

実装のレスポンスには `suggestions`、`advices` もJSONに含まれる（nullで出力される。`model.CalculationResult` を直接埋め込んでいるため）。仕様に記載するか、実装側で外部APIレスポンスから除外するか要判断。

### 9. 数値型の不一致

以下は仕様で `integer` だが実装は `float64`（小数が返りうる）:

| フィールド | 仕様 | 実装 |
| --- | --- | --- |
| `calculationResult.stockOne` | integer | float64 |
| `calculationResult.netAssetValue` | integer | float64 |
| `calculationResult.similarValue` | integer | float64 |
| `calculationResult.corporateTaxOne` | integer | float64 |
| `calculationResult.dividendDiscountOne` | integer | float64 |

仕様側を `number` に修正すべき。なお transition のリクエスト `calculationResult.stockOne` 等も実装は `*float64` 受けのため、integer送信は問題ない。

## 重要度: 低

### 10. 仕様未記載のエンドポイント・レスポンス

- `GET /v1/ping`（ヘルスチェック、認証必須）
- 404: `{"error":"Not found"}`（存在しないルートへのリクエスト時）

### 11. 仕様ファイル自体の問題

- `company` スキーマの `summary:` キーはOpenAPI 3.0のSchema Objectでは無効（`title` か `description` にすべき）（openapi.yml:148）
- `ownedStockAll` のdescriptionに `// ` の消し忘れ（openapi.yml:339）
- calculationのリクエスト例にレスポンス専用の `inActiveCompanyReasons` が混入（openapi.yml:583、実装では無視される）

## 修正方針の整理

| 項番 | 対応先 | 内容 |
| --- | --- | --- |
| 1, 4, 5, 9, 11 | 仕様書 | openapi.yml の修正で解消 |
| 6, 7, 10 | 仕様書 | 制約・エンドポイントの追記 |
| 2, 3 | 要判断 | 仕様に合わせて実装のエラーメッセージを統一するか、仕様を実態に合わせるか |
| 8 | 要判断 | 仕様に追記するか、実装でフィールドを除外するか |
