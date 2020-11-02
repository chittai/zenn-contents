---
title: "EC2のバックアップ運用について調べてみた"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: false
---

# 概要
AWSのEC2バックアップ運用について調べてみました。EC2上でアプリケーションを動かしている時に、どのようにバックアップを取ればいいのか、さらには運用をどのサービスの組み合わせで実施できるのか調べてみました。

# EC2のバックアップの種類
1. EBSスナップショット
2. AMI

# 停止して取るべき？
停止しなくても取得することはできますが、基本は静止点を定めるためにアプリケーションの停止やインスタンスの停止が推奨されています。

# 運用はどのようなサービスを使う？
1. インスタンス / アプリケーションの停止
2. バックアップ
3. インスタンス / アプリケーション起動


# AMIの自動化は可能？
コンソール画面からやるっぽいので自動は色々工夫が必要かも
1. AWS CLI
2. AWS CloudWatch Alarm
3. Lambda
4. SystemManager Automation(メンテンスウィンドウ、SSM Agentの導入が必須) →　世代管理どうする？
5. AWS Backupで可能

# EBSの自動化は？
1. AWS Backup
2. DLM
3. Lambda + Cloud Watch Event?Alarm?
4. SystemManager Automation
5. ssm auto でsnapshotを取得するときはAPIを使用している

# 世代管理
* DLMでもBackupでも可能
* backupをautoからたたけないか


# 整理
* バックアップは決まり
* サーバの停止・起動を同実施するか

Automationを使用したバックアップ方法についてまとめる


