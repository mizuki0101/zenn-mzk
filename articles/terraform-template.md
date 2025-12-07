---
title: "よく利用するTerraformのファイル構造"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform","aws"]
published: true
---

# はじめに

TerraformでAWS環境を作成する際のファイルは毎回最初は似た内容で作成するので、  
備忘録も兼ねてまとめ記事を作成しました。  
Terraformの公式スタイルガイドにならった構成にしているので初めてTerraformでAWS環境を構築する方の参考になれば幸いです。  
内容は随時追記予定です。  

# ファイル

## terraform.tf
Terraformのバージョンとプロバイダーに利用するAWSのバージョンを指定

Terraformバージョン確認  
https://registry.terraform.io/providers/hashicorp/aws/latest/docs

AWS プロバイダーバージョン確認  
https://registry.terraform.io/providers/hashicorp/aws/latest

バージョン指定にはマイナーバージョンは調整できる`~>`を基本的には使用しています。  
バージョンの指定方法  
https://developer.hashicorp.com/terraform/language/expressions/version-constraints

```
terraform {
  required_version = "~> 1.14.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.23.0"
    }
  }
}
```

## providers.tf
使用するプロバイダーであるAWSの設定内容の記載。  
デフォルトタグを設定することで、リソースに対して一括でタグを付与することができます。

```
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      Terraform   = "true"
    }
  }
}
```

## backend.tf

TerraformのstateをS3に保存するための設定。  
stateを保存するためのS3バケットを作成する方法はCLIやマネジメントコンソールから手動作成など色々ありますが、  
先にS3バケットをTerraformで作成し、あとからバックエンドで指定する方法を利用しています。  
詳細は[こちらの方の記事](https://kakakakakku.hatenablog.com/entry/2025/09/19/084816)を参考にしていただければと思います。

サンプルコードでは最終的な形を記載しています。

```
terraform {
  backend "s3" {
    bucket       = "tfstate-backend-[AWSアカウントID]"
    key          = "terraform.tfstate"
    encrypt      = true
    use_lockfile = true
    region       = "ap-northeast-1"
  }
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "tfstate-backend-[AWSアカウントID]"
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## terraform.tfvars
変数を管理するファイル。  
terraformが構築管理している対象のインフラに関する状態を保存。

```
project_name = "test"
environment  = "dev"
aws_region   = "ap-northeast-1"
```

## variables.tf
変数を定義するファイル。  
main.tfに合わせて変数を定義しています。

```
variable "project_name" {
  description = "Project name"
  type        = string
  default     = "test"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-1"
}

```

# 参考サイト
- https://developer.hashicorp.com/terraform/language/style
- https://kakakakakku.hatenablog.com/entry/2025/09/19/084816
- https://future-architect.github.io/arch-guidelines/documents/forTerraform/terraform_guidelines.html
