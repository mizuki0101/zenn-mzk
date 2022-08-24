---
title: "loadavg上昇時に稼働しているプロセスの確認方法"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux","ShellScript","bash"]
published: false
---
# はじめに
サーバで定期的にロードアベレージが上昇することがありました。
その原因を探るため、ロードアベレージが上がった時のtopコマンドから稼働しているプロセス一覧を取得するスクリプトを作成しました。

# スクリプト全文
```
#!/bin/bash

#uptimeコマンドからLoadavgを取得
loadavg=`uptime | awk '{print $8}' | sed -e 's/,$//'`

#topコマンド実行結果を保存するファイルを変数に代入
top_result=top_result-`date "+%Y%m%d"`.txt

#決めたLoadavgより上昇した場合にtopコマンドを実行
if [ `echo "$loadavg >= 4.00" | bc` -eq 1 ]; then
    for i in `seq 3`
    do
        echo -e "=======================================\n`date "+%Y%m%d%H%M"`" >> $top_result
        top -b -d 15 | head -15 >> $top_result
    done
fi
```