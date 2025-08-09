---
title: "Amazon Q Developerのビルトインツール 'use_aws'の強みとは？"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS"]
published: false
---

# はじめに
Amazon Q Developerには `use_aws` というビルドインのツールがあります。公式ドキュメントの説明では、**AWS サービスとやり取りするための CLI AWS 呼び出しを行います。** と書かれているのですが、これって **execute_bash** でCLI Commandを叩いたり**AWS API MCP Sever**を使ってAPI叩くのと何か違うのか？と思ったので調査してみました。

https://docs.aws.amazon.com/ja_jp/amazonq/latest/qdeveloper-ug/command-line-chat-tools.html
https://awslabs.github.io/mcp/servers/aws-api-mcp-server/

# まとめ
use_aws には、AWS API 実行時のストリーミングレスポンス対応や、安全性を高めるエラーハンドリング・確認プロンプト、見やすい結果表示などが備わっています。
パラメータが間違っていた場合には、botocore の情報をもとに正しい形式を提案してくれるなど、AWS 利用を想定した便利な機能が多数組み込まれているのも強みかと思います。

実際に use_aws / execute_bash / AWS API MCP Server で同じ処理を試したところ、AWS環境を操作する場合、use_aws のほうが全体的に快適な体験を提供してくれると感じました。

基本的なことはどれでも実行できるため、「必須」というわけではありませんが、安全性や再現性を重視するなら use_aws を選ぶ理由になると思います。


