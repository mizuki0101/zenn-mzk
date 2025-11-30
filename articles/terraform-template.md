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
内容は追記予定です。

# ファイル

## terraform.tf
Terraformのバージョンとプロバイダーに利用するＡＷＳのバージョンを指定

Terraformバージョン確認
https://registry.terraform.io/providers/hashicorp/aws/latest/docs

AWS プロバイダーバージョン確認
https://registry.terraform.io/providers/hashicorp/aws/latest

バージョンしていにはマイナーバージョンは調整できる`~>`を基本的には使用します。
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

## providers .tf
使用するプロバイダーであるAWSの設定内容の記載

```
provider "aws" {
  region = "ap-northeaset-1"
}
```


ここからは追記予定。。。