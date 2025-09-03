# renovate-config

共有 Renovate 設定 - 依存関係自動更新 (Go/Terraform/Hugo)

## 🚀 開発コマンド

このリポジトリは設定ファイルのみのため、特別な開発コマンドはありません。

設定変更後は他リポジトリでの動作確認を行います。

## Available Configurations

- `default.json` - Base configuration for all projects
- `go.json` - Go/Golang specific settings
- `hugo.json` - Hugo static site generator settings
- `terraform.json` - Terraform project settings (AWS providers, modules, toolchain)

## Usage

### Default Configuration

In your repository, add `.github/renovate.json` with:

```json
{
  "extends": [
    "github>iloc-inc/renovate-config"
  ]
}
```

### Specialized Configurations

For specific project types, use the appropriate configuration:

#### Terraform Projects (including security-infra)
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:terraform"
  ]
}
```

#### Go Projects
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:go"
  ]
}
```

#### Hugo Projects
```json
{
  "extends": [
    "github>iloc-inc/renovate-config:hugo"
  ]
}
```