# 確認方法
AWSの公式ブログにて Amazon Q Developer の開発は**Strands Agents SDK** が使われていると記載されているため、GitHubにある strands-agent/tools の [use_aws.py](https://github.com/strands-agents/tools/blob/main/src/strands_tools/use_aws.py) のコードを読み解くのが良いでしょう。以降では、Strands Agents のコードを参照しながら確認を進めます。

:::message
実際にAmazon Q Developer のuse_awsがStrandsのuse_awsと同等であるという記載はなかったのでこれはあくまでも間接的な情報から導いた推測であることは前提としてください。
:::

https://aws.amazon.com/jp/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/

> AWS の複数チームが既に本番環境で AI エージェントに Strands を使用しており、これには Amazon Q Developer、AWS Glue、VPC Reachability Analyzer などが含まれます。


## Strands Agent のコードから読み解くuse_awsとは？
https://github.com/strands-agents/tools/blob/main/src/strands_tools/use_aws.py

最初にKeyfeatureとして以下の3つが書かれています。それぞれについてどういうことか見ていきます。

```
Key Features:
1. Universal AWS Access:
2. Safety Features:
3. Response Handling:
```

### 1. Universal AWS Access
```
   • Access to all boto3-supported AWS services
   • Support for all service operations in snake_case format
   • Region-specific API calls
   • AWS profile support for credential management
```
まとめると「すべてのAWS操作を、統一的な書き方で実行できる」ということになります。これはAmazon Q Developerを利用している側には伝わりづらい部分かもしれません。開発する側からするとAWSサービスによらず統一的なインタフェースでAWSサービスにアクセスできるという点がメリットになると思います。

コメントの該当処理を見ていきます。Amazon Q Developer(CLI)から作業の指示を出すと、その指示の内容から`service_name`と`operation_name` など必要な情報をインプットにしてuse_awsを実行します。そのサービス、オペレーションが実行可能なものであるかを確認し、boto3クライアントを設定します。
```
# ここでAWSサービスが実行できるか確認 get_available_services　でチェック
available_services = get_available_services()
if service_name not in available_services:
    logger.debug(f"Invalid AWS service: {service_name}")
    return {
        "toolUseId": tool_use_id,
        "status": "error",
        "content": [
            {"text": f"Invalid AWS service: {service_name}\nAvailable services: {str(available_services)}"}
        ],
    }
```

```
# ここでオペレーションがあるかを get_available_operations でチェック 
available_operations = get_available_operations(service_name)
if operation_name not in available_operations:
    logger.debug(f"Invalid AWS operation: {operation_name}")
    return {
        "toolUseId": tool_use_id,
        "status": "error",
        "content": [
            {"text": f"Invalid AWS operation: {operation_name}, Available operations:\n{available_operations}\n"}
        ],
    }
```

そして、そのオペレーションを実行しています。
```
operation_method = getattr(client, operation_name) 
response = operation_method(**parameters)
```

以下は、boto3設定時にリージョンやProfileを指定しています。
```
def get_boto3_client(service_name: str, region_name: str, profile_name: Optional[str] = None):
    """Create an AWS boto3 client for the specified service and region."""
    session = boto3.Session(profile_name=profile_name)
    return session.client(service_name=service_name, region_name=region_name, config=config)
```

このように、AWSサービスへアクセスするための実装が準備されています。実際に、Amazon Q Developerで use_aws を実行すると以下のような表示が出ると思います。上記実装と照らし合わせると裏で何が行われているのかイメージが少しつくのではないでしょうか。

```
🛠️  Using tool: use_aws (trusted)
 ⋮ 
 ● Running aws cli command:

Service name: lambda
Operation name: get-function
Parameters: 
- FunctionName: "demo-error-function"
Profile name: xxx
Region: ap-northeast-1
Label: Lambda関数の詳細情報を取得
 ⋮ 
 ● Completed in 0.908s
```

### 2. Safety Features
```
   • Confirmation prompts for mutative operations (create, update, delete)
   • Parameter validation with helpful error messages
   • Automatic schema generation for invalid requests
   • Error handling with detailed feedback
```
まとめると「操作ミスや入力不備を未然に防ぎ、安全にAWSを操作できる仕組み」です。Write系の処理である作成・更新・削除などの操作には確認を挟み、入力エラー時にはわかりやすい説明と正しい形式の提案を自動で提示します。これは、Amazon Q Developerから利用している側としてもメリットとして感じ易い部分ではないでしょうか。

ここもコメントの該当処理を見ていきます。以下のように、Write系の処理を行うOperationの単語一覧があります。AWSの処理に見慣れている人にはおなじみの単語かと思います。この単語が入っているアクションを対象に確認を挟みます。
```
MUTATIVE_OPERATIONS = [
    "create", "put", "delete", "update", "terminate", "revoke", "disable",
    "deregister", "stop", "add", "modify", "remove", "attach", "detach",
    "start", "enable", "register", "set", "associate", "disassociate",
    "allocate", "release", "cancel", "reboot", "accept",
]
```
以下のように変更系のオペレーションのときは必ず確認するメッセージを表示します。そうすることで、勝手な変更などはせず安全な利用を実現しています。他の処理もコマンドの実行時にy/nを確認することはあると思いますがAWS環境のWrite系操作に絞ることでRead系のアクションでいちいち許可する手間が省けます。
```
# 変更系操作の検出と確認
is_mutative = any(op in operation_name.lower() for op in MUTATIVE_OPERATIONS)

if is_mutative and not STRANDS_BYPASS_TOOL_CONSENT:
    confirm = get_user_input(
        f"The operation '{operation_name}' is potentially mutative. Do you want to proceed? [y/*]"
    )
    if confirm.lower() != "y":
        return {"status": "error", "content": [{"text": "Operation canceled by user."}]}
```

以下はパラメータバリデーションチェックとパラメータを間違えたとき、その API が求める入力スキーマをその場で自動生成して返す仕組みです。要は「何をどう渡せばいいか」を即座に示してくれます。[generate_input_schema](https://github.com/strands-agents/tools/blob/main/src/strands_tools/utils/generate_schema_util.py) が実装です。operation_name からAPI名を見つけ、そこからServiceModel(boto3 が内部で使っている AWS サービスの API 定義オブジェクト)を取得し、最終的にJSON Schemaへ変換しています。

```
# 入力が不正な場合は、期待される入力スキーマを提示
try:
    response = operation(**parameters)
except (ValidationError, ParamValidationError) as val_ex:
    schema = generate_input_schema(service_name, operation_name)
    return {
        "toolUseId": tool_use_id,
        "status": "error",
        "content": [
            {"text": f"Validation error: {str(val_ex)}"},
            {"text": f"Expected input schema for {operation_name}:"},
            {"text": json.dumps(schema, indent=2)},
        ],
    }
```
最後の「詳細なエラーハンドリング」はここまでに説明したことも含め、単なる try/except ではなく入力の妥当性チェック（サービス・操作名）、API パラメータ検証（スキーマ提示）、安全確認のための対話、という 多段階のエラー処理 になっています。

### 3. Response Handling
```
   • JSON formatting of responses
   • Special handling for streaming responses
   • DateTime object conversion for JSON compatibility
   • Pretty printing of operation details
```
まとめると「AWS からの結果を見やすく、扱いやすい形で返す仕組み」です。レスポンスは JSON 形式に整えられ、ストリーミングレスポンスや日時データも適切に変換されます。さらに操作内容をわかりやすく整形表示し、結果の理解がスムーズになります。

ストリーミングレスポンスの特別処理は以下です。ストリーミングレスポンスとは boto3 / botocore が返す特殊なオブジェクトで、AWS からの大きなレスポンスデータを一気にメモリに載せず、少しずつ（ストリームとして）読み出すための仕組みです。これにより、大きなファイルも逐次処理や部分取得が可能となります。
```
def handle_streaming_body(response: Dict[str, Any]) -> Dict[str, Any]:
    """Process streaming body responses from AWS into regular Python objects."""
    for key, value in response.items():
        if isinstance(value, StreamingBody):
            content = value.read()
            try:
                response[key] = json.loads(content.decode("utf-8"))
            except json.JSONDecodeError:
                response[key] = content.decode("utf-8")
    return response
```

最後に、表示についてです。use_aws 実行時に指定された AWS サービス・オペレーション・リージョン・パラメータを、見やすく整形（Pretty Print）してコンソールに表示します。
```
# Create a panel for AWS Operation Details using Rich's native styling
details_table = Table(show_header=False, box=box.SIMPLE, pad_edge=False)
details_table.add_column("Property", style="cyan", justify="left", min_width=12)
details_table.add_column("Value", style="white", justify="left")

details_table.add_row("Service:", service_name)
details_table.add_row("Operation:", operation_name)
details_table.add_row("Region:", region)

if parameters:
    details_table.add_row("Parameters:", "")
    for key, value in parameters.items():
        details_table.add_row(f"  • {key}:", str(value))
else:
    details_table.add_row("Parameters:", "None")
console.print(Panel(details_table, title=f"[bold blue]🚀 {label}[/bold blue]", border_style="blue", expand=False))
logger.debug(
    "Invoking: service_name = %s, operation_name = %s, parameters = %s" % (service_name, operation_name, parameters)
)
```

# 結局、use_aws を使うのと、MCPサーバ/execute_bash を使うことの違いは何？
今回、Strands Agent を対象に調査しましたが、実際に操作してみた感覚としては、どちらも「AWS の情報を取得して処理する」という流れは同じです。ただし、体験にはいくつか違いがあります。見た目の違いもありますし、上で紹介したような機能の有無にも差があります。

明確な基準があるわけではありませんが、Strands Agent の中身を確認した限りでは、Amazon Q Developer を使って AWS 環境を操作することを想定した機能が組み込まれている use_aws の方が適していると感じます。体験そのものに大きな差を感じない場合でも、実装を知っておくことで「選ぶ理由」が作れるかなと思いました。

また、エラー処理や UI、変更時の確認などは、今後 UX の面で差が出てくる可能性があります。 use_aws には自動スキーマ生成やストリーミング対応といった、開発者・運用者の体験を向上させる仕組みがあり、AWS 利用に特化していると言えます。もちろん、これらの機能がなくても AI ツール側で「いい感じ」にやってくれる可能性はありますが、再現性のある形で実装されているという点は強みではないでしょうか。

これは定量的に測りづらい部分ではありますが、どの点が強みなのかを把握しておくことで、選択の根拠にできるのではないでしょうか。
