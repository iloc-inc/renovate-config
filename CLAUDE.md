# Renovate Config - Claude AI 開発ガイド

全ilocプロジェクトの統一依存関係更新管理システムのAI支援開発ルールとベストプラクティス

## 🎯 プロジェクト概要

共有Renovate設定リポジトリ - 全プロジェクトの依存関係更新ポリシーを一元管理

## 🤖 Claude AI設定（全プロジェクト共通）

### 日本語使用ルール

**基本方針**: 全て日本語で記述（技術用語は英語のまま使用可）

#### 📝 ドキュメント
- **コードコメント**: 日本語で記述（JSON設定の説明等）
- **README**: 日本語で記述（使い方・例・ベストプラクティス等）

#### 💬 コミュニケーション
- **説明・解説**: 日本語で記述（設定説明・エラー解説・調査結果等）
- **コミットメッセージ**: 日本語で記述（例: `feat: Go依存関係の自動マージ条件を追加`）
  - Conventional Commitsのプレフィックスは英語: `feat:`, `fix:`, `docs:` 等
- **PR（Pull Request）**: 日本語で記述（タイトル・説明・要約等）

#### 🔤 技術用語
英語のまま使用OK: Renovate, JSON, Go, Terraform, Hugo, AWS SDK 等

### Git/PR作成ルール

**基本方針**: mainブランチへの直接プッシュは禁止、必ずPR作成まで

#### 📌 必須フロー
1. **ブランチ作成**: 作業内容を示すブランチ名（例: `feat/xxx`, `fix/xxx`, `docs/xxx`）
2. **設定変更**: JSON設定ファイルを編集
3. **PR作成前チェック**: 以下を実行して、正常動作を確認
   - JSON構文チェック（`jq . <ファイル名>`）
   - Renovate設定バリデーション（可能なら）
   - テストプロジェクトでの動作確認
4. **コミット**: 変更をステージング・コミット
5. **プッシュ**: リモートブランチにプッシュ（`-u` オプション使用）
6. **PR作成**: `gh pr create` でPull Requestを作成
7. **レビュー待機**: ユーザーがレビュー・マージ

#### ❌ 禁止事項
- **mainブランチへの直接プッシュ**: 例外なく禁止
- **レビュー無しマージ**: AIが勝手にマージしない
- **JSON構文チェックのスキップ**: 必ずバリデーション実行

#### ✅ PR作成時のポイント
- **タイトル**: 日本語で簡潔に（例: 「Terraform自動マージ条件を厳格化」）
- **本文**: 変更概要、影響範囲、設定の意図を日本語で記載
- **ラベル/Assignee**: 必要に応じて設定（ユーザーの指示があれば）

#### 🔧 例外ケース
以下の場合のみ、ユーザーに確認してから直接プッシュ可能：
- 緊急のセキュリティ修正
- ドキュメントの軽微な修正（typo等）

**重要**: 迷った場合は常にPR作成を選択すること

## 🏗️ アーキテクチャ

### 設定ファイル構成

```
renovate-config/
├── default.json           # 全プロジェクト共通設定
├── go.json               # Go専用設定
├── hugo.json             # Hugo専用設定
├── terraform.json        # Terraform専用設定
└── README.md             # 使用方法ドキュメント
```

### 設計原則

- **DRY原則**: 共通設定をdefault.jsonに集約、技術固有設定を分離
- **一元管理**: 全プロジェクトで統一されたポリシー適用
- **タイムゾーン最適化**: Asia/Tokyo営業時間に最適化されたスケジュール
- **安全性**: 重要度に応じた自動/手動マージ戦略

## 💻 Renovate設定開発ルール

### JSON設定規約

#### ファイル構造
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "設定の目的を日本語で記載",
  "extends": ["継承する設定"],
  "packageRules": [
    {
      "description": "ルールの説明を日本語で記載",
      "matchManagers": ["対象マネージャー"],
      "matchUpdateTypes": ["更新タイプ"],
      "automerge": true/false
    }
  ]
}
```

#### コメント規約
- JSONはコメント不可のため、`description`フィールドに日本語で説明を記載
- 各ルールには必ず`description`を付与
- 複雑なロジックはREADME.mdで詳述

#### バリデーション
```bash
# JSON構文チェック
jq . default.json

