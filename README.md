# Weekly Entra ID RiskUser Analytics Report

Microsoft Security Copilot 用カスタムエージェント。  
Entra ID のサインインログおよび Entra ID P2 リスク情報をビルトインの **Entra プラグイン**から収集・分析し、視認性の高い HTML レポートを生成して **Outlook メール**で SOC に通知します。  
週次スケジュール（毎週自動実行）または手動実行に対応しています。

---

## 概要

| 項目 | 内容 |
|---|---|
| **エージェント種別** | Standard Agent（スケジュール／手動トリガー型） |
| **使用モデル** | gpt-4o |
| **データソース** | Microsoft Entra ビルトインプラグイン（Sentinel 不要） |
| **通知方法** | Azure Logic App → Office 365 Outlook |
| **レポート形式** | HTML / CSS インラインスタイル（Meiryo フォント、メール互換） |
| **実行間隔** | 週次（604800 秒）または手動 |

---

## アーキテクチャ

```
Security Copilot（週次スケジュール or 手動）
  └── SocReportOrchestrator (Agent / gpt-4.1)
        │
        ├── GetEntraData          ──▶ Entra プラグイン (Microsoft Graph)
        │     サインイン統計・時系列・国別分布・リスク検出集計
        │
        ├── GetEntraSignInLogsV1  ──▶ Entra プラグイン (Microsoft Graph)
        │     失敗サインイン詳細（上位50件）・高リスクサインイン詳細
        │
        ├── GetEntraRiskyUsers    ──▶ Entra プラグイン (Microsoft Graph)
        │     リスクユーザー一覧 (High / Medium / Low)
        │
        └── [HTML レポート生成]
              ├── KPI カード ×4
              ├── 全体傾向分析・危険ユーザー兆候レポート（事実ベース）
              ├── 時系列・国別・リスク分布テーブル
              ├── 失敗サインイン詳細テーブル（エラーコード日本語説明付き）
              ├── リスクユーザー・高リスクサインイン一覧
              └── 推奨アクション（自動判定）
                    │
                    └── SendSocReportEmail
                          └── Azure Logic App ──▶ Outlook ──▶ SOC
```

---

## データソースについて — Sentinel 不使用・Entra プラグイン利用

本エージェントは **Microsoft Sentinel の KQL スキルを一切使用しません**。  
代わりに、Security Copilot に標準搭載されている **Microsoft Entra ビルトインプラグイン**のスキルを直接呼び出して Entra ID データを取得します。

### なぜ Sentinel ではなく Entra プラグインを使うのか

| 比較項目 | Sentinel KQL スキル | **Entra ビルトインプラグイン（本エージェント）** |
|---|---|---|
| **前提条件** | Sentinel ワークスペース + Entra サインインログの Log Analytics 転送設定が必要 | **不要**（Entra プラグインを Security Copilot で有効化するだけ） |
| **データの鮮度** | Log Analytics への転送遅延（数分〜数十分）が発生する場合あり | Microsoft Graph API 経由でリアルタイムデータを取得 |
| **コスト** | Sentinel ライセンス + Log Analytics データ取り込みコストが発生 | Security Copilot のライセンスのみ（追加コストなし） |
| **セットアップ工数** | Sentinel ワークスペース構成・診断設定・テーブルスキーマの把握が必要 | プラグイン有効化のみ（設定 5 分以内） |
| **Entra ID P2 リスクデータ** | `AADRiskyUsers` / `AADUserRiskEvents` テーブルへの転送設定が別途必要 | `GetEntraRiskyUsers` で直接取得可能 |

### 使用する Entra プラグインスキル

Security Copilot の組み込みプラグイン **「Microsoft Entra」**（SkillsetName: `Entra`）が提供する以下の 3 スキルのみを使用します。

| スキル名 | 取得元（Microsoft Graph API） | 取得内容 |
|---|---|---|
| `GetEntraData` | `/v1.0/` (汎用クエリ) | サインイン統計・時系列集計・国別分布・リスク検出（risk detections）集計 |
| `GetEntraSignInLogsV1` | `/auditLogs/signIns` | サインインログ詳細（失敗・エラーコード・IP・場所・リスクレベルでフィルタ） |
| `GetEntraRiskyUsers` | `/identityProtection/riskyUsers` | リスクありユーザー一覧（High / Medium / Low・atRisk 状態） |

> **有効化方法**: Security Copilot ポータル → 左メニュー「プラグイン」→「Microsoft Entra」をオンにするだけです。  
> カスタム KQL クエリ・ワークスペース ID・接続文字列などの設定は不要です。

