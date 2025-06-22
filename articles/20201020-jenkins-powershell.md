---
title: "JenkinsのPipelineでPowershellを使用するときのポケットリファレンス"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Jenkins", "Powershell"]
published: true
---

# 概要
Jenkinsの実行ノードがWindowsであった場合、Powershellで処理をしたいことがあるかと思います。以前自分がハマったケースを整理して参照できるようにしたいと思います。

# 環境
* Jenkinsサーバ(2.0以降)
* 実行ノード(Windows)

# 処理まとめ

## Pipelineの処理でPowershellのコマンドを実行する場合
```
powershell "<powershell command>"
```

## Powershellコマンドの中で変数を展開する場合
`""`の中に`${}`を記載することで変数を使用することができる。
```
powershell "<powershell command> ${variable}"
```

## Powershellコマンドの中で "" を使用する(エスケープする)場合
エスケープ処理をした文字の前に`\`を記載。
```
powershell "<powershell command> \"${variable}\" "
```

## Powershellコマンドを複数行に渡って書く場合
`"""`で囲む。
```
powershell script: """
    <powershell command1>
    <powershell command2>
"""
```

## Powershellコマンドで実行した内容をPipelineスクリプトの変数で受け取る場合
```
def result = powershell returnStdout: true, script: "<powershell command>"
```

### 実際に変数に格納されるのは文字列のため、ファイル数など値の情報を取得する場合

```
result = result.toInteger()
```

### 実際に変数に格納されるのは文字列(改行あり)のため、改行情報を削除する場合
```
result = result.replace("\r\n","")
```

## なにかしらの値をファイルに出力する場合
`-Encoding default` を指定することでUTF8を指定できる。ファイルがなければ作成する。既存ファイルは上書きされる。
```
powershell 'Out-File -Encoding default -FilePath ' + file
```

### ファイルに追記する場合
```
powershlell `Out-File ${file} -Encoding UTF8 -Append`
```

## 指定したファイルの行数を取得する方法
`${file}`に対象となるファイルを指定
```
def lines = powershell returnStdout: true, script: "(Get-Content -Path ${file} | Measure-Object -Line).Lines"
```

## ファイルに記載されているファイル名を削除する
`${file}`の中にファイルの絶対パスが記載されていて、それを読み取り削除する場合
```
C:\file1.txt
```
```
powershell script: "Get-Content ${file} | Remove-Item"
```

### ファイルの行数を複数指定する削除する場合
複数要素ある場合、範囲選択する。`${i}`から`${j}`までのファイルを選択して一つずつパイプの次の処理に渡すことができる。0インデックス。
```
C:\file1.txt
C:\file2.txt
C:\file3.txt
```
```
powershell script: "Get-Content ${file}[${i}..${j}] | Remove-Item"
```

## フォルダにあるファイル数を取得する
取得後、`toInteger()`が必要。
```
def str_file_num = powershell returnStdout: true, script: "(Get-ChildItem ${folder_path} | Measure-Object).Count"
```

## AWS CLIコマンドを使用する場合
AWS CLIを実行ノードにインストール済みの場合。
```
powershell "aws <command>"

e.g. powershell "aws s3 ls"
```

## S3上にあるフォルダの先頭ファイルの拡張子を取り除いた場合
これは特殊で、`AWSのS3`上のファイル情報を編集し、ls実行した結果先頭に来るファイルのファイル名から拡張子を除いた情報を取得する。`${extension}`には拡張子の情報を入れる(拡張子は指定する必要がある)。
```
def file_name = powershell returnStdout: truescript: "(aws s3 ls ${s3_folder_path})[1].Split().Trim[-1].Trim(\"${extension}\")"
```

## S3にあるファイルからとある拡張子を除いたファイルの数を取得する場合
指定したS3のディレクトリ内にあるファイルで`${extension}`拡張子以外のファイルの数を数える。
```
def lines_count = powershell returnStdout: true, script: "(aws s3 ls ${aws_s3_folder_path} | Select-String -Pattern \"${extension}\" -NotMatch | Measure-Object -Line).Lines"
```

## S3にあるファイルからとある拡張子を除いたファイルの一覧を取得する場合
指定したS3のディレクトリ内にあるファイルで`${extension}`拡張子以外のファイル名の一覧を取得する。
```
def lines = powershell returnStdout: true, script: """
(aws s3 ls ${aws_s3_folder_path} | Select-String -Pattern \"${g_extension}\"-NotMatch | ForEach-Object {\$(\$_ -split\" \")[-1]})[${i}] 
"""    
```

以上です。

