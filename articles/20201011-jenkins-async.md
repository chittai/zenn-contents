---
title: "Jenkinsで非同期処理を行う方法"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Jenkins"]
published: true
---

# 概要
Jenkinsで非同期処理を行う方法について記述します。今回調べたのは、メイン処理が実行されている裏でファイルのダウンロード処理を動かす方法です。

# 前提
* Jenkins2.0以降
* Pileline(Declarative)を使用

# 処理
1. Jenkinsジョブ実行
2. メイン処理が稼働
3. その裏でファイルダウンロード実
4. 終了を確認

## 記述方法
scriptブロック内でparallelを使用します。同時に実行されますが何回か実行してみた結果、稼働するときは記載された処理の順番で実行されるようです。

```
stages{
    stage('SAGRADA実行'){
        steps{
            script{
                parallel(
                    execute: { //メイン処理実行
                    },
                    file_download: { //ファイルのダウンロード実行
                    }
                )
            }
        }
    }
}
```

## Groovyの機能を使用した場合どうなるか
下記を参考に、JenkinsnのPilpelineスクリプトにGroovyによる非同期処理を組み込みました。

[Groovyでマルチスレッド](https://devs.nobushige.net/groovy/15/)

しかし、結果としてメイン処理は実行されるもののGroovyで書いた非同期処理は実行されず。理由としてはまだ調べきれてないですが今回はparallelで実装することにしました。

# 今後
気になるポイントとして
* 並列処理のプログラムの終了待ちなどできるのか(今回はグローバル変数を使用して並列している処理の間で状態管理を実施)

