# Renovate Config for iloc-inc

This repository contains the shared Renovate configuration used across iloc-inc projects.

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