# Renovate設定検証（renovate-config-validator）
renovate-config-validator default.json
```

### 自動化戦略

#### 自動マージ条件
- **パッチ更新**: セキュリティ・バグ修正（`patch`）
- **Go依存関係**: テスト通過後の全更新
- **AWS SDKパッチ**: AWS SDK v2のパッチ更新

```json
{
  "packageRules": [
    {
      "description": "Go依存関係の自動マージ（テスト通過後）",
      "matchManagers": ["gomod"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["test"]
    }
  ]
}
```

#### 手動レビュー条件
- **メジャーバージョン**: 破壊的変更の可能性
- **Terraformメジャー**: インフラへの影響大
- **Hugo更新**: サイト生成ロジック変更

```json
{
  "packageRules": [
    {
      "description": "Terraformメジャーバージョンは手動レビュー",
      "matchManagers": ["terraform"],
      "matchUpdateTypes": ["major"],
      "automerge": false
    }
  ]
}
```

### スケジューリング戦略

#### タイムゾーン設定
- **タイムゾーン**: `Asia/Tokyo`
- **実行時間**: 平日朝8時前（`before 8am on Monday through Friday`）
- **意図**: 営業時間開始前にPR作成、日中にレビュー

```json
{
  "timezone": "Asia/Tokyo",
  "schedule": ["before 8am on Monday through Friday"]
}
```

#### スケジュール調整
- **緊急度高**: `at any time`（セキュリティアップデート）
- **緊急度低**: `monthly`（マイナー更新）

## 🔧 技術固有設定

### Go設定（go.json）

#### 対象ファイル
- `.go-version`: Go本体バージョン
- `go.mod`: Go依存関係
- `.golangci.yml`: Linterバージョン

#### 特徴
- 全更新を自動マージ（テスト通過後）
- 高頻度更新に対応

### Hugo設定（hugo.json）

#### 対象環境
- Netlify: `netlify.toml`の`HUGO_VERSION`
- GitHub Actions: `.github/workflows/`の`hugo-version`

#### 特徴
- メジャー更新は手動レビュー
- サイト生成への影響を慎重に評価

### Terraform設定（terraform.json）

#### 対象リソース
- Terraform本体バージョン
- AWSプロバイダー（`hashicorp/aws`）
- その他プロバイダー

#### 特徴
- パッチのみ自動マージ
- メジャー/マイナーは手動レビュー
- インフラ変更の影響を慎重に評価

## 🔒 セキュリティベストプラクティス

### 脆弱性対応
- **自動パッチ**: セキュリティアップデートは最優先
- **アラート連携**: GitHub Security Alertsと連携
- **迅速対応**: 検出後即座にPR作成

### 権限管理
- **最小権限**: Renovate Botに必要最小限の権限のみ付与
- **シークレット管理**: GitHub Secretsで認証情報管理
- **監査ログ**: 全更新履歴をGitコミットログで追跡

### テスト要件
- **CI/CD統合**: 全PRで自動テスト実行
- **マージ条件**: テスト通過を自動マージの必須条件に設定
- **ロールバック**: 問題発生時の即座のロールバック体制

## 🚀 運用ワークフロー

### 設定変更フロー

#### 1. ローカルでの変更
```bash
# JSON構文チェック
jq . default.json

# 設定バリデーション
renovate-config-validator default.json
```

#### 2. テストプロジェクトでの検証
- テスト用リポジトリで設定を適用
- Renovate Botの動作確認
- PR作成・自動マージ動作の検証

#### 3. 本番適用
- PR作成 → レビュー → マージ
- 全プロジェクトに自動反映

### トラブルシューティング

#### PR未作成
- **原因**: スケジュール設定、権限不足、Renovate Bot無効化
- **確認**: Renovate Dashboardログ、リポジトリ設定、Bot権限

#### マージ失敗
- **原因**: テスト失敗、依存関係競合、CI/CD障害
- **確認**: CI/CDログ、依存関係ツリー、マージ条件設定

#### 設定未適用
- **原因**: JSON構文エラー、継承パス誤り、設定競合
- **確認**: JSON構文、`extends`パス、packageRules優先順位

### モニタリング

#### 定期確認項目
- Renovate Dashboard（PRステータス）
- 自動マージ成功率
- セキュリティアラート対応時間
- 未マージPRの滞留状況

## 📚 参考ドキュメント

プロジェクト全体の設計・アーキテクチャは [`company-ai-strategy/`](../../company-ai-strategy/) を参照：

- [`renovate.md`](../../company-ai-strategy/renovate.md): 依存関係管理システム全体像

---

**🤖 AI システム専用**: この設定により、統一された依存関係更新管理を維持しながら効率的な開発が可能になります。
