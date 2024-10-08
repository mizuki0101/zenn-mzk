---
title: "keyball44からkeyball39へ移行した設定について"
emoji: "🐤"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["keyboard","keyball","keyball39"]
published: true
---

# はじめに
今までHHKBを愛用していましたが、キーボードとトラックボールスタイルのため分割キーボード+トラックボール一体型のkeyballに興味を持ちました。
技術の授業以外ではんだこてを使用したことはなかったので、自分で作成できるか不安でしたがなんとか完成まで進みました。
keyballは2台所有しており、keyball44とkeyball39があります。

# なぜ2台あるのか
HHKBから乗り換えたこともあり、最初keyball39のキー数では不安があったためkeyball44を使用していました。
メインＰＣはWindowsのためAlt+TabやCtrl+Tabなどの小指で操作するショートカットを多用しており、keyball39で足りるイメージが当初全く持てませんでした。
しかし、レイヤーでのキー割り当てにはAキー長押し時はCtrlにするなどの機能が購入後設定を詰めていく中で分かりました。
これならKeyball39の少ないキーでも小指キーにあたるところにCtrlなどのキーを割り当てればいけるイメージが持てたため移行しました。

# keyball39を使用する上でのキーマップ
keyball39はデフォルトのままでは使用できなので、自分で設定を詰めていく必要があります。
設定で意識した点などをまとめてみました。

## CtrlやAltキーなど
CtrlやAltキーなどのキーは長押し時に設定しています。Ctl+Aキーなどの左手で同時押しする必要があるものもあるので左右にCtrlはおいています。
Altキーも左下だけだと打ちずらいので右手のJキー長押しでも押せるようにしています。
こうすることでAlt+TabなどのAltを組み合わせる場合の打ちやすさがHHKBよりも上がりました。

## マウスキーについて
マウスキーをどこに置くかは人によって変わる部分かと思います。
現在マウスキーは右手の通常キーだと"<"と">"にあたるキーに配置しています。
初めは違和感ありますが片手だけで操作が可能になります。すぐに慣れると思います。
また、keyballではデフォルトでLayer3がスクロールモードに移行するキーになっているため、Mを長押しすることでLayer3へ移行できるようにしています。
これも片手で操作をするためです。
普段両手で操作するときは押しやすいようにFキー長押しでLayer3へ移行するようにしています。
### 補足AML(AutoMouseLayer)
AMLというトラックボールを動かしているときだけ専用のマウスレイヤーへ移行するモードが公式のQMKでも用意されていますが、こちらは私の使用感には合いませんでした。
少しふれたときにLayerが切り替わってしまったり、クリックだけしたいのにトラックボールを動かす必要があったりと誤検知が多かったためです。
もっと設定を詰めればスムーズにできる可能性はありましたが、そこまでいきませんでした。

## 矢印キーやHOMEキーなど文字操作関連のキー
文字を効率よく打つ上で矢印キーやHOMEキーなどの打ちやすさは大事だと思います。
それらのキーはLayer3にまとめており、ホームポジションのFキーを長押しすることで右手ですべて打てるようにしています。
矢印キーはVimに合わせて横並びにしています。ほかのキーは合わせる形で配置しています。
追加のポイントとしてLayer3のときにDキー長押しでShiftキーにしています。
こうすることで、矢印キーなどと組み合わせて範囲選択をし易くしています。
今まで打ちずらかったENDキーで文の末尾に移動してから、Shift+HOMEで範囲選択などのキー操作が素早く打てます。
はじめは後述するマクロで設定していましたが必要なくなりました。

## 記号キー
@#$":などの記号キーはLayer1でまとめています。
打つごとにLayerを切り替えるのは少し面倒ですが慣れて配置を覚えるとそれほど課題には感じなくなりました。
一点問題がありShiftと組み合わせて入力する記号キーには長押しキーが割り当てれません。（ファームウェアからもできなかったのでおそらく方法はない）
Ctlr+/など打ちたいときは、小指でCtrlキーを押しながらLayer1に移動して/を打つなどの必要がありました。
あるとき先にLayer0でAキー長押しでCtlrキーを押すと、Layer移動後も継続できることが分かったので、今は先に押してから移動するようにしています。

## 言語切替
言語切替はAlt+`キーで切替れるようにマクロを設定しています。
左右の親指に英語と日本語で切り替える方法もありますが、無駄にキーを使用している気がして一つのキーで切り替えれるようにしました。

## その他
数字キーはテンキーと同じレイアウトでLayer2にまとめています。
F1キーなども同様に並べています。

# ファームウェア
## ビルド方法
上記まではRemapだけで設定ができます。AML(AutoMouseLayer)やコンボキーなどの細かい調整を行いたい場合はファームウェアから設定する必要があります。
QMKに対応しているのでQMKのドキュメントを参照しながら設定をしました。
設定をいじった後にProMicroに書き込めるようにビルドする必要がありますが、QMKだと少々複雑でここで挫折してしまう方も多いかと思います。
はじめはローカルでやろうとしたのですが、keyballが対応しているQMKのバージョンが古いもの？だったりなどバージョン指定も必要ですんなりできませんでした。
悩んでいたところkeyball公式リポジトリでGitHub Actionsを使ってオンライン上でビルドが可能になっていました。
[ヨーキースさんの公式リポジトリ](https://github.com/Yowkees/keyball)からフォークを行い、連携したリポジトリでプッシュすることで自動的にビルドが走りhexファイル形式で取得することができました。
この方法だとGitHubで自分のキーマップなども管理ができ、ビルドも自動で行えるのでとても便利でした。

## コンボー
主にファームウェアで設定しているのはコンボキーのみです。
vimに合わせてj+kだとESCになるように設定しています。vimと同じ環境になるのでとても気に入っています。

# まとめ
ながながと書きましたがkeyball44とkeyball39で悩んでいる方は、はじめからkeyball39でもよいかと思います。
記述したとおり設定でカバーできる項目がたくさんあるためご自身にあった設定を見つけれると思います。
最初は作業効率爆落ちしますが、慣れていけば巻き返せるはずです。（私は一か月くらいかかりました。。。）

これから行いたいのは、今界隈で盛り上がっている無線化にも手を出せたらいいなと思っています。
ご覧いただきありがとうございました！