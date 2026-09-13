---
title: "【aws/nx-plugin】AIに、ユーザーストーリーだけ渡したら、どこまでアーキテクチャ設計構築できるのか？"
emoji: "🏗️"
type: "tech"
topics: ["aws", "claudecode", "nx", "cdk", "ai"]
published: false
---

## はじめに

2026年9月、AWS の Nx 向け Generator 集 `@aws/nx-plugin` が v1.0.0 としてリリースされました。README の冒頭には「Build full-stack AWS apps in minutes」とあり、[公式ドキュメント](https://awslabs.github.io/nx-plugin-for-aws/jp/)には、「プラグインにはMCPサーバーが付属しているため、AIアシスタントがプロジェクトをスキャフォールドして接続できます。」と書かれています。

部品がこれだけ揃っていて、AI がその部品を調べて呼べるなら、ユーザーストーリーだけを渡したら、どこまで自力でアーキテクチャを決めて実装まで持っていけるのか気になったので、検証してみたという趣旨の内容になります。

Claude Code に `@aws/nx-plugin` を使える状態で、難易度の異なる3つのユーザーストーリーを与えました。AWS 上で構築することと Generator を優先することは指定していますが、具体的な AWS サービス名やアーキテクチャは指定していません。その条件で、何を選び、何を選ばず、どこで人間の判断が必要になったかを記録しました。

## @aws/nx-plugin とは

[`@aws/nx-plugin`](https://github.com/awslabs/nx-plugin-for-aws)（Nx Plugin for AWS）は、awslabs が公開している Nx 用のプラグインで、Nx モノレポの中に「アプリケーションコード＋それをデプロイするための IaC」をセットで生成する Generator 群です。2026年9月7日に v1.0.0 がリリースされています。

検証時点（v1.0.0）で公開されている主な Generator は次のとおりです（`nx list @aws/nx-plugin` の出力から、hidden でないものを抜粋）。

| Generator | 生成されるもの |
| --- | --- |
| `ts#website` | React（Vite）の静的サイト。インフラは S3 + CloudFront + WAF |
| `ts#website#auth` | 既存サイトへの Cognito 認証追加（User Pool / Identity Pool / Hosted UI） |
| `ts#api` | tRPC または Smithy の API。インフラは API Gateway（REST or HTTP）+ Lambda。認証は IAM / Cognito / custom |
| `py#api` | FastAPI 版の API。同じく API Gateway + Lambda |
| `ts#dynamodb` / `py#dynamodb` | DynamoDB テーブル＋ ElectroDB（TS）によるエンティティ層 |
| `ts#rdb` / `py#rdb` | Aurora（PostgreSQL / MySQL）+ RDS Proxy + Prisma、マイグレーション用 Lambda |
| `ts#lambda-function` / `py#lambda-function` | 単体の Lambda 関数。40種類以上のイベントソーススキーマ（SQS / EventBridge / S3 / DynamoDB Stream など）を型付きで選べる |
| `ts#infra` | CDK アプリケーション本体（`packages/infra`）。Checkov によるテンプレート検査を build に組み込み |
| `terraform#project` | Terraform 版のインフラプロジェクト |
| `connection` | プロジェクト同士の接続（Website → API、API → DynamoDB など）。クライアント生成と IAM 権限付与をまとめて行う |
| `ts#agent` / `py#agent` / `ts#mcp-server` / `agentcore-*` | Strands Agent、MCP サーバ、Bedrock AgentCore 関連 |

 **SQS、EventBridge、SES、SNS、Step Functions、S3（単体）、ECS の Generator は存在しません。** これらが必要な場合は、既存の Generator が生成した構成に加えて、CDK などで補う必要があります。

また、冒頭でも触れた通り、AI エージェント向けの MCP サーバが同梱されており、`@aws/create-nx-workspace` でワークスペースを作ると、`nx-plugin-for-aws` という MCP サーバが自動登録され、次の7ツールが使えるようになります。

- `general-guidance`：Nx とプラグインの使い方
- `best-practices`：セキュリティやランタイム設定に関する横断的なガイド
- `list-generators`：Generator の一覧と、それぞれの実行コマンド・オプション
- `generator-guide`：特定 Generator の詳細ガイド。`options` を渡すと、その組み合わせに関係する部分だけに絞って返してくれる
- `create-workspace-command` / `add-to-existing-project` / `upgrade-workspace`：ワークスペースの作成・導入・更新

### 前提

**@aws/nx-plugin 自身はアーキテクチャを考えません。** 

- この要件に DynamoDB が適切か
- 非同期処理にキューを挟むべきか

といった判断は、Generator の外側にあります。プラグインの[公式ドキュメント（Security ページ）](https://awslabs.github.io/nx-plugin-for-aws/en/guides/security/)にも、次のように明記されています。

> The scope of the plugin is limited to its generators. The plugin has no knowledge of your application's business logic, data classification, threat model, or regulatory obligations, and cannot make decisions that depend on them.
>
> （訳）プラグインの守備範囲は Generator に限られます。プラグインはあなたのアプリケーションのビジネスロジック、データの機密区分、脅威モデル、規制上の義務については何も知らず、それらに依存する判断はできません。

今回の検証は、この「外側の判断」をコーディングエージェントがどこまで担えるかを見るものです。

## 検証ルール

Claude Code には AWS サービス名も、具体的なアーキテクチャも指定しません。渡したプロンプトの全文は次のとおりです（作業ディレクトリと MCP ラッパーの絶対パスだけ `<…>` に省略しています）。

```text
以下のユーザーストーリーを実現してください。
AWS上で動作するアプリケーションとして構築してください。
利用可能な場合は @aws/nx-plugin のGeneratorを優先して利用してください。
必要なアプリケーション構成とAWSリソースは、要件から判断してください。

ユーザーストーリー:
「（各ケースのユーザーストーリー）」

補足（検証運用上の制約。アーキテクチャ判断には影響させないでください）:
- 作業ディレクトリ: <ワークスペースのパス> 。このワークスペースは @aws/create-nx-workspace で作成済み（パッケージマネージャは pnpm、IaC は CDK）です。このディレクトリの外は変更しないでください。
- このワークスペースの .mcp.json には @aws/nx-plugin の MCP サーバ（nx-plugin-for-aws）が設定されていますが、あなたのセッションからは MCP ツールを直接呼べません。代わりに同じ MCP サーバを次のコマンドで呼べます（ワークスペース直下で実行）:
    node <…>/tools/nxmcp.mjs tools
    node <…>/tools/nxmcp.mjs <tool-name> '<json-args>'
  使うかどうか、どう使うかはあなたの判断に任せます。
- AWSアカウントへのデプロイ（cdk deploy / cdk bootstrap 等）は実行しないでください。ビルド・テスト・cdk synth が通るところまでで止めてください。
- 一部のシェルコマンドは権限ポリシーで拒否されることがあります。その場合は別の方法で進めてください。
- 作業の最後に、ワークスペース直下に次の2ファイルを日本語で作成してください。
  - DESIGN.md: 「解釈した要件（機能・非機能）」「選択したAWSサービスと選択理由」「利用した @aws/nx-plugin のGeneratorと、検討したが選ばなかったGenerator」「Generatorで賄えず手書きした部分」「人間のレビューが必要だと考える点」
  - WORKLOG.md: 実行した主要コマンド（Generatorの調査・実行コマンドを含む）を実行順に、実行結果（成功/失敗と要点）とともに記録
- 途中で判断に迷っても私には質問できません。自分で判断して進め、その判断と理由を DESIGN.md に残してください。
- 最終報告では、DESIGN.md の要約と、ビルド/テスト/cdk synth の結果（成功・失敗を正直に）を報告してください。
```

### 検証環境

| 項目 | 内容 |
| --- | --- |
| Claude Code | 2.1.268 |
| モデル | Claude Fable 5.1（`claude-fable-5-1`） |
| @aws/nx-plugin | 1.0.0（2026-09-07 リリース） |
| Nx | 23.2.0 |
| パッケージマネージャ / IaC | pnpm 10 / AWS CDK（aws-cdk-lib 2.268.0） |

ケースごとにワークスペースは完全に別で、Claude Code は他のケースの結果を知りません。

## Case 1：画像共有アプリ

### 与えたユーザーストーリー

```text
「ユーザーとして、写真をアップロードして一覧表示し、他のユーザーと共有したい。
ログインしたユーザーだけが写真を投稿できるようにしたい。」
```

このケースでは、認証・API・DB の基本構成に加えて、画像本体の保存先と配信経路（オブジェクトストレージ、CDN）、そしてアップロードの方式をどう判断するかを見ます。

### Claude Code が解釈した要件

DESIGN.md の要件表を、機能要件と非機能要件に分けて示します。「出どころ」は、ストーリーのどの語から導いたか、ストーリーにない補完かを表します。

**機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| F1 | サインアップ / サインインができる | 「ログインしたユーザー」 | Cognito User Pool、Hosted UI、自己サインアップ有効 |
| F2 | ログイン済みユーザーだけが投稿できる | ストーリー | API 全体を Cognito オーソライザーで保護。未認証は Lambda に到達させない |
| F3 | 写真をアップロードできる | ストーリー | 署名付き PUT URL で S3 に直接アップロードし、`confirmUpload` で登録 |
| F4 | 写真を一覧表示できる | ストーリー | DynamoDB の GSI（作成日時降順）を Query し、閲覧用の署名付き GET URL を付与。ページング対応 |
| F5 | 他のユーザーと共有できる | ストーリー（解釈） | ログイン済みユーザー全員に公開（All / Mine タブ）。宛先指定・リンク共有・公開範囲設定はスコープ外と明記 |
| F6 | 自分の写真を削除できる | 補完 | 投稿者本人のみ許可。S3 オブジェクトと DynamoDB レコードの両方を削除 |

**非機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| N1 | 画像は API を経由させない | 「写真」（サイズ） | API Gateway の 10 MB 制限、Lambda の同期呼び出しペイロード上限（6 MB）、Lambda の実行時間・コストを回避 |
| N2 | 画像のサイズ・種別を制限する | 補完 | 1 枚 10 MiB、JPEG / PNG / GIF / WebP / HEIC。クライアント・入力スキーマ・`HeadObject` でサイズと申告された `Content-Type` を検証 |
| N3 | 写真を非公開に保つ | 補完（共有範囲の解釈） | バケットはパブリックアクセス全ブロック。閲覧も署名付き URL（1 時間）経由のみ |
| N4 | 最小権限 | 補完 | Lambda を操作ごとに分離し、必要な S3 / DynamoDB 権限だけ付与 |
| N5 | 暗号化 | 補完（Generator 既定） | S3・DynamoDB は KMS CMK、SSL 強制、バージョニング |
| N6 | データ保護 | 補完 | 写真バケットとテーブルは `RETAIN` |

### 選択した AWS サービス

| 役割 | 選択されたもの | Generator 既定か手書きか | 選択理由（DESIGN.md より要約） |
| --- | --- | --- | --- |
| Frontend | S3 + CloudFront（React + shadcn/ui + Tailwind） | Generator（`ts#website` の既定） | `ts#website` の既定。「ギャラリーの自由なグリッドレイアウトには Tailwind + shadcn が扱いやすい」 |
| API | API Gateway REST + Lambda（tRPC、操作ごとに Lambda 分離） | Generator（`ts#api` の既定 `rest-lambda`） | `ts#api` の既定。フロントと型を共有できる tRPC。REST は WAF・アクセスログ付きで Cognito オーソライザーが使える |
| Authentication | Cognito User Pool + Identity Pool + Hosted UI | Generator（`ts#website#auth`、`ts#api --auth=cognito` を指定） | `ts#website#auth` が標準で構成。API Gateway と直結でき JWT 検証を自前実装しなくてよい |
| Database | DynamoDB（ElectroDB、GSI 2 本：全体フィード / 投稿者別） | Generator + 手書き（テーブルは `ts#dynamodb`、エンティティと GSI 設計は手書き） | 一覧と投稿者別一覧を GSI で Query できる。Aurora は過剰 |
| Storage | **S3（PhotoBucket、手書き）**：KMS CMK、バージョニング、非公開、CORS、ライフサイクル | **手書き CDK**（対応する Generator なし） | 画像はオブジェクトストレージが最適。署名付き URL で API を経由せず転送 |
| Upload 方式 | 署名付き PUT URL（5 分）→ `confirmUpload` で `HeadObject` 検証 | **手書き**（アプリコードと、CDK の CORS / 操作別 IAM） | API Gateway のペイロード上限（10 MB）や Lambda の同期呼び出し上限（6 MB）、Base64 化によるサイズ増加を避けるため S3 に直接転送。S3 イベント→Lambda 登録の非同期方式は「タイトル・説明を確定しづらい」ため見送り |
| CDN | CloudFront（SPA 配信のみ）。**画像配信には CloudFront を使わず S3 署名付き GET URL** | Generator（SPA 配信）。画像配信は手書き（署名付き URL） | 「CloudFront + 署名付き Cookie は効率的だがキーペア管理が必要。まずは S3 署名付き URL で実現し、規模拡大時の改善候補」 |
| Async | なし | ― | ― |
| Security | WAF（API / CloudFront）、KMS、Cognito **MFA 必須（Generator 既定のまま）** | Generator 既定（WAF、KMS、MFA 必須）。写真バケットの KMS は手書き | 「写真共有アプリとしては重い可能性があるが、セキュア既定を崩さずレビュー判断に委ねた」 |

### @aws/nx-plugin で利用した Generator

MCP ラッパーの呼び出し順は次のとおりです。

```text
tools
general-guidance
list-generators
best-practices {"pages":["workspace","typescript-project","security","runtime-config","local-development"]}
generator-guide ts#infra
generator-guide ts#dynamodb
generator-guide ts#website {"ux":"cloudscape","framework":"react"}
generator-guide ts#website#auth
generator-guide ts#api {"framework":"trpc","auth":"cognito","infra":"rest-lambda"}
generator-guide connection {"sourceType":"ts#trpc-api","targetType":"ts#dynamodb"}
generator-guide connection {"sourceType":"ts#react-website","targetType":"ts#trpc-api"}
```

`ts#website` のガイドは `ux=cloudscape` で引いたのに、実行時は `--ux=shadcn` を選んでいる点は特徴的でした。ガイドを読んだうえで、ギャラリー UI には shadcn の方が向くと判断を変えたことがDESIGN.mdに残されていました。

実行した Generator は 7 回でした。

```bash
pnpm nx g @aws/nx-plugin:ts#infra --name=infra
pnpm nx g @aws/nx-plugin:ts#website --name=website --framework=react --ux=shadcn --infra=cloudfront-s3
pnpm nx g @aws/nx-plugin:ts#website#auth --project=website --allowSignup=true
pnpm nx g @aws/nx-plugin:ts#api --name=photo-api --framework=trpc --auth=cognito \
  --infra=rest-lambda --integrationPattern=isolated
pnpm nx g @aws/nx-plugin:ts#dynamodb --name=photo-db --framework=electrodb --infra=dynamodb
pnpm nx g @aws/nx-plugin:connection --sourceProject=website --targetProject=photo-api
pnpm nx g @aws/nx-plugin:connection --sourceProject=photo-api --targetProject=photo-db
```

### 最終的な AWS アーキテクチャ

![Case 1 のアーキテクチャ。CloudFront + S3 の SPA、Cognito、API Gateway REST + Lambda ×4、DynamoDB、手書きの S3 PhotoBucket。画像は署名付き URL でブラウザから S3 に直接 PUT / GET する](/images/nx-plugin-user-story/case1-architecture.png)
*Case 1：画像共有アプリ。グレーの枠は Generator が生成した部分（枠の下に Generator 名）、オレンジの破線枠は Generator がなく手書きした部分、オレンジの矢印は要件に直結する経路です。*

個人的にはCloudFrontが入るかなと思っていたのですが、**画像配信には CloudFront を使っていません**。一覧 API が写真ごとに S3 の署名付き GET URL を返す方式です。

### AI が置いた前提（人間が確認すべきポイント）

ユーザーストーリーに書かれていない部分を、Claude Code は次のように決めていました。良し悪しではなく、要件次第で答えが変わる点です。

| AI の判断 | 前提にした要件 | 要件が違えば |
| --- | --- | --- |
| 「共有」= ログイン済みユーザー全員に公開 | 特定ユーザー宛ての共有や公開範囲設定は不要 | 宛先指定やリンク共有が要件なら、共有先を持つデータモデルと認可ロジックが必要 |
| 画像配信は S3 の署名付き GET URL（CloudFront を通さない） | 小規模で、CDN キャッシュ効率を重視しない | 規模が出て、URL を変えずに複数の画像へのアクセス制御を CloudFront 側へ寄せたいなら CloudFront + OAC + 署名付き Cookie |
| MFA 必須（Generator 既定のまま） | セキュリティ既定を優先し、体験は後で調整 | カジュアルな写真共有で離脱を避けるなら任意化。組織の方針で決める |
| 1 枚 10 MiB まで、JPEG / PNG / GIF / WebP / HEIC。この前提から署名付き URL で S3 に直接転送する構成を選択 | 一眼レフ級の写真も原寸で保存する | 上限を小さくする（クライアントで圧縮して数 MB 以下にするなど）なら API Gateway 経由でもよく、署名付き URL の仕組み自体が不要になる。RAW や動画なら逆に上限と種別を広げる。HEIC はブラウザで表示できない場合がある |
| 署名付き URL の期限は PUT 5 分・GET 1 時間 | 期限内に URL が共有されるリスクは許容 | 有効期間を短くするか、複数画像へのアクセス制御を CloudFront 側に寄せるなら署名付き Cookie も検討 |
| アップロード確定は同期の `confirmUpload`（S3 イベント方式は見送り） | タイトル・説明を確定してから登録したい | クライアントが落ちても登録したいなら S3 イベント → Lambda 方式 |
| 写真バケットとテーブルは `RETAIN` | 誤削除防止を優先 | 検証環境を頻繁に作り直すなら `DESTROY` |
| 全写真のフィードを単一パーティション（`feed=ALL`） | 書き込みは個人利用程度 | 投稿が多いなら日付などでパーティションを分割 |

### 今回の要件に対して妥当だった設計・実装

- **S3 キーに所有者を埋め込む**：`photos/<ownerSub>/<photoId>.<ext>` を呼び出しユーザーの `sub` から組み立てるため、他人のアップロードを自分の写真として登録できません。
- **操作ごとの最小権限**：`createUploadUrl` には PutObject だけ、`list` には Read だけと、Lambda 単位で S3 / DynamoDB の権限を分けています。`integrationPattern=isolated` を選んだ理由もこれでした。

### 本番利用に向けて修正・追加検証が必要な点

- **孤児オブジェクト**：`createUploadUrl` の後に `confirmUpload` が呼ばれないと、S3 に未登録の画像が残ります。ライフサイクルは不完全マルチパートと旧バージョンにしか効きません。
- **削除の一貫性**：S3 削除 → DynamoDB 削除の順で、途中失敗するとレコードだけが残ります。
- **写真バケットのアクセスログを Checkov 抑制で通した**：`CKV_AWS_18` を理由付きで抑制していますが、本番利用を想定するなら、アクセスログの要否を改めて確認すべきです。
- **実ファイル形式の検証はしていない**：`HeadObject` で確認できる `Content-Type` はアップロード時に設定されたメタデータです。実際のバイト列が JPEG / PNG などとして妥当かまで保証したい場合は、magic bytes や画像デコードによる検証が別途必要です。

## Case 2：CSV 分析サービス

### 与えたユーザーストーリー

```text
「ユーザーとして、大きなCSVファイルをアップロードすると、その内容を集計してグラフで確認できるようにしたい。
処理に時間がかかる場合でも、ブラウザを開いたまま待つ必要はないようにしてほしい。」
```

このケースでは、「時間のかかる処理を同期 HTTP リクエストで処理すべきではない」と要件から読み取れるかを見ます。

### Claude Code が解釈した要件

DESIGN.md の要件表を、機能要件と非機能要件に分けて示します。

**機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| F1 | CSV をアップロードできる | ストーリー | 画面から署名付き URL を取得し、ブラウザから S3 に直接 PUT |
| F2 | 大きなファイルを扱える | 「大きな」 | API Gateway（10 MB 制限）を経由しない。集計はストリーム処理でメモリに載せない |
| F3 | 内容を集計する | ストーリー | 列ごとに型を推定。数値列は件数 / 合計 / 最小 / 最大 / 平均 / 標準偏差 / ヒストグラム、文字列列はユニーク数と上位 20 件 |
| F4 | グラフで確認できる | ストーリー | Cloudscape の `BarChart`、列概要テーブル、サマリー KPI |
| F5 | ブラウザを開いたまま待たなくてよい | ストーリー | ジョブを DynamoDB に永続化し、S3 → SQS → Lambda でサーバ側で処理。後からジョブ一覧で確認。開いている間は 3〜5 秒のポーリング |
| F6 | 自分のジョブだけが見える | 補完（「ユーザーとして」） | IAM 認証、所有者 ID を保存、他人のジョブは NOT_FOUND |

**非機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| N1 | サイズと処理時間の上限 | 「大きな」 | 5 GiB（単一 PUT の上限）、Lambda 15 分 / 2048 MB |
| N2 | メモリ安全性 | 「大きな」 | 集計は「列数 × 定数」のメモリで動作（ユニーク値 10,000 / 列、サンプル 5,000 / 列） |
| N3 | 冪等性・耐障害性 | 「時間がかかる」 | S3 イベントの重複配信で `COMPLETED` 済みはスキップ。SQS 再試行 3 回 → DLQ。可視性タイムアウトは Lambda の 6 倍 |
| N4 | セキュリティ | 補完（Generator 既定 + 手書き） | KMS CMK、Block Public Access、SSL 強制、アクセスログ、WAF、CORS は CloudFront ドメインのみ、署名付き URL は 1 時間 |
| N5 | データ保持 | 補完 | 入力 CSV 7 日、結果 90 日、アクセスログ 90 日で削除 |
| N6 | 可観測性 | 補完（Generator 既定） | Powertools、`JobCompleted` などのメトリクス、10 万行ごとの進捗ログ |

メール通知（SNS / SES）は意図的に採用せず、「開いたまま待たなくてよい」はジョブ永続化と一覧画面で満たせると判断していました。完了通知は拡張候補としてレビュー項目に回されています。ジョブのライフサイクル（`PENDING_UPLOAD → QUEUED → PROCESSING → COMPLETED / FAILED`）も DESIGN.md に図示されていました。

### 選択した AWS サービス

| 役割 | 選択されたもの | Generator 既定か手書きか | 選択理由（DESIGN.md より要約） |
| --- | --- | --- | --- |
| Frontend | S3 + CloudFront（React + Cloudscape） | Generator（`ts#website --ux=cloudscape` を指定） | Cloudscape は `BarChart` などのチャート部品を標準で持つため。shadcn だとチャートを別途導入する必要がある |
| API | API Gateway REST + Lambda（tRPC、5 プロシージャ） | Generator（`ts#api` の既定 `rest-lambda`） | Generator 既定。WAF・アクセスログ付き |
| Authentication | Cognito User Pool + Identity Pool、**API は IAM 認証（SigV4）** | Generator（`ts#api` の既定 `auth=iam`、`ts#website#auth` の既定 `allowSignup=false`）。他の詳細ケースはここを Cognito に変えている | Identity Pool の一時クレデンシャルで API を呼ぶ。セルフサインアップは無効（管理者がユーザーを作る運用を想定） |
| Database | DynamoDB（ジョブ状態と所有者、GSI で所有者別一覧） | Generator + 手書き（テーブルは `ts#dynamodb`、ジョブ状態のエンティティは手書き） | キー参照と所有者別一覧だけの単純なアクセスパターン |
| Storage | **S3 DataBucket（手書き）**：`uploads/` と `results/` をプレフィックスで分離、KMS CMK、ライフサイクル | **手書き CDK**（対応する Generator なし） | 署名付き URL で直接 PUT。結果 JSON も同じバケットに |
| Async | **S3 イベント通知 → SQS（+ DLQ）→ Lambda（15 分 / 2 GB）** | **手書き CDK**（S3 通知、SQS、DLQ、イベントソースマッピング）+ Generator（関数本体は `ts#lambda-function`。Construct に props を追加する改変あり） | S3 → Lambda 直接に比べ、再試行回数・可視性タイムアウト・DLQ を明示的に制御できる。**Step Functions は単一ステップには過剰、EventBridge は再試行制御が SQS より弱い** と判断。**ECS/Fargate や Glue は 15 分を超える超大容量で必要になるが、まずはサーバレス最小構成** |
| 進捗確認 | DynamoDB のジョブ状態を API 経由でポーリング | **手書き**（アプリコード） | WebSocket / tRPC subscription は「画面を閉じてよい前提なので不要」 |
| CDN | CloudFront（SPA 配信のみ） | Generator（`ts#website` の既定） | ― |
| Security | WAF、KMS CMK（S3 / SQS 共通の `DataKey`）、Cognito MFA 必須（既定のまま）、S3 アクセスログ | Generator 既定（WAF、MFA 必須）+ 手書き（S3 / SQS 共通の CMK、アクセスログバケット） | SQS を SSE-KMS で暗号化しつつ S3 から直接イベント通知するため、S3 サービスプリンシパルに必要な KMS 権限を付与できるカスタマーマネージドキーを使用 |

### @aws/nx-plugin で利用した Generator

MCP ラッパーの呼び出し順です。

```text
tools
list-generators
generator-guide ts#website#auth
generator-guide ts#infra
generator-guide ts#lambda-function {"event":"S3SqsEventNotificationSchema"}
best-practices {"pages":["workspace","typescript-project","runtime-config","security","local-development"]}
generator-guide ts#dynamodb
generator-guide ts#api {"framework":"trpc","auth":"iam","infra":"rest-lambda"}
generator-guide ts#website {"ux":"cloudscape","framework":"react"}
generator-guide ts#project
generator-guide connection {"sourceType":"ts#trpc-api","targetType":"ts#dynamodb"}
generator-guide connection {"sourceType":"ts#react-website","targetType":"ts#trpc-api"}
```

`list-generators` の直後、3 番目に `ts#lambda-function` を `S3SqsEventNotificationSchema` で引いています。イベントスキーマ一覧の中から「S3 → SQS 経由」の型をこの時点で選んでいる、つまり **S3 と Lambda の間に SQS を挟む構成は、Generator を調べ始めた最初期に決まっていた** ことがログから分かります。

実行した Generator は 9 回です。

```bash
pnpm nx g @aws/nx-plugin:ts#infra --name=infra
pnpm nx g @aws/nx-plugin:ts#website --name=website --framework=react --ux=cloudscape --tanstackRouter=true --infra=cloudfront-s3
pnpm nx g @aws/nx-plugin:ts#website#auth --project=website --allowSignup=false
pnpm nx g @aws/nx-plugin:ts#api --name=api --framework=trpc --auth=iam --infra=rest-lambda --integrationPattern=isolated
pnpm nx g @aws/nx-plugin:ts#dynamodb --name=jobs --framework=electrodb --infra=dynamodb
pnpm nx g @aws/nx-plugin:ts#project --name=csv-processor
pnpm nx g @aws/nx-plugin:ts#lambda-function --project=@case3-csv-analytics/csv-processor \
  --name=process-csv --event=S3SqsEventNotificationSchema --infra=lambda
pnpm nx g @aws/nx-plugin:connection --sourceProject=@case3-csv-analytics/website --targetProject=@case3-csv-analytics/api
pnpm nx g @aws/nx-plugin:connection --sourceProject=@case3-csv-analytics/api --targetProject=@case3-csv-analytics/jobs
```

S3 バケット、SQS + DLQ、S3 イベント通知、Lambda の SQS イベントソースマッピングは Generator にないため、すべて `application-stack.ts` に CDK で手書きされていました。また、`ts#lambda-function` が生成した Construct にタイムアウト・メモリ・DLQ を渡す `props` がなかったため、**生成された Construct に引数を追加する小さな改変** をしています（`packages/common` に手を入れたのはこのケースだけです）。

### 最終的な AWS アーキテクチャ

![Case 2 のアーキテクチャ。CloudFront + S3 の SPA、Cognito、API Gateway REST（IAM 認証）+ Lambda ×5、DynamoDB Jobs、S3 DataBucket → SQS（+DLQ）→ Lambda csv-processor。ブラウザは署名付き URL で S3 に直接 PUT](/images/nx-plugin-user-story/case3-architecture.png)
*Case 2：CSV 分析サービス。S3 バケット、SQS、DLQ、イベント通知、イベントソースマッピングが手書き部分です。*

### AI が置いた前提（人間が確認すべきポイント）

| AI の判断 | 前提にした要件 | 要件が違えば |
| --- | --- | --- |
| API は IAM 認証（`ts#api` の既定のまま） | 既定を変える理由がなかった | Cognito オーソライザーにすると JWT クレームから直接 `sub` が取れる。現状は `cognitoAuthenticationProvider` 文字列を `:` で split して末尾を使う間接的な実装 |
| セルフサインアップ無効（`ts#website#auth` の既定のまま） | 管理者がユーザーを作成する運用 | 一般公開サービスなら有効化 |
| 完了通知なし（後からジョブ一覧で確認） | ユーザーは自分で見に来る | 完了メールが要るなら `COMPLETED` 更新時に SNS / SES |
| 進捗は 3〜5 秒のポーリング | 画面を開いている間だけ更新できればよい | リアルタイム性が要るなら WebSocket / SSE |
| 集計は Lambda（15 分 / 2048 MB） | 数 GB 級までで、処理時間は 15 分以内 | それを超えるなら Fargate / Glue |
| 集計は近似（ヒストグラムは 5,000 件のサンプル、ユニーク数は 10,000 で打ち切り、数値列の判定は 90%） | 概要が分かればよい | 厳密な統計が要るなら上限を外すか別基盤 |
| 入力 CSV は 7 日、結果は 90 日で削除 | 一時的な分析 | 監査や再集計が要るなら保持期間を変更。個人情報を含むなら短縮 |
| サイズ上限 5 GiB（単一 PUT の上限） | 5 GiB で足りる | 超えるならマルチパートアップロード |

### 今回の要件に対して妥当だった設計・実装

- **同期 HTTP で処理しない判断を、最初期に下している**：MCP ログ上、`list-generators` の次に `S3SqsEventNotificationSchema` を引いています。「アップロード完了イベントをキューに入れて非同期に処理する」構成は Generator の詳細を読む前に決まっていました。
- **S3 → Lambda 直接ではなく SQS を挟んだ理由が具体的**：再試行回数、可視性タイムアウト、DLQ の制御、失敗ファイルの事後調査。Step Functions / EventBridge / Fargate / Glue との比較も一段ずつ書かれています。
- **SQS + Lambda の定石を押さえている**：可視性タイムアウト = Lambda タイムアウト × 6、`reportBatchItemFailures` による部分失敗応答、`s3:TestEvent` の破棄。
- **Checkov の指摘を設計に取り込んだ**：初回の Checkov で SQS の KMS 暗号化（`CKV_AWS_27`）とログバケットのバージョニング（`CKV_AWS_21`）が失敗し、SQS を CMK 暗号化に変更しています。S3 から SSE-KMS で暗号化された SQS へ直接通知する場合、S3 サービスプリンシパルに KMS 権限を付与する必要があり、そのポリシーを変更できるカスタマーマネージドキーを使う必要があります。
- **プレフィックスごとのライフサイクル**：入力・結果・アクセスログで削除ポリシーを分けています。

### 本番利用に向けて修正・追加検証が必要な点

- **`QUEUED → PROCESSING` が無条件更新**：`COMPLETED` 済みはスキップしますが、同じジョブの通知が重複して届いた場合に、処理開始を条件付き更新で排他していません。`batchSize: 1` は 1 回の Lambda 呼び出しに渡すメッセージ数を制限する設定であり、Lambda の並列実行数を 1 にする設定ではありません。SQS + Lambda は重複処理が起こり得るため、`QUEUED → PROCESSING` にも条件式を入れる方が堅牢です。
- **DLQ の監視がない**：DLQ にメッセージが入ってもアラームは未実装です。
- **処理容量が未計測**：15 分 / 2048 MB でどこまで処理できるかは測っていません。

## Case 3：アクセス集中するチケット販売

### 与えたユーザーストーリー

```text
「ユーザーとして、イベントのチケットをオンラインで購入したい。
発売開始直後に大量のユーザーがアクセスしても、同じ座席が二重販売されないようにしてほしい。」
```

このケースでは、「大量のユーザー」からスケーラビリティを、「二重販売されない」から強い整合性や条件付き書き込み・トランザクションを読み取れるかを見ます。

### Claude Code が解釈した要件

DESIGN.md の要件表を、機能要件と非機能要件に分けて示します。

**機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| F1 | イベント一覧を見て、座席表から座席を選べる | ストーリー | `events.list` / `seats.list`、座席表 UI |
| F2 | 仮押さえ → 購入確定の 2 段階 | 補完 | `orders.hold` → `orders.confirm`。仮押さえは 5 分 |
| F3 | 仮押さえの期限切れは自動解放、明示的なキャンセルも可 | 補完 | 期限切れは読み取り時に判定（バッチ不要）、`orders.cancel` |
| F4 | 1 注文で最大 4 席 | 補完 | Zod スキーマとリポジトリの両方で検証 |
| F5 | 販売開始前は確保できない | 「発売開始直後」 | `salesStartAt` をサーバー時刻で判定 |
| F6 | 自分の注文の状態と履歴を見られる | 補完 | `orders.get` / `orders.listMine` |
| F7 | 管理者がイベントと座席レイアウトを登録できる | 補完 | `events.create`（Cognito の `admin` グループ限定） |

**非機能要件**

| # | 要件 | 出どころ | 設計上の対応 |
| --- | --- | --- | --- |
| N1 | 同じ座席が複数の注文に紐づくことが絶対にない | 「二重販売されない」 | DynamoDB `TransactWriteItems` + 座席ごとの条件式。アプリ側ロックや read-then-write に依存しない |
| N2 | 発売直後のスパイクで壊れない・詰まらない | 「大量のユーザー」 | サーバーレス構成、座席ごとに分散しやすい partition key、座席一覧 GSI のシャーディング |
| N3 | 二重クリックやネットワーク断で二重注文しない | 補完 | 購入試行ごとにクライアントで UUID を生成し、再試行でも同じ `orderId` を再利用。`attribute_not_exists` で重複を拒否 |
| N4 | 取れなかった座席が即座に分かる | 補完 | CONFLICT 応答に座席 ID を載せ、UI がその座席だけ選択から外す |
| N5 | 過剰アクセス・ボットからの保護 | 「大量のユーザー」 | WAF 管理ルール + IP 単位のレートベースルール、API Gateway スロットリング、Cognito 認証必須 |
| N6 | 監査・追跡可能性 | 補完 | 状態遷移の永続化、構造化ログ・メトリクス・X-Ray |

「大量アクセス」と「二重販売されない」の 2 語から、整合性・スケーラビリティ・冪等性・レート制限・監査まで展開できていました。

### 選択した AWS サービス

| 役割 | 選択されたもの | Generator 既定か手書きか | 選択理由（DESIGN.md より要約） |
| --- | --- | --- | --- |
| Frontend | S3 + CloudFront（React + Cloudscape） | Generator（`ts#website --ux=cloudscape` を指定） | 静的アセットは CloudFront でエッジ配信できる。ただし、この構成では購入 API 自体は CloudFront のキャッシュ対象ではなく、API Gateway へ直接到達するため、API のアクセス集中を CloudFront が吸収するわけではない |
| API | API Gateway REST + Lambda（プロシージャごとに 1 関数、9 個） | Generator + 手書き（API 本体は `ts#api`。操作別タイムアウトと書き込み権限の分離は手書き） | `orders.hold` だけタイムアウトを短く（10 秒）、`events.create` だけ長く（60 秒）など操作ごとの調整と最小権限のため isolated |
| Authentication | Cognito User Pool + Identity Pool、`admin` グループ | Generator + 手書き（`ts#website#auth`。`admin` グループは手書き） | 購入者識別（`sub`）と管理操作の認可（グループ）を標準機能で |
| Database | **DynamoDB（オンデマンド、単一テーブル、`TransactWriteItems`）** | Generator + 手書き（テーブルは `ts#dynamodb`、データモデル・条件式・シャーディングは手書き。生成された GSI2 は削除） | 「条件付き書き込みとトランザクションが、同一座席の二重販売を DB 層で排他する要件に直結する」。RDB の行ロックでも実現できるが、接続数スパイクへの対応（RDS Proxy、スケールアップ判断）で運用負荷が高い |
| Storage | なし | ― | ― |
| Async / Queue | **なし（同期 API）** | ―（意図的に採用せず） | 「SQS でキューイングして直列処理は順序公平性が上がるが、非同期 UX になり複雑さが見合わない。待合室やキューは負荷試験で上限に当たった段階で前段に足せる」 |
| Cache / Lock | なし（ElastiCache 不採用） | ―（意図的に採用せず） | 「永続層と別にロック層を持つと整合性の境界が増える。DynamoDB 単体で完結する方が単純」 |
| CDN | CloudFront | Generator（`ts#website` の既定） | ― |
| Security | WAF 管理ルール + **手書きの IP レート制限（1,000 req / 5 分）**、API Gateway のスロットリング、MFA 必須（既定のまま） | Generator 既定（WAF 管理ルール、MFA 必須）+ **手書き**（WAF の IP レート制限ルール） | 「過剰アクセスを API より手前で落とす」。なお、10,000 RPS / バースト 5,000 は多くのリージョンにおける API Gateway のアカウント・リージョン単位の既定クォータであり、Generator 固有の保証値ではない |

### @aws/nx-plugin で利用した Generator

MCP ラッパーの呼び出し順です。

```text
tools
list-generators
general-guidance
generator-guide ts#api {"framework":"trpc","auth":"cognito","infra":"rest-lambda"}
generator-guide ts#dynamodb
generator-guide ts#infra
generator-guide ts#website {"ux":"cloudscape","framework":"react"}
generator-guide ts#website#auth
generator-guide connection {"sourceType":"ts#trpc-api","targetType":"ts#dynamodb"}
generator-guide connection {"sourceType":"ts#react-website","targetType":"ts#trpc-api"}
best-practices {"pages":["workspace","typescript-project","security","runtime-config"]}
```

他のケースと違い、`list-generators` の次に真っ先に `ts#api` と `ts#dynamodb` のガイドを引いています。API と DynamoDB の組み合わせで排他を実現する方針が最初から固まっていたことがうかがえます。`ts#lambda-function` のガイドは引いていません（「期限切れ解放バッチが不要な設計にしたため、API 以外の Lambda が要らなかった」）。

実行した Generator は 7 回で、Case 1 と同じ組み合わせです。

```bash
pnpm nx g @aws/nx-plugin:ts#infra --name=infra
pnpm nx g @aws/nx-plugin:ts#dynamodb --name=ticket-table --framework=electrodb --infra=dynamodb
pnpm nx g @aws/nx-plugin:ts#api --name=ticket-api --framework=trpc --auth=cognito --infra=rest-lambda --integrationPattern=isolated
pnpm nx g @aws/nx-plugin:ts#website --name=ticket-web --framework=react --ux=cloudscape --tanstackRouter=true --infra=cloudfront-s3
pnpm nx g @aws/nx-plugin:ts#website#auth --project=ticket-web --allowSignup=true
pnpm nx g @aws/nx-plugin:connection --sourceProject=ticket-web --targetProject=ticket-api
pnpm nx g @aws/nx-plugin:connection --sourceProject=ticket-api --targetProject=ticket-table
```

つまり、**Generator の構成だけ見ると Case 1（写真共有）と Case 3（チケット販売）は同じ** です。難しさの差はすべて、Generator の外側にある手書き部分（データモデル、トランザクション、WAF ルール、Lambda の操作別設定）に現れています。

### 最終的な AWS アーキテクチャ

![Case 3 のアーキテクチャ。CloudFront + S3 の SPA、Cognito（admin グループ）、WAF（IP レート制限追加）+ API Gateway REST + Lambda ×9、DynamoDB 単一テーブル。Lambda から DynamoDB へは TransactWriteItems と条件式で二重販売を防止](/images/nx-plugin-user-story/case4-architecture.png)
*Case 3：チケット販売。使ったサービスの種類は Case 1 より少なく、難しさはデータモデルと条件式（手書き）に集中しています。*

### AI が置いた前提（人間が確認すべきポイント）

| AI の判断 | 前提にした要件 | 要件が違えば |
| --- | --- | --- |
| 決済はスコープ外。`confirm` は決済なしで確定 | 決済は別途組み込む | 決済オーソリ → `confirm` → キャプチャの順にし、`confirm` 失敗時にオーソリを取り消す補償処理が必要 |
| 仮押さえ 5 分、1 注文最大 4 席 | 一般的なチケット販売 | 決済フローの長さや興行の方針で変更 |
| 同期 API のみ（キュー・待合室なし） | 発売直後でも API Gateway / Lambda / DynamoDB の上限内に収まる | 上限を超える規模なら待合室（Virtual Waiting Room）やキューを前段に |
| 先着順の公平性は保証しない | 同時到達時の勝敗は不定でよい | 抽選や順番保証が要件なら前段の設計が必要 |
| MFA 必須（Generator 既定のまま） | セキュリティ既定を優先 | 一般消費者向けでは離脱要因。任意化を検討 |
| WAF の IP レート制限 1,000 req / 5 分 | 1 IP ≒ 1 ユーザー | 企業や大学など NAT 配下の多数ユーザーで誤検知するなら閾値を調整 |
| 座席表は 5 秒ポーリング、1 回で 8 本の Query | 同時閲覧者は限定的 | 多いなら CloudFront / API Gateway キャッシュや WebSocket |
| 管理者は Cognito の `admin` グループ、コンソールで追加 | 管理者は少数 | 管理画面や座席レイアウトの変更・削除 API が必要 |

### 今回の要件に対して妥当だった設計・実装

- **整合性を DB 層に置いた**：仮押さえは 1 つの `TransactWriteItems` に「注文の Put（`attribute_not_exists`）」と「座席 N 件の Update（`status = AVAILABLE OR (HELD AND holdExpiresAt < now)`）」を載せ、1 席でも条件を満たさなければ全体をロールバックします。「3 席中 2 席だけ確保できた」も「同じ座席が 2 注文に載る」も構造的に起きません。
- **ホットパーティションの回避**：座席ごとに partition key を分散させ、発売直後の書き込みが特定の partition key に集中することを避けています。DynamoDB の物理パーティションには書き込みスループットの上限があるため、eventId のような低カーディナリティなキーにアクセスを集中させるより、seatId を含めて負荷を分散しやすいキー設計にしています。
- **DynamoDB TTL を座席解放に使わなかった**：TTL の期限を過ぎても即時削除は保証されず、AWS のドキュメントでは通常「期限切れから数日以内」に削除されるとされています。そのため、5 分の仮押さえ解除のような厳密な期限判定には TTL を使わず、読み取り・更新時に `holdExpiresAt` を評価しています。
- **冪等キー**：購入試行ごとにクライアントで UUID を生成し、ネットワークエラーなどで再試行する際にも同じ `orderId` を再利用することで、同一試行の重複注文を防いでいます。

### 本番利用に向けて修正・追加検証が必要な点

- **整合性の担保が未検証**：二重販売防止の要である条件式は、DocumentClient のフェイクに対するテストしかなく、DynamoDB で意図どおり評価されることは確認できていません。
- **インフラ側のスパイク対策が既定値**：スパイク対策として追加されたのは WAF の IP レート制限だけで、API Gateway のスロットリングは Generator 既定（10,000 rps）のまま、Lambda の予約同時実行数は未設定、オンデマンドテーブルの初期スループット上限（新規作成直後は 4,000 WCU / 12,000 RCU 程度）も未対応です。

## 3 ケースを比較してみる

| 軸 | Case 1 画像共有 | Case 2 CSV 分析 | Case 3 チケット販売 |
| --- | --- | --- | --- |
| 主に見たかったこと | 標準的な Web アプリをどこまで組めるか | 長時間処理から非同期アーキテクチャを導けるか | 整合性・競合制御まで設計できるか |
| API の認証方式 | Cognito オーソライザー（JWT） | IAM 認証（SigV4、Generator 既定のまま） | Cognito オーソライザー（JWT） |
| サインアップ / MFA | 自己サインアップ可 / MFA 必須（既定） | 自己サインアップ不可（既定）/ MFA 必須（既定） | 自己サインアップ可 / MFA 必須（既定） |
| 認可の実装 | S3 キーに投稿者 `sub` を反映 + 削除時に投稿者チェック | 所有者 ID を属性に保存、他人は 404 | 注文の所有者チェック + `admin` グループ |
| データストア | DynamoDB + S3 | DynamoDB + S3 | DynamoDB |
| DynamoDB の設計 | GSI 2（全体フィード、投稿者別） | GSI 1（所有者別）、ジョブの状態遷移 | 座席単位で分散しやすい partition key、GSI 8 シャード、`TransactWriteItems` |
| 非同期処理 | なし | S3 通知 → SQS（+ DLQ）→ Lambda | なし |
| 重複・競合への対策 | なし | 完了済みジョブのスキップ。ただし同時重複には追加対策が必要 | トランザクション + 条件式 + 冪等キー |
| Generator 外の主な実装 | S3 バケット、アップロード/閲覧フロー | S3、SQS、DLQ、イベント通知、イベントソースマッピング | WAF レート制限、DynamoDB の条件式・トランザクション、Cognito グループ |
| Generator 実行回数 | 7 | 9 | 7 |
| Generator 生成物の改変 | なし | Lambda Construct に props 追加 | logger と local-server の小改変 |

3 ケースを通じて共通していたのは次の点です。

- **フロント / API / 認証 / DB の基幹となる「型」は共通**：3ケースとも `ts#website`（CloudFront + S3）+ `ts#api`（API Gateway REST + Lambda + tRPC）+ `ts#website#auth`（Cognito）+ `ts#dynamodb` を採用し、`ts#rdb`（Aurora）や `smithy`、`py#*`、`http-lambda` は検討したうえで見送っていました。プロンプトで「利用可能な場合は Generator を優先」と指示しているため、Generator でカバーされる構成に寄ること自体は指示の帰結です。一方で、Generator が用意されている Aurora を 3ケースとも退けて DynamoDB を選んだことや、Generator のない SQS・EventBridge・SES を必要なケースでは追加していることから、利用可能な Generator を機械的に組み合わせただけではなく、要件に応じたサービス選定も行っていたとみてよさそうです。
- **要件が Generator の守備範囲から外れるほど、AI が手書きする範囲が増える**：S3 や SQS といった AWS リソースだけでなく、トランザクション、条件式、GSI のデータモデル、冪等性など、アプリケーション側の設計も AI が補っていました。

## どこまで AI に任せられそうか

3 ケースの結果を、「AI に任せられた範囲」と「人間が持つべき範囲」に分けて整理します。

### AI に任せられた範囲

**Generator の発見と実行**

詳しく見た 3 ケースでは、`list-generators` → `generator-guide`（オプション付き）→ 実行という流れで、存在する Generator と利用方法を自分で調べていました。Generator の選択ミスや、存在しない Generator を呼ぼうとした形跡はありませんでした。少なくとも今回の検証では、この部分は安定していました。

**定番構成のサービス選定**

Cognito / API Gateway / Lambda / DynamoDB / S3 / CloudFront といったサーバーレス構成には、サービス名を一切与えなくても到達しました。さらに、CSV 分析では SQS、補助検証では EventBridge / SES と、Generator にないサービスも必要に応じて追加しています。

**ユーザーストーリーに書かれていない要件を読み取り、設計に落とす**

1〜2 文のストーリーから、単なる CRUD 以上の要件まで展開していました。

| ストーリーの一文 | 読み取った要件 | 設計への落とし方 |
| --- | --- | --- |
| 「ログインしたユーザーだけが投稿」（Case 1） | 認証に加えて、他人のアップロードを自分の写真として登録できないこと | S3 キーに所有者の `sub` を埋め込む |
| 「ブラウザを開いたまま待つ必要はない」（Case 2） | 同期 HTTP で処理しない。失敗時の再試行と隔離 | S3 → SQS（+ DLQ）→ Lambda |
| 「大量のユーザーがアクセスしても二重販売されない」（Case 3） | 強い整合性、冪等性、スパイク耐性、ボット対策 | `TransactWriteItems` と条件式、冪等キー、分散しやすい partition key、WAF レート制限 |

逆に、入れない判断もしています。Case 3 では、キューを挟むと購入 UX が非同期になるとして同期 API を選び、Case 1 では S3 イベント方式を検討したうえで同期の `confirmUpload` を選びました。

**レビュー項目の洗い出し**

Cognito の設定、DynamoDB のスループット、Lambda の同時実行数、WAF のレート制限、DLQ 監視など、本番導入前に確認すべき点も DESIGN.md に挙げていました。「何を人間が確認すべきか」を洗い出す作業自体はかなり任せられます。

### 人間が持つべき範囲

**要件の解釈そのもの**

「共有」とは誰に見せることか（Case 1）、CSV 処理完了時に通知は必要か（Case 2）、決済をどこまで含めるか（Case 3）。AI は不足している要件を補って先に進めますが、その補完がプロダクトとして正しいかは人間にしか決められません。

**定番から外れる選定**

Aurora、Fargate、Step Functions、ElastiCache なども比較候補には挙がりましたが、「過剰」「まずはサーバーレスで」と退けられました。今回の要件では妥当でも、想定負荷、運用体制、コスト、SLO が変われば別の選択肢が正解になる可能性があります。「選定できる」と「最適に選定できる」は別です。

**セキュリティ既定を緩める判断**

補助検証では、Claude Code が「個人向けサービスなのでサインアップの摩擦を減らす」と解釈し、Generator 既定の MFA 必須を任意に変更しました。理由は説明されていますが、セキュリティを弱める方向の変更をユーザーストーリーだけから決めてよいとは限りません。こうした変更は人間が明示的にレビューすべきです。

**数値の入る非機能要件**

Lambda の同時実行数、DynamoDB のスループット、API Gateway や WAF のレート制限、キューの可視性タイムアウト、データ保持期間などは、最終的には想定負荷や SLO、コスト要件に依存します。AI は妥当そうな初期値を置けても、その根拠が実際のサービス要件に合っているかまではユーザーストーリーだけでは判断できません。

## @aws/nx-plugin と コーディングエージェントの組み合わせの可能性

それでも、この組み合わせには AI に CDK を素で書かせるのとは違う良さがありました。組み合わせによる効果を「AI の責務を軽くするもの」と「人間のレビューを楽にするもの」に分けて整理します。

### AI の責務を軽くするもの

**定型部分の品質のばらつきを減らせる**

CloudFront + S3 + WAF + セキュリティヘッダ、API Gateway + Cognito オーソライザー + アクセスログ、DynamoDB の暗号化や PITR、Lambda の Powertools 統合など、定型部分は Generator / Construct を通して生成されます。AI が毎回すべてを手書きするより、実装のばらつきを抑えやすくなります。

**Checkov がフィードバックループになる**

Case 2 では、初回の Checkov で SQS の暗号化やログバケットのバージョニングが指摘され、Claude Code が設計を修正しました。Generator が build に組み込んだ静的検査が、Generator 外で AI が追加した CDK にも効いています。

### 人間のレビューを楽にするもの

**AI の判断過程を追跡しやすい**

MCP のログを見ると、どの Generator を調べ、どのオプションを検討したかが残ります。Case 2 では `S3SqsEventNotificationSchema` を早い段階で調べ、Case 3 では `ts#api` と `ts#dynamodb` から確認していました。最終コードだけを見るより、「何を候補にして、何を選んだか」を追いやすくなります。

**レビューの起点を作りやすい**

インフラ側の追加差分や判断理由は `application-stack.ts` と DESIGN.md に集まりやすいため、レビューの入口を作りやすくなります。ただし、今回の Case 3 のように、整合性の要は DynamoDB の条件式やトランザクションを呼ぶアプリケーションコード側にあります。認可・状態遷移・冪等性まで含めてレビューする必要があります。

### 限界

Generator にない部品は、結局 AI が CDK やアプリケーションコードを手書きします。Case 2 では S3 / SQS / DLQ を追加し、Generator が生成した Lambda Construct に props を足す改変まで行いました。Case 3 ではトランザクションや条件式の正しさが Generator によって保証されるわけではありません。

つまり、プラグインの守備範囲が広がるほど定型部分を任せやすくなる一方、**本当に難しい要件ほど Generator の外側に設計判断が残る**という構造です。

## まとめ

今回見たかったのは、「ユーザーストーリーからかなりのところまで AWS 構成を生成できるか」だけではなく、**Generator の外側にあるアーキテクチャ判断やデータ設計まで、コーディングエージェントがどこまで踏み込めるか**でした。

本文の 3 ケースを並べると、境界が分かりやすくなります。

- Case 1 では、認証付き Web アプリに必要な基本構成と S3 のアップロード方式まで組み立てました。
- Case 2 では、「長時間処理」という要件から S3 → SQS → Lambda の非同期構成を選びました。
- Case 3 では、「二重販売禁止」から DynamoDB のトランザクション、条件式、冪等キーまで設計しました。

実施したケースはいずれも、具体的な AWS サービス名や構成を与えず、ビルド・テスト・`cdk synth`・Checkov が通るアプリ + インフラまで到達しました。一方で、要件解釈、セキュリティ既定の変更、数値を伴う非機能要件は人間側に残ります。

そのため、「AI に設計を丸投げする」というより、AI に設計案と実装を作らせ、人間がその前提・境界条件・重要なアプリケーションロジックをレビューする使い方が現実的だと思います。

`@aws/nx-plugin` は、そのとき AI の判断対象を減らし、定型部分を再現性のある Generator に寄せる役割を果たします。設計・実装・検証のサイクルを速くしながら品質を担保するには、AI の出力だけでなく、開発者自身が業務ドメインや非機能要件を理解していることが引き続き重要です。
