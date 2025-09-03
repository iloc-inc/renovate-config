# renovate-config

共有 Renovate 設定 - 依存関係自動更新 (Go/Terraform/Hugo)

## 概要

iloc全体での統一された依存関係更新管理を提供する共有Renovate設定システムです。

## 🚀 開発コマンド

設定ファイル専用リポジトリのため、特別な開発コマンドはありません。

設定変更後は他リポジトリでの動作確認を行います。

## 設定ファイル

- **`default.json`**: 全プロジェクト共通設定（Asia/Tokyoタイムゾーン、平日朝8時前スケジュール）
- **`go.json`**: Go専用設定（.go-version、go.mod、golangci-lint管理）
- **`hugo.json`**: Hugo専用設定（Netlify/GitHub Actions版本管理）
- **`terraform.json`**: Terraform専用設定（AWS provider、バージョン制約管理）

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

## 主な機能

- **自動スケジューリング**: 平日朝8時前（Asia/Tokyo）にPR作成
- **自動マージ**: パッチ更新・Go依存関係・AWS SDK更新
- **手動レビュー**: メジャーバージョン・Terraform・Hugo更新
- **グループ化**: 関連パッケージをまとめて更新
