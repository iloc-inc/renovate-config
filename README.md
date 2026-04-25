# renovate-config

共有 Renovate 設定 - 依存関係自動更新 (Go/Terraform/Hugo)

## 概要

iloc全体での統一された依存関係更新管理を提供する共有Renovate設定システムです。

### 目的とメリット

- **統一ガバナンス**: 全プロジェクトの依存関係更新ポリシーを一元管理
- **技術固有最適化**: Go、Terraform、Hugo向けの専用更新戦略
- **自動化バランス**: 重要度に基づく自動/手動マージ戦略
- **タイムゾーン最適化**: 日本営業時間での更新スケジュール
- **サプライチェーン攻撃対策**: Digest Pinning + 脆弱性アラート自動対応

## 🔒 サプライチェーン攻撃対策

`default.json`で全プロジェクトに以下の対策を一括適用しています。

### Digest Pinning（`pinDigests: true`）

依存関係をバージョンタグではなくSHA256ハッシュで固定し、コメントでバージョンを表示します。

```yaml
# Before: タグは上書き可能で危険
- uses: actions/checkout@v4

# After: ハッシュはコンテンツの指紋なので改ざん不可
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

**防御できる攻撃**:
- タグ上書き攻撃（既存バージョンへの悪意あるコード注入）
- レジストリ侵害（パッケージの差し替え）
- メンテナアカウント乗っ取り（更新時のレビューで検知可能）

**Renovateとの連携**: ハッシュの更新PRを自動作成し、コメントのバージョンも自動更新するため、運用負荷なし。

### GitHub Actions Digest Pinning（`helpers:pinGitHubActionDigests`）

GitHub Actionsのアクション参照に特化したdigest pinning。全ワークフローで自動適用。

### 脆弱性アラート自動対応（`vulnerabilityAlerts`）

GitHubのセキュリティアドバイザリと連携し、脆弱性が検出された依存関係に対して`security`ラベル付きPRを自動作成。スケジュールに関係なく即座に対応。

## 設定ファイル

- **`default.json`**: 全プロジェクト共通（Asia/Tokyoタイムゾーン、平日朝8時前スケジュール、サプライチェーン攻撃対策）
- **`go.json`**: Go専用（.go-version、go.mod、golangci-lint管理）
- **`hugo.json`**: Hugo専用（Netlify/GitHub Actions版本管理）
- **`terraform.json`**: Terraform専用（AWS provider、バージョン制約管理）

## ⚡ 自動化戦略

### 自動マージ
- パッチ更新（セキュリティ・バグ修正）
- Go依存関係（テスト通過後）
- AWS SDKパッチ更新

### 手動レビュー
- メジャーバージョン（破壊的変更）
- Terraform メジャー（インフラ影響）
- Hugo更新（サイト生成影響）

### スケジューリング
- **実行タイミング**: 平日朝8時前（Asia/Tokyo）
- **レビューフロー**: 朝にPR作成 → 営業時間中レビュー → 条件満たすもの自動マージ

## 使用方法

### 基本設定

各リポジトリで `.github/renovate.json` を作成：

```json
{
  "extends": [
    "github>iloc-inc/renovate-config"
  ]
}
```

### 技術別設定

#### Terraformプロジェクト
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:terraform"
  ]
}
```

#### Goプロジェクト
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:go"
  ]
}
```

#### Hugoプロジェクト
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:hugo"
  ]
}
```

### 組み合わせ使用

```json
{
  "extends": [
    "github>iloc-inc/renovate-config:default",
    "github>iloc-inc/renovate-config:terraform"
  ]
}
```

## 技術別の適用対象

| 設定 | 対象プロジェクト | 主な機能 |
|------|-----------------|---------|
| `default` | 全プロジェクト | タイムゾーン、スケジュール、サプライチェーン対策 |
| `go` | martify, edinet-feeder, go-common | .go-version、go.mod、golangci-lint |
| `terraform` | infra, security-infra | AWS provider、バージョン制約 |
| `hugo` | company-site | Netlify/GitHub Actions版本管理 |

## 🔧 プロジェクト固有のカスタマイズ

共有設定を拡張してプロジェクト固有のルールを追加：

```json
{
  "extends": [
    "github>iloc-inc/renovate-config",
    "github>iloc-inc/renovate-config:go"
  ],
  "packageRules": [
    {
      "description": "このプロジェクト固有: 特定パッケージは手動レビュー",
      "matchPackageNames": ["github.com/critical/package"],
      "automerge": false
    }
  ]
}
```

## 🚨 トラブルシューティング

### PR未作成

| 原因 | 確認方法 | 対処 |
|------|---------|------|
| スケジュール設定 | Renovate Dashboardでログ確認 | タイムゾーン・スケジュール確認 |
| 権限不足 | GitHub Settings → Integrations → Renovate | contents/pull_requests write権限付与 |
| Bot無効化 | `.renovaterc.json`の`enabled`確認 | `enabled: true`に設定 |
| 依存関係ファイル未検出 | `go.mod`、`*.tf`等の存在確認 | ファイルが無いとRenovateは動作しない |

### マージ失敗

| 原因 | 確認方法 | 対処 |
|------|---------|------|
| テスト失敗 | `gh pr view <PR番号> --json statusCheckRollup` | ローカルで再現・修正 |
| 依存関係競合 | `go mod tidy` / `terraform init` | 依存関係を整理 |
| ブランチ保護ルール | `gh api repos/{owner}/{repo}/branches/main/protection` | Renovate Botが必須チェックをパスできるか確認 |

### 設定未適用

| 原因 | 確認方法 | 対処 |
|------|---------|------|
| JSON構文エラー | `jq . .renovaterc.json` | JSONLintでバリデーション |
| 継承パス誤り | `github>`を使っているか | `github>iloc-inc/renovate-config`（`:`ではなく`>`） |
| 設定検証 | `npx renovate-config-validator .renovaterc.json` | 公式ツールでバリデーション |

### よくある質問

**PRが大量に作成されて困る** → `"prConcurrentLimit": 5` で制限

**特定の依存関係を更新したくない** → `packageRules`で`"enabled": false`

**自動マージを無効化したい** → `"automerge": false`をルートに追加

## 🆕 新言語サポートの追加方法

### 追加手順

1. `renovate-config`リポジトリに`<言語>.json`を作成
2. PRで動作確認
3. 対象プロジェクトの`.renovaterc.json`で`extends`に追加

### 設定のベストプラクティス

1. **パッチ更新は自動マージ**: セキュリティ・バグ修正
2. **マイナー更新は慎重に**: devDependenciesは自動、本番dependenciesは手動
3. **メジャー更新は手動レビュー**: 破壊的変更の可能性
4. **テスト通過を必須条件**: `requiredStatusChecks`で指定

### Python対応の例

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Python依存関係管理（pip, poetry, pipenv）",
  "packageRules": [
    {
      "description": "Pythonパッチ更新の自動マージ",
      "matchManagers": ["pip_requirements", "poetry", "pipenv"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "requiredStatusChecks": ["test"]
    },
    {
      "description": "Pythonメジャー更新は手動レビュー",
      "matchManagers": ["pip_requirements", "poetry", "pipenv"],
      "matchUpdateTypes": ["major"],
      "automerge": false
    }
  ]
}
```

## 運用

### モニタリング
- Renovate Dashboard（PRステータス）
- 自動マージ成功率
- セキュリティアラート対応時間
- 未マージPRの滞留状況

### 将来計画
- 新言語サポート（Python、Node.js）の実装
- より精密な自動マージ条件（依存関係グラフ分析）
- 依存関係影響分析機能（破壊的変更の事前検出）