---

## ファイル構成

```
WeeklyEntraIdRiskUserAnalyticsReport/
├── WeeklyEntraIdRiskUserAnalyticsReport.yaml      # Security Copilot エージェントマニフェスト
├── WeeklyEntraIdRiskUserAnalyticsReport_ARM.json  # Outlook 送信用 Logic App ARM テンプレート
├── WeeklyEntraIdRiskUserAnalyticsReport.html      # Plugin Card（参照用）
└── README.md
```

---

## スキル一覧

### Agent スキル

| スキル名 | 役割 |
|---|---|
| `SocReportOrchestrator` | 全ワークフローのオーケストレーター。データ収集 → 分析 → HTML 生成 → メール送信を制御する |

### 外部スキル（Entra ビルトインプラグイン）

| スキル名 | SkillSet | 役割 |
|---|---|---|
| `GetEntraData` | `Entra` | サインイン統計・時系列・国別分布・リスク検出の集計クエリ（Microsoft Graph） |
| `GetEntraSignInLogsV1` | `Entra` | サインインログの詳細フィルタクエリ（失敗・高リスク） |
| `GetEntraRiskyUsers` | `Entra` | リスクありユーザー一覧の取得（High / Medium / Low） |

### LogicApp スキル

| スキル名 | 役割 |
|---|---|
| `SendSocReportEmail` | HTML レポートを Outlook 経由で SOC に送信する |

---

## 生成レポートの仕様

### デザイン
- **フォント**: Meiryo, 'メイリオ', 'Segoe UI', sans-serif
- **プライマリカラー**: #0078D4（Microsoft Blue）
- **レイアウト**: `<table>` ベース・CSS インラインスタイル（Outlook 完全互換）
- **幅**: max-width: 680px（メール表示最適化）
- **JavaScript / 外部 CDN**: 一切不使用（Outlook 互換のため）

### セクション構成

| セクション | 内容 |
|---|---|
| **ヘッダー** | タイトル・分析期間（過去7日間）・生成日時（JST） |
| **セクション 1** | KPI サマリーカード ×4（総サインイン数・成功率・リスクユーザー数・高リスクサインイン数） |
| **セクション 2** | **全体傾向分析・危険ユーザー兆候レポート**（事実ベース分析・推奨優先対応ユーザー一覧） |
| **セクション 3** | 時系列サインイン（日付 / 成功数 / 失敗数） |
| **セクション 4** | 国別サインイン分布（上位10カ国、海外ハイライト） |
| **セクション 5** | リスクユーザー分布（リスクレベル別件数、色分け） |
| **セクション 6** | リスクイベント種別（イベント種別 / リスクレベル / 件数） |
| **セクション 7** | 失敗サインイン詳細（ユーザー / エラーコード / **エラー理由** / 場所 / IP / 発生数） |
| **セクション 8** | リスクユーザー一覧（UPN / リスクレベル / リスク状態 / 最終更新） |
| **セクション 9** | 高リスクサインイン一覧（ユーザー / 日時 / アプリ / 国 / リスク詳細） |
| **セクション 10** | 推奨アクション（データに基づき自動判定） |
| **フッター** | 自動生成注記・エージェント名・日時 |

### セクション 2「全体傾向分析・危険ユーザー兆候レポート」詳細

取得したすべてのデータを事実ベースで分析し、以下の5つの脅威シグナルを判定します。

| 判定項目 | 検出条件 |
|---|---|
| **ブルートフォース兆候** | 同一ユーザーへ7日間で5件以上の失敗 / 同一 IP から複数ユーザーへの失敗 |
| **異常アクセス元** | 日本国外からのサインイン / 地理的に不可能なログイン |
| **MFA・CA 関連エラー** | エラーコード 50074・50076・50079・53003 等の反復発生 |
| **アクティブリスクユーザー** | `atRisk` 状態の High・Medium リスクユーザー |
| **高リスクサインイン** | Entra ID P2 判定による High リスクサインイン |

判定結果は「**推奨優先対応ユーザー一覧**」テーブルに UPN・リスク種別・根拠・推奨対応の形でまとめられます。

### エラーコード日本語説明

セクション 7 の失敗サインイン詳細テーブルには、Entra ID サインインエラーコードの日本語説明（約100件対応）が自動付与されます。

| 代表コード | 説明 |
|---|---|
| `50074` | MFA チャレンジが必要（Conditional Access による強力な認証要求） |
| `50199` | ユーザー確認（セキュリティ確認）が必要（インタラクティブ操作が必要） |
| `70044` | セッションが期限切れか、または条件付きアクセスポリシー変更によりトークンが無効化された |
| `50053` | アカウントがロックされている（不正な ID またはパスワードで過多にサインインを試みた） |
| `53003` | 条件付きアクセスポリシーによってアクセスがブロックされた |

### 推奨アクションの自動判定ロジック

| 条件 | 推奨アクション |
|---|---|
| High リスクユーザーが 1 名以上 | 即時パスワードリセットおよび MFA 強制を推奨 |
| 失敗サインインが全体の 10% 超 | ブルートフォース攻撃の可能性を調査推奨 |
| 海外からのサインインを検出 | Conditional Access の地理的制限を確認推奨 |
| Medium/High リスクイベントが 5 件超 | Identity Protection ポリシーの強化を推奨 |

---

## 前提条件

| 要件 | 詳細 |
|---|---|
| Microsoft Security Copilot | ライセンス済み・有効化済み |
| Microsoft Entra プラグイン | Security Copilot で有効化済み |
| Entra ID P2 ライセンス | リスクユーザー・リスク検出機能に必要 |
| Azure サブスクリプション | Logic App のデプロイ先 |
| Office 365 ライセンス | Outlook 送信アカウント |

> **Sentinel は不要です。** データソースは Microsoft Entra ビルトインプラグイン（Microsoft Graph API）を使用します。

---

## セットアップ手順

### ステップ 1 — Logic App のデプロイ

ARM テンプレートをデプロイします。`emailAddress` パラメーターに **SOC のメールアドレス**を指定してください。

```bash
az deployment group create \
  --resource-group <YOUR-RESOURCE-GROUP> \
  --template-file WeeklyEntraIdRiskUserAnalyticsReport_ARM.json \
  --parameters emailAddress="soc-team@contoso.com"
```

### ステップ 2 — Office 365 API 接続の認証

Azure ポータルで `office365` API 接続を開き、**Outlook アカウントでサインイン**して認証を完了します。

```
Azure ポータル → リソースグループ → office365 (API 接続) → [接続の編集] → サインイン
```

### ステップ 3 — Security Copilot へのアップロード

