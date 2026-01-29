---
title: "CDKでAuroraのインスタンスサイズを安全に変更する - 宣言的なDB定義のすすめ"
emoji: "🔄"
type: "tech"
topics: ["aws", "cdk", "aurora", "rds", "typescript"]
published: false
publication_name: "genda_jp"
---

# はじめに

株式会社GENDAでSRE/インフラエンジニアをしている布田です。本記事では、Aurora MySQL のインスタンスサイズを CDK で安全に変更した事例を紹介します。

今回は `db.r7g.4xlarge` から `db.r7g.2xlarge` へのダウンサイジングでした。ダウンタイムを最小限に抑えながら安全に行うためにDBのインスタンスを宣言的にすると良いと思ったので、その内容をまとめました。

# やりたかったこと：B/Gデプロイ方式での移行

ダウンタイムを最小化するために、以下の手順で移行したいと考えました。

1. **新しいサイズでDBインスタンスを追加**（ダウンタイムなし）
2. **新しいインスタンスへフェイルオーバー**（20-30秒のダウンタイム）
3. **古いインスタンスを削除**（ダウンタイムなし）

この方式なら、フェイルオーバーの1回分（20-30秒）だけでサイズ変更が完了します。
ただ、元々のCDKコードの構造だと、この手順が実現できませんでした。

# 問題：count形式で実装していたためB/Gができない

元々のCDKコードはインスタンス数を `count` で指定する形式でした。

### prod.ts

```typescript
provisionedReaders: {
  count: 3, ★ここに定義
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4),
},
```

### database-construct.ts

```typescript
if (props.provisionedReaders) {
  for (let i = 0; i < props.provisionedReaders.count; i++) {
    readers.push(
      rds.ClusterInstance.provisioned(`provReader${i + 1}`, {
        instanceType: props.provisionedReaders.instanceType,
      })
    );
  }
}
```

この形式だと以下の点でB/Gデプロイができませんでした。

1. **個別インスタンスの制御が難しい**
すべてのインスタンスが同じ設定になるため、特定のインスタンスだけサイズを変えるといったことができない。

2. **新旧インスタンスを混在させられない**
「古いサイズのインスタンス」と「新しいサイズのインスタンス」を同時に存在させることができない。

ちなみに、この形式で `instanceType` だけを変更してデプロイすると、Writerのサイズ変更→フェイルオーバー→フェイルオーバー先でもサイズ変更...と複数回のフェイルオーバーが発生する可能性があり、ダウンタイムが予測しづらくなりました。

これらの問題を解決するために、CDKのコードをリファクタリングすることにしました。

# 解決策：宣言的な配列形式へのリファクタリング

各インスタンスを明示的に定義する形式に変更しました。

### prod.ts

```typescript
dbWriter: {
  id: 'writer',
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4),
},
dbReaders: [
  { id: 'reader1', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
  { id: 'reader2', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
  { id: 'reader3', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
],
```

### database-construct.ts

```typescript
const readers: rds.IClusterInstance[] = [];

for (const readerConfig of props.readers) {
  readers.push(
    rds.ClusterInstance.provisioned(readerConfig.id, {
      instanceType: readerConfig.instanceType,
    })
  );
}
```

## この形式のメリット

この形式にしたことで、B/Gデプロイができるようになりました。

- インスタンスごとに異なる設定が可能 → 新旧サイズを混在させられる
- 配列に追加/削除するだけで制御できる → 段階的な移行が可能

また、運用面でもメリットがありました。

- 設定ファイルを見れば現在の構成がわかる
- 変更のレビューがしやすい

# 実際の移行手順

リファクタリング後のコードを使って、実際に移行を行いました。

## Step 1: 新規インスタンスを追加

新しいサイズのインスタンスを配列に追加します。

```typescript
dbReaders: [
  // 既存のインスタンス（4xlarge）
  { id: 'reader1', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
  { id: 'reader2', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
  { id: 'reader3', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE4) },
  // 新規追加（2xlarge）
  { id: 'newReader1', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
  { id: 'newReader2', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
  { id: 'newReader3', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
  { id: 'newReader4', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
],
```

この時点で新旧合わせて7台のReaderが稼働します。

## Step 2: フェイルオーバーの実行

AWS CLIでフェイルオーバーを実行し、新しいインスタンス（`newReader1`）をWriterに昇格させます。

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier <クラスターID> \
  --target-db-instance-identifier <newReader1の物理ID>
```

ダウンタイムは20-30秒程度でした。

## Step 3: 古いインスタンスを削除

フェイルオーバー完了後、古いインスタンスを配列から削除します。今回はWriterも変わっているため、あたらしいインスタンスを指定しました。

```typescript
dbWriter: {
  id: 'newReader1',
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2),
},
dbReaders: [
  { id: 'newReader2', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
  { id: 'newReader3', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
  { id: 'newReader4', instanceType: ec2.InstanceType.of(ec2.InstanceClass.R7G, ec2.InstanceSize.XLARGE2) },
],
```

これで、安全にCDK上でのDBサイズ変更が完了しました。

# まとめ

今回、Auroraのインスタンスサイズを安全に変更するためにB/Gデプロイ方式を取りたかったのですが、元々のcount形式のコードだと実現できませんでした。そこで、各インスタンスを宣言的に定義する形式にリファクタリングしたところ、B/Gデプロイができるようになりました。

最初は `count` 形式の方がシンプルに見えるのですが、運用を考えると宣言的な配列形式の方が扱いやすい所が多いなと感じています。設定ファイルを見れば構成がわかりますし、変更のレビューもしやすくなります。同じような悩みを抱えてる方がいたら参考になればと思います！

