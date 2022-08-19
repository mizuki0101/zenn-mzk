---
title: "SESドメイン登録方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","ses"]
published: false
---
# はじめに
SESで事前に用意したドメインを登録してセットアップする機会があいrましたので、備忘録もかねて記事にしました。

# 用語
## SESとは
awsのEメール送信サービス

# セットアップ
- 利用予定のメールアドレスまたはドメインを登録します。
- 今回の例ではドメインの登録を行っていきます。
## Domain登録
- Create Identityをクリックします。
![](/images/SES/Snipaste_2022-08-19_22-51-01.png)
- identity typeをDomainを選択します。
- Domainに用意したドメイン名を入力します。
- 他二つのオプションは希望がなければオフにします。
![](/images/SES/ses2.png)
- DKIMの設定をします。
- Easy DKIMを選択します。
- RSA_2048_BITを選択します。
- Route53を使用している場合はオン。
- DKIM sinaturesもオン。（オンが推奨されています。）
![](/images/SES/ses3.png)