1. [Security Copilot ポータル](https://securitycopilot.microsoft.com) にアクセス
2. 左メニュー **「プラグイン」** → **「カスタムプラグインをアップロード」**
3. `WeeklyEntraIdRiskUserAnalyticsReport.yaml` を選択してアップロード

### ステップ 4 — プラグイン設定

インストール後、プラグインの設定画面で以下の 3 項目を入力します。

| 設定項目 | 内容 | 例 |
|---|---|---|
| **Logic App サブスクリプション ID** | Logic App がデプロイされている Azure サブスクリプション ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **Logic App リソースグループ** | Logic App のリソースグループ名 | `rg-securitycopilot` |
| **Logic App ワークフロー名** | デプロイした Logic App のワークフロー名 | `SocReportEmailLogicApp` |

### ステップ 5 — 動作確認

エージェントを有効化後、手動で実行して動作を確認します。  
Security Copilot のエージェント画面から対象エージェントを選択し、**「今すぐ実行」** をクリックします。

---

## スケジュール実行について

本エージェントは `DefaultPollPeriodSeconds: 604800`（7日間）で**週次自動実行**するよう設定されています。  
初回有効化後、Security Copilot が自動的に週次スケジュールを開始します。

手動実行はいつでも Security Copilot のエージェント管理画面から行えます。

---

## トラブルシューティング

| 症状 | 確認ポイント |
|---|---|
| Logic App がトリガーされない | プラグイン設定の Logic App 名・リソースグループ・サブスクリプション ID を確認 |
| メールが届かない | Azure ポータルで `office365` API 接続の認証状態を確認 |
| Entra データが取得できない | Security Copilot で「Entra」プラグインが有効化されているか確認 |
| リスクユーザーが常に 0 件 | Entra ID P2 ライセンスが割り当てられているか確認 |
| プラグイン設定項目が表示されない | YAML を再インストール（削除 → 再アップロード） |


Microsoft Security Copilot 用カスタムエージェント。  
Entra ID のサインインログおよび Entra ID P2 リスク情報をビルトインの **Entra プラグイン**から収集・分析し、Chart.js グラフ付き HTML レポートを生成して **Outlook メール**で CISO に通知します。

---

## 概要

| 項目 | 内容 |
|---|---|
| **エージェント種別** | Interactive Agent（チャット対話型） |
| **使用モデル** | gpt-4.1 |
| **データソース** | Microsoft Entra ビルトインプラグイン（Sentinel 不要） |
| **通知方法** | Azure Logic App → Office 365 Outlook |
| **レポート形式** | HTML / CSS（Meiyro フォント、Chart.js グラフ×4） |

---

## アーキテクチャ

```
CISO
  │  「週次レポートを送信して」
  ▼
Security Copilot
  └── CisoReportOrchestrator (Interactive Agent / gpt-4.1)
        │
        ├── GetEntraData          ──▶ Entra プラグイン (Microsoft Graph)
        │     統計サマリー・時系列・国別分布・リスク検出集計
        │
        ├── GetEntraSignInLogsV1  ──▶ Entra プラグイン (Microsoft Graph)
        │     失敗サインイン詳細・高リスクサインイン詳細
        │
        ├── GetEntraRiskyUsers    ──▶ Entra プラグイン (Microsoft Graph)
        │     リスクユーザー一覧 (High / Medium / Low)
        │
        └── [HTML レポート生成]
              ├── KPI カード ×4（総サインイン・成功率・リスクユーザー・高リスクサインイン）
              ├── Chart.js グラフ ×4（折れ線・ドーナツ・横棒・棒）
              ├── 詳細テーブル ×3
              └── 推奨アクション（自動判定）
                    │
                    └── SendCisoReportEmail
                          └── Azure Logic App ──▶ Outlook ──▶ CISO
```

---

## ファイル構成

```
EntraIdCiscoReportAgent/
├── EntraIdCisoReportAgent.yaml          # Security Copilot エージェントマニフェスト
├── CisoReportEmailLogicApp_ARM.json     # Outlook 送信用 Logic App ARM テンプレート
├── EntraIdCisoReportAgent_card.html     # Plugin Card（参照用）
└── README.md
```

---

## スキル一覧

### Agent スキル

| スキル名 | 役割 |
|---|---|
| `CisoReportOrchestrator` | 全ワークフローのオーケストレーター。データ収集 → HTML 生成 → メール送信を制御する |

### 外部スキル（Entra ビルトインプラグイン）

| スキル名 | SkillSet | 役割 |
|---|---|---|
| `GetEntraData` | `Entra` | 統計・時系列・国別分布・リスク検出の集計クエリ（Microsoft Graph） |
| `GetEntraSignInLogsV1` | `Entra` | サインインログの詳細フィルタクエリ（失敗・高リスク） |
| `GetEntraRiskyUsers` | `Entra` | リスクありユーザー一覧の取得 |

### LogicApp スキル

| スキル名 | 役割 |
|---|---|
| `SendCisoReportEmail` | HTML レポートを Outlook 経由で CISO に送信する |

---

## 生成レポートの仕様

### デザイン
- **フォント**: Meiryo, "メイリオ", sans-serif
- **プライマリカラー**: #0078D4（Microsoft ブランド）
- **グラフ**: Chart.js（CDN）
- **幅**: max-width: 900px（Outlook メール対応）

### セクション構成

| セクション | 内容 |
|---|---|
| ヘッダー | タイトル・生成日時（JST）・分析期間 |
| KPI カード ×4 | 総サインイン数・成功率・リスクユーザー数・高リスクサインイン数 |
| グラフ 1 | サインイン時系列（折れ線グラフ）：成功（青）/ 失敗（赤） |
| グラフ 2 | リスクユーザー分布（ドーナツグラフ）：High / Medium / Low |
| グラフ 3 | リスクイベント種別（横棒グラフ） |
| グラフ 4 | 国別サインイン分布（棒グラフ、上位 10 カ国） |
| テーブル 1 | 失敗サインイン詳細（ユーザー / エラーコード / 場所 / IP / 発生数） |
| テーブル 2 | リスクユーザー一覧（UPN / リスクレベル / リスク状態 / 最終更新） |
| テーブル 3 | 高リスクサインイン一覧（ユーザー / 日時 / アプリ / 国 / リスク詳細） |
| 推奨アクション | データに基づき自動判定したセキュリティ推奨事項 |
| フッター | 自動生成注記・エージェント名・日時 |

### 推奨アクションの自動判定ロジック

| 条件 | 推奨アクション |
|---|---|
| High リスクユーザーが 1 名以上 | 即時パスワードリセットおよび MFA 強制を推奨 |
| 失敗サインインが全体の 10% 超 | ブルートフォース攻撃の可能性を調査推奨 |
| 海外からのサインインを検出 | Conditional Access の地理的制限を確認推奨 |
| Medium/High リスクイベントが 5 件超 | Identity Protection ポリシーの強化を推奨 |

---

## 前提条件

| 要件 | 詳細 |
|---|---|
| Microsoft Security Copilot | ライセンス済み・有効化済み |
| Microsoft Entra プラグイン | Security Copilot で有効化済み |
| Entra ID P2 ライセンス | リスクユーザー・リスク検出機能に必要 |
| Azure サブスクリプション | Logic App のデプロイ先 |
| Office 365 ライセンス | Outlook 送信アカウント |

> **Sentinel は不要です。** データソースは Microsoft Entra ビルトインプラグイン（Microsoft Graph API）を使用します。

---

## セットアップ手順

### ステップ 1 — Logic App のデプロイ

```bash
az deployment group create \
  --resource-group <YOUR-RESOURCE-GROUP> \
  --template-file CisoReportEmailLogicApp_ARM.json \
  --parameters defaultToAddress="ciso@contoso.com"
```

### ステップ 2 — Office 365 API 接続の認証

Azure ポータルで `office365` API 接続を開き、**Outlook アカウントでサインイン**して認証を完了します。

```
Azure ポータル → リソースグループ → office365 (API 接続) → [接続の編集] → サインイン
```

### ステップ 3 — YAML のプレースホルダー置換

`EntraIdCisoReportAgent.yaml` 内の以下の値を実際の値に置換します。

| プレースホルダー | 置換内容 |
|---|---|
| `<YOUR-SUBSCRIPTION-ID>` | Azure サブスクリプション ID |
| `<YOUR-RESOURCE-GROUP>` | Logic App のリソースグループ名 |

### ステップ 4 — Security Copilot へのアップロード

1. [Security Copilot ポータル](https://securitycopilot.microsoft.com) にアクセス
2. 左メニュー **「プラグイン」** → **「カスタムプラグインをアップロード」**
3. `EntraIdCisoReportAgent.yaml` を選択してアップロード
4. **「アクティブエージェント」** の設定でエージェントを有効化

### ステップ 5 — 実行確認

チャット画面で以下のスターターショットから実行：

```
過去 7 日間の Entra ID サインインとリスク情報をまとめて CISO レポートを作成し、メールで送信してください。
```

---

## 使用例

### スターターショット（チャット起動時に表示）

```
# 週次レポート生成
過去 7 日間の Entra ID サインインとリスク情報をまとめて CISO レポートを作成し、メールで送信してください。

# 月次リスク分析
過去 30 日間のリスクユーザーとサインイン失敗の傾向を分析して CISO レポートを送付してください。

# 高リスクユーザー集計
高リスクユーザーだけに絞ったサマリーレポートを作成してください。
```

### フォローアップ質問の例

```
直近 24 時間のサインイン失敗をグラフ化してレポートしてください。
リスクイベントの種類別内訳を表示してください。
国別サインイン分布を分析してください。
レポートを再生成して再送信してください。
```

---

## ARM テンプレートパラメーター

`CisoReportEmailLogicApp_ARM.json` で設定可能なパラメーター：

| パラメーター | 既定値 | 説明 |
|---|---|---|
| `defaultToAddress` | `ciso@contoso.com` | CISO のメールアドレス（デフォルト宛先） |
| `location` | `resourceGroup().location` | デプロイリージョン |

---

## Logic App のフロー

```
HTTP トリガー (manual)
  │  受信: { subject, htmlBody, toAddress }
  ▼
Parse JSON
  ▼
Send CISO Report Email (Office 365 Outlook)
  │  To:        toAddress（未指定時は defaultToAddress）
  │  Subject:   subject
  │  Body:      htmlBody（IsHtml: true）
  │  Importance: High
  ▼
Response (200 OK)
```

---

## セキュリティ考慮事項

- Logic App のエンドポイントは Security Copilot からのみ呼び出す設計です
- Office 365 接続の認証情報は Azure Key Vault での管理を推奨します
- メール送信アカウントには最小権限（Mail.Send スコープのみ）を付与してください
- Logic App の `Access control (IP restrictions)` を設定することを推奨します

---

## ライセンス

MIT License

---

## 関連リンク

- [Microsoft Security Copilot ドキュメント](https://learn.microsoft.com/ja-jp/copilot/security/)
- [Security Copilot カスタムプラグイン](https://learn.microsoft.com/ja-jp/copilot/security/custom-plugins)
- [Microsoft Entra ID Protection](https://learn.microsoft.com/ja-jp/entra/id-protection/)
- [Azure Logic Apps](https://learn.microsoft.com/ja-jp/azure/logic-apps/)
