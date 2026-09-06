# renovate-config

共有 Renovate 設定 - 依存関係自動更新 (Go/Terraform/Hugo)

## 概要

iloc全体での統一された依存関係更新管理を提供する共有Renovate設定システムです。

### 目的とメリット

- **統一ガバナンス**: 全プロジェクトの依存関係更新ポリシーを一元管理
- **技術固有最適化**: Go、Terraform、Hugo向けの専用更新戦略
- **レビュー必須**: 全更新をPR経由の手動マージにし、インフラへの無レビュー適用を防ぐ
- **更新スケジュール**: Asia/Tokyo で毎月1日にまとめて実行（脆弱性のみ即時）
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

> [!IMPORTANT]
> **`github-releases` datasource は対象外**にしています（`"matchDatasources": ["github-releases"], "pinDigests": false`）。
>
> `.terraform-version` や tflint の ruleset のように「GitHub Release のバージョン番号」を参照する依存は、実タグが `v1.16.1` なのに Renovate が `1.16.1` でタグを探すため digest 解決に必ず失敗し、**更新候補そのものが破棄されます**。エラーにはならず DEBUG ログに `Could not determine new digest for update.` が出るだけで、Dependency Dashboard にも何も表示されません。
>
> 実際 `.terraform-version` は `pinDigests` 導入（2026-04-17）から165日間 1.14.8 に据え置かれ、誰も気づけませんでした。
>
> GitHub Actions は datasource が `github-tags` で `helpers:pinGitHubActionDigests` が担当しているため、digest 固定はそのまま有効です。

### GitHub Actions Digest Pinning（`helpers:pinGitHubActionDigests`）

GitHub Actionsのアクション参照に特化したdigest pinning。全ワークフローで自動適用。

### 脆弱性アラート自動対応（`vulnerabilityAlerts`）

GitHubのセキュリティアドバイザリと連携し、脆弱性が検出された依存関係に対して`security`ラベル付きPRを自動作成。スケジュールに関係なく即座に対応。

## 設定ファイル

- **`default.json`**: 全プロジェクト共通（Asia/Tokyoタイムゾーン、毎月1日スケジュール、サプライチェーン攻撃対策）
- **`go.json`**: Go専用（.go-version、go.mod、golangci-lint管理）
- **`hugo.json`**: Hugo専用（Netlify/GitHub Actions版本管理）
- **`terraform.json`**: Terraform専用（provider、core/toolchainバージョン、tflint、tflintプラグイン）

## ⚡ 自動化戦略

### 自動マージ

**現状、全プリセットで無効**（`default.json` の `"automerge": false`）。上書きしているプリセットはありません。

特に `infra` / `security-infra` は **mainへのマージで `terraform apply -auto-approve` が走る**ため、依存更新を自動マージするとレビュー無しでAWSに適用されます。Terraform関連は今後も自動マージしない方針です。

### 手動レビュー（＝すべての更新）
- Terraform provider / modules / core / toolchain（`.terraform-version`）
- tflint 本体・rulesetプラグイン
- GitHub Actions digest
- Go / Hugo の各依存

### スケジューリング
- **実行タイミング**: 毎月1日 3時前（Asia/Tokyo）
- **例外**: 脆弱性アラート（`vulnerabilityAlerts`）はスケジュール無視で即時PR作成

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
| digest解決の失敗 | dry-runログに `Could not determine new digest for update.` が出ていないか | 該当 datasource を `pinDigests: false` の対象に加える（[実例](#-サプライチェーン攻撃対策)） |
| hostRulesのシークレット未設定 | dry-runが `init` フェーズで `Unknown secrets name` により中断していないか | Mend側でシークレットを再設定。未設定だと extract も lookup も一切走らず、**PRが出ないだけの静かな停止**になる |

いずれも Dependency Dashboard には何も表示されず、CIも緑のままです。**「更新が来ないだけ」の障害は気づけない**ので、疑ったら必ず dry-run でログを見てください。

```bash
LOG_LEVEL=debug \
  RENOVATE_TOKEN="$(gh auth token)" \
  RENOVATE_SECRETS='{"GO_COMMON_TOKEN":"dummy"}' \
  npx --yes --package renovate renovate --dry-run=full --platform=github iloc-inc/infra
```

- リポジトリ名は**位置引数**（`--repositories=` は不正オプションで起動に失敗します）
- `RENOVATE_SECRETS` を渡さないと `hostRules` の `{{ secrets.* }}` を解決できず init で止まります
- 生成されるはずのブランチは `"branchName": "renovate/..."` を grep すると一覧できます

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

**更新PRが一向に出ない** → 「[PR未作成](#pr未作成)」を参照。ダッシュボードには何も出ないため dry-run でログを見る必要があります

## 🆕 新言語サポートの追加方法

### 追加手順

1. `renovate-config`リポジトリに`<言語>.json`を作成
2. PRで動作確認
3. 対象プロジェクトの`.renovaterc.json`で`extends`に追加

### 設定のベストプラクティス

1. **原則すべて手動マージ**: 本リポジトリの全プリセットで `automerge: false`。特に Terraform は main へのマージが本番 apply を意味する
2. **メジャー更新は特に慎重に**: 破壊的変更の可能性
3. **発火しないルールを置かない**: 追加時は `renovate-config-validator` に通し、可能なら `--dry-run=full` で生成ブランチが期待どおり変わるかまで確認する。設定エラーのルールは Renovate に読まれず、黙って無視されます
4. **datasource ごとの差異に注意**: `github-tags`（GitHub Actions）と `github-releases`（ツールのバージョン）は挙動が異なります
5. **`matchUpdateTypes` と `rangeStrategy` は併用不可**: `packageRules cannot combine both matchUpdateTypes and rangeStrategy` で設定エラーになります

### Python対応の例

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Python依存関係管理（pip, poetry, pipenv）",
  "packageRules": [
    {
      "description": "Python依存はまとめてPR。手動レビュー",
      "matchManagers": ["pip_requirements", "poetry", "pipenv"],
      "groupName": "Python dependencies",
      "labels": ["python"],
      "automerge": false
    },
    {
      "description": "メジャー更新は分離して差分を追いやすくする",
      "matchManagers": ["pip_requirements", "poetry", "pipenv"],
      "matchUpdateTypes": ["major"],
      "groupName": "Python dependencies (major)",
      "labels": ["python", "major"],
      "automerge": false
    }
  ]
}
```

## 運用

### モニタリング
- Dependency Dashboard（各リポジトリの Issue）— 検出済み依存と更新待ちの一覧
- 未マージPRの滞留状況
- セキュリティアラート対応時間
- **検出漏れ**: ダッシュボードの「Detected Dependencies」に想定した依存が並んでいるか。件数が0のmanagerは、設定が効いていないサインです

### 既知の制約

- **provider のパッチのみのリリースは PR にならない**: 制約が `~> 6.62` 形式でパッチ桁を持たないため、パッチリリースでは制約が変わらず差分が出ません。次の minor 更新で `.terraform.lock.hcl` ごと追随します。埋めるには `rangeStrategy` の変更が必要ですが、lockfile を持たないモジュールの制約が更新されなくなるため見送っています
- **Python / Node.js のプリセットは未整備**: 必要になった時点で「[新言語サポートの追加方法](#-新言語サポートの追加方法)」の手順で追加してください
