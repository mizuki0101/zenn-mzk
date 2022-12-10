---
title: "Dockerコマンド備忘録"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker"]
published: true
---
# 基本コマンド
docker images
docker login
docker pull hello-world
docker images

# コンテナ作成
docker run hello-world

# 起動しているコンテナ確認
docker ps 
docker ps -a

# ubuntuをbashで動かす
docker run -it ubuntu bash

# docker再起動
docker restart ID

# コンテナを動かす
# execコマンドはstatusがexited状態だとエラーになるため、restartをかけてupにする
docker exec -it 951a0d1b9d9e bash

# UP状態のコンテナに入る
docker attach 951a0d1b9d9e

# dockerコンテナに名前をつける
docker run --name sample_container ubuntu

# --rmでexit時にコンテナを削除する。
docker run --rm hello-world

# inspect コンテナの詳細な情報を表示
docker inspect <imageID>

# ファイルシステム共有
-v <host/path>:<contaner/path>

# アクセス権限
-u $(id -u):$(id -g)

# ポートを繋げる
-p <host_port>:<contaner_port>

# リソースの上限
--cpus<#ofCPUs> --memory <byte 2gなど> 

# imageをtarファイルに変換
docker save 8505f7acd7e6 > testimage.tar

# tarファイルからimageの読み込み
$ docker load < myimage.tar
