# Okapiガイドとリファレンス

これはマイクロサービスの管理と実行のゲートウェイである、
Okapiのガイドとリファレンスです。

<!-- `make guide-toc.md`を実行し、その出力をここに含めて、必要に応じてこれを再生成します -->
* [イントロ](#イントロ)
* [アーキテクチャ](#アーキテクチャ)
    * [Okapiのウェブサービス](#Okapiのウェブサービス)
    * [デプロイとディスカバリ](#デプロイとディスカバリ)
    * [リクエスト処理](#リクエスト処理)
    * [ステータスコード](#ステータスコード)
    * [ヘッダー・マージ・ルール](#ヘッダー・マージ・ルール)
    * [バージョンと依存関係](#バージョンと依存関係)
    * [セキュリティ](#セキュリティ)
    * [Open Issues](#open-issues)
* [実装](#実装)
    * [Missing features](#missing-features)
* [コンパイルと実行](#コンパイルと実行)
* [Okapiの利用](#Okapiの利用)
    * [ストレージ](#ストレージ)
    * [Curl例](#Curl例)
    * [モジュール例](#モジュール例)
    * [Okapi自身を実行する](#Okapi自身を実行する)
    * [例1:簡単なモジュールをデプロイして利用する](#例1:簡単なモジュールをデプロイして利用する)
    * [例2:Authモジュールを追加する](#例2:Authモジュールを追加する)
    * [例3:アップグレード、バージョン、環境、`_tenant`インタフェース](#例3:アップグレード、バージョン、環境、`_tenant`インタフェース)
    * [例4:ModuleDescriptorを完成させる](#例4:ModuleDescriptorを完成させる)
    * [複数のインタフェース](#複数のインタフェース)
    * [クリーンアップ](#クリーンアップ)
    * [ラスタモードで実行](#ラスタモードで実行)
    * [Okapiのセキュリティ](#Okapiのセキュリティ)
    * [モジュール記述子（ディスクリプタ）の共有](#モジュール記述子（ディスクリプタ）の共有)
    * [テナントごとにモジュールをインストールする](#テナントごとにモジュールをインストールする)
    * [テナントごとにモジュールをアップデートする](#テナントごとにモジュールをアップデートする)
* [参考](#参考)
    * [Okapiプログラム](#Okapiプログラム)
    * [環境変数](#環境変数)
    * [ウェブサービス](#ウェブサービス)
    * [内部モジュール](#内部モジュール)
    * [デプロイメント](#デプロイメント)
    * [Docker](#docker)
    * [システムインタフェース](#システムインタフェース)

## イントロ

この文書では、Okapiに関連する以下の概念の概要を提供することを目的としています。

Okapiとその周辺のエコシステム全体（例：コアとモジュール）
Okapiの実装の詳細とOkapiの使用：具体的なWebサービスのエンドポイントとリクエスト処理
（リクエストとレスポンスエンティティの処理、ステータスコード、エラー条件など）の詳細の説明

Okapiは、マイクロサービスアーキテクチャに一般的に使用される、いくつかの異なるデザインパターンを実装しています。
それらの最も中心的なものは、Okapiコアの 'proxy'サービスによって実装されている、
いわゆる「APIゲートウェイ」パターンです。

概念的には、APIゲートウェイは、システムの単一のエントリーポイントとなるサーバーです。
これはオブジェクト指向デザインの[ファサードパターン]（http://en.wikipedia.org/wiki/Facade_pattern）
と同様です。
Okapiが非常に厳密に従っている[標準定義]（https://www.nginx.com/blog/building-microservices-using-an-api-gateway/）
によると、API Gatewayは、内部システムのアーキテクチャをカプセル化し、
各クライアントに合わせた、統合されたAPIを提供します。
次のコアの役割も含まれるでしょう。
認証、監視、ロードバランシング、キャッシング、リクエストの整形と管理、
スタティックなレスポンスのハンドリング_：
複数のリクエストへのブロードキャストとファイナル・レスポンスを返すことを可能にする
メッセージ・キュー・デザインパターン（当初は同期的かつ最終的にはおそらく、非同期で。）
最後に、Okapiはサービス・ディスカバリー・ツールとしての役割を果たすことによって、
サービス間のコミュニケーションを容易にします：サービスBと話したいサービスAが知っている必要があるのは、
HTTPインタフェースだけです。
Okapiが利用可能なサービスのレジストリを検査して、サービスの物理的なインスタンスを特定するからです。

Okapiは設定可能で拡張可能であるように設計されています。
ソフトウェア自体のプログラム的な変更をすることなく、
Webサービスエンドポイントを新規で公開したり、
既存の品質を良くしたりすることができます。

OkapiコアWebサービスをコールすることで、新しいサービス（Okapiから見た'modules'）の登録が行われます。
登録および関連するコア管理タスクは、サービス・プロバイダ・アドミニストレーターによって実行されます。
この設定可能性と拡張性により、アプリストアでテナントごとに
オンデマンドでサービスまたはサービス・グループ（「アプリケーション」）を有効または無効にすることができます。


## アーキテクチャ

OkapiのWebサービス・エンドポイントは、おおよそ2つに分けられます：
（1）時に「コア」と呼ばれる一般モジュールとテナント管理API - 当初はOkapi自体の一部でしたが、独立したサービスになるかもしれません
（2）モジュールが提供するビジネスロジック特有のインターフェースにアクセスするためのエンドポイント
例：パトロン管理またはサーキュレーション。 
このドキュメントでは、前者の詳細と、後者の許可されたフォーマットとスタイルの一般的な概要を提供します。

現在のフォームでは、コアなOkapiウェブサービスの仕様は、[RAML]（http://raml.org/）
（RESTful APIモデリング言語）を取り込んでいます。 [参考文献]（＃web-service）セクションを参照してください。 

しかし、この仕様では、特定のモジュールによって公開される実際のAPIエンドポイントについてはアサンプションをほとんどたてておらず、
基本的に未定義にしています。 目標は、共通トランスポートプロトコル（HTTP）の基本要件のみで、
それらのAPI（RESTful vs RPCとJSON vs XMLなど）のさまざまなスタイルとフォーマットを許可することです。

いくつかの特殊なケース（例えば、メッセージキューと同様の動作のための真の非同期プロトコルのようなバイナリプロトコルなど、
非HTTPを統合する能力）に関して、トランスポートプロトコルのアサンプションは解除されるか、回避されることが思い描かれています。


### Okapiのウェブサービス

前述のように、Okapi独自のWebサービスはモジュールの設定、構成、有効化、テナントの管理のための基本的な機能を提供します。 
コア・エンドポイントは次のとおりです。

 * `/_/proxy`
 * `/_/discovery`
 * `/_/deployment`
 * `/_/env`

特別な接頭辞 `/_` はモジュールによって提供される拡張ポイントから
Okapi内部のWebサービスへのルーティングを区別するために利用されます。

 *  `/_/proxy` エンドポイントはプロキシサービスの設定に使用されます：
    私たちが知っているモジュール、リクエストがどのようにルーティングされるべきか、
    私たちが知っているテナント、どのモジュールがどのテナントで有効になっているかを特定します。
    
 * `/ _ / discovery`エンドポイントは、クラスタ上のサービスIDからネットワークアドレスへのマッピングを管理します。
    情報がそこにポストされ、プロキシサービスがそれを照会して、必要なモジュールが実際に利用可能な場所を見つけます。 
    また、モジュールを一気にデプロイしてレジスターするためのショーカットを提供します。
    クラスタ内のノードを全てカバーする検出エンドポイントは1つしかありません。
    ディスカバリ・サービスへのリクエストはモジュールを特定のノード上にデプロイすることもできます。そのため、
    デプロイを直接呼び出すことが必要になることはほとんどありません。

 * `/ _ / deployment`エンドポイントはモジュールの配備を担当します。
    クラスタ化された環境では、各ノードで実行されているデプロイメント・サービスは一つであるべきです。
    そのノードでプロセスを開始し、さまざまなサービスモジュールにネットワークアドレスを割り当てる役割を担います。
    これは、ほとんどがディスカバリ・サービスによって内部で利用されますが、
    一部のクラスタ管理システムはそれを利用することができるように開かれています。

 * `/ _ / env`エンドポイントは、デプロイ時にモジュールに渡されるシステム全体のプロパティである
    環境変数を管理するために使用されます。

これら4つの部分は別々のサービスとして実装されています。
そのため、代替のデプロイとディスカバリー方法を使用することが可能です。
もし選択したクラスタリングシステムがそのような方法を提供しているのであれば。

![Module Management Diagram](module_management.png "Module Management Diagram")

#### 「モジュール」とは？

Okapiエコシステム内のモジュールは、コンテンツよりもふるまい（つまり、_interface contract_の観点から定義されます。
パッケージまたはアーカイブとしてのモジュールの正確な定義がないことを意味しています。
例えば、基礎となる標準化されたファイル構造があること。
これらの詳細は特定のモジュール実装に委ねられています（前述の通り、
Okapiのサーバー側モジュールは、あらゆるテクノロジー・スタックを利用できます）。

したがって、以下の特性が明らかになっているソフトウェアは、Okapiモジュールになり得ます：

* RESTスタイルのWebサービス・プロトコル（通常は、必須ではありませんが、JSONペイロードを使用します）を利用するHTTPネットワークサーバーであること。

* デスクリプタ・ファイル、すなわち
[`ModuleDescriptor.json`]（../ okapi-core / src / main / raml / ModuleDescriptor.json）、
これはモジュールの基本的なメタデータ（id、nameなど）を宣言し、
モジュールの他のモジュール（正確にはインタフェース識別子）との依存関係を指定し、
すべての"提供された"インターフェースにレポートすること。

* `ModuleDescriptor.json` がすべての `routes` (HTTP パスとメソッド)のリストを持っていること。
ルートは与えられたモジュールがハンドルすることで、モジュールへトラフィックをプロキシするために必要な情報をOkapiに提供します。
(これは単純化されたRAML仕様と同様です。)

* この章で定義されているバージョン管理規則に従っていること。
[_Versioning and Dependencies_](#versioning-and-dependencies).

* WIP: 監視と計装に必要なインタフェースを提供すること。

ご覧のとおり、これらの要件のいずれもデプロイメントのルールを明確に述べているわけではありません。
そのため、第三者ウェブサービス（例えば、公的にアクセス可能なインターネットサーバのAPI）
をOkapiモジュールとして統合することが完全に可能となります。

つまり、エンドポイントスタイルの過程を立てることと、バージョン管理の
セマンティクスは、Okapiで必要とされるものによく似ています。
それを記述するのに適切なモジュール記述子を書くことができます

ただし、Okapiには、自身が管理するクラスタ上で、Okapiがサービスのネイティブな起動、実行、監視を可能にする
追加のサービスが含まれています（サービス・デプロイメントとディスカバリー）。

それらの_ネイティブモジュール_初以下の記述子ファイル
[`DeploymentDescriptor.json`]（../ okapi-core / src / main / raml / DeploymentDescriptor.json）を必要します。
これはモジュールの実行方法に関する低いレベルの情報を指定します。 
また、ネイティブ・モジュールは、Okapiのデプロイメント・サービスでサポートされている
パッケージオプションの1つに従ってパッケージ化する必要があります：
この時点では、各ノードの実行可能ファイル（およびすべての依存ファイル）または自己完結型
Dockerイメージを使用して、中央から実行可能ファイルを配布します。


#### APIガイドライン

Okapiの独自のWebサービスは必須で、その他のモジュールは推奨として、
実際に可能な限りこれらのガイドラインに準拠する必要があります。

 * パスの後続のスラッシュは書かないこと
 * 常に適切なJSONを期待して返す
 * 主キーは常に 'id'と呼ばれるべきである

私たちは、Okapiのコードを模範的なものにすることで、
他のモジュール開発者がエミュレートする例としようとしています。

#### Core Okapi Web Service Authentication and Authorization

コアサービス（ `/ _ /`パスの下のすべてのリソース）へのアクセスはサービスプロバイダ（SP）管理者に付与されます。
これらのサービスが提供する機能は、複数のテナントにまたがっているためです。

SP管理者の認可と認証の詳細は後の段階で定義されるべきであり、
特定のサービスプロバイダの認証システムにフックできる外部モジュールによって提供される可能性が高いです。

### デプロイとディスカバリ

複数のステップからなるプロセスによって、テナントがモジュールを利用できるようになります。
いくつかの異なる方法で実行されますが、最も一般的なプロセスは次のとおりです:

 * ModuleDescriptorを `/ _ / proxy`にポストし、私たちがそのようなモジュールについて知っていること、
 それが提供するサービス、およびそれが依存するものをOkapiに伝えます。

 * `/ _ / discovery`に、任意のノードで実行したいモジュールにPOSTし、
 そのノードのデプロイ・サービスに必要なプロセスを開始するように伝えます。

 * 特定のテナントに対してモジュールを有効にします。

外部の管理プログラムがこれらの要求を行うと仮定しています。 
適切なOkapiモジュールそのものではありえません。なぜならモジュールがデプロイされる前に実行される必要があるからです。
テストについては、本書の後半にあるcurlコマンドライン[examples]（＃using-okapi）を参照してください。

別の方法として、モジュールIDではなく、完全なLaunchDescriptorをディスカバリーに渡す方法があります。
この場合、ModuleDescriptorはLaunchDescriptorさえも持ちません。 
これはかなり異なるノードのクラスターで実行されている場合や、ファイルの場所を正確に把握したい場合に便利です。
これはOkapiクラスターの実行方法として考えている方法ではありませんが、選択肢は開かれたままにしておきたいと考えています。

もう一つの代替案はさらに低いレベルに進み、LaunchDescriptorを任意のノード上の `/_/deployment`に直接POSTすることです。
つまり、管理ソフトウェアは個々のノードと直接話さなければいけないということですが、
ファイアウォールなどに関するあらゆる種類の疑問が生じます。
しかし、フルコントロールが許可されます。それにより、通常とは違ったクラスタリングの設定に役立つ可能性があります。
それでもやはり、OKAPIにモジュールについて知らせるために、
ModuleDescriptorを`/_/proxy`にPOSTする必要があること、
そして`/_/deployment` が `/_/discovery`にデプロイしたモジュールを通知することに注意をしてください。

もちろん、デプロイを管理するためにOkapiを使用しなくてはならないというわけではありません。
DeploymentDescriptorを`/_/discovery`にPOSTし、LaunchDescriptorの代わりにURLを与えてもよいです。
それにより、Okapiにサービスが実行される場所を教えることができます。
それでも、URLを先ほどPOSTしたModuleDescriptorに接続するためにサービスIDが必要です。
前の例とは異なり、モジュールのこのインスタンスを特定するために、`/_/discovery`に固有のインスタンスIDを提供する必要があります。

おそらくクラスタ内外の異なるノード上で、同じモジュールを別々のURLで動作させるために必要です。

このメソッドはクラスタ外にあるOkapiモジュールを使用する場合や
コンテナシステム、モジュールが稼働している恐らくは異なるURL上のCGIスクリプト
を利用する場合に便利です。

デプロイメントとディスカバリーの内容は一時的なものであり、Okapiはデータベース上にどちらも保存しません。
ノードが停止すると、ノード上のプロセスも終了します。
再起動すると、Okapi、または他の何らかの方法を経由してモジュールを再度デプロイする必要があります。

ディスカバリーデータはクラスタ上で1つのOkapiが実行されている限り、共有地図に保存されているので、
マップは存続します。 しかし、全体のクラスターがダウンすると、
ディスカバリーデータが失われます。 とにかくそのような時には、あまり役に立たないでしょう。

対照的に、 `/ _ / proxy`にPOSTされたModuleDescriptorsは、データベースに保持されます。

### リクエスト処理

モジュールはリクエストを処理する2つの方法としてハンドラとフィルタを宣言することができます。
それぞれのパスに対して正確に1つのハンドラが必要です。 それは デフォルトでは`リクエスト - レスポンス`
タイプになるでしょう（下記参照）。 
ハンドラが見つからない場合、Okapiは404 NOT FOUNDを返します。

各リクエストは、1つまたは複数のフィルタを通じてパスされます。 `フェーズ`は、
フィルタが適用される順序を決めます。
現時点では少し過剰なようですが、`auth`という、ハンドラの前に呼び出されるフェーズが一つしかありません。
ハンドラのアクセス許可のチェックに使用されます。
後でより多くのフェーズを導入するでしょう。
例えば、ハンドラによって処理されたリクエストの後に監査ログを書き込むフェーズなど。

（以前のバージョンでは、ハンドラとフィルタを
順序を制御するための数値レベルを持つパイプラインに1つにまとめていました。
それはバージョン1.2で煩わしいとされ、バージョン2.0では削除されます）

ModuledescriptionのRoutingEntryの `type`パラメータは、
どのようにリクエストがフィルタとハンドラに渡され、どのようにレスポンスが処理されるかを制御します。
現在、以下のタイプをサポートしています：

 * `headers` -- モジュールはヘッダ/パラメータのみに関心があり、
それを検査して、ヘッダ/パラメータの有無と
それに対応する値に基づいてアクションを実行することができます。
このモジュールは、レスポンス内にエンティティを返すことを期待されていません。
さらなる実行のチェーンまたは、エラーの場合は、即時終了を制御するための
ステータスコードのみを返します。
モジュールは以下のヘッダー操作規則に従い、
完全なレスポンスヘッダーリストにマージされるある種のレスポンスヘッダーを返すかもしれません。

 * `request-only` -- モジュールは完全なクライアント・リクエスト：ヘッダ/パラメータとリクエストに付けられた
エンティティのボディーに興味があります。
聯本巣の変更されたバージョン、または新しいエンティティを作りませんが、
関連するアクションをおこし、追加のオプションヘッダーと、さらなる処理または終了を意味するステータスコードを返します。
エンティティが返された場合、Okapiはそれを破棄し、
パイプラインの続くモジュールにオリジナルのリクエストボディをフォワードし続けます。

 * `request-response` -- モジュールはヘッダー/パラメーターとリクエストボディの両方に興味があります。 
 また、レスポンスにエンティティを返すことを期待されています。
例えば、モジュールがフィルターとして動く場合に改変されたリクエストボディを返し、
その後、返されたレスポンスは新しいリクエストボディとして続くモジュールにフォワードされる、などです。
処理のチェーンまたは終了はレスポンスのステータスコードを経由して制御され、
下記の規則を使ってレスポンスヘッダーの完全なレスポンスにマージして返します。

* `redirect` -- モジュールはこのパスを直接提供しませんが、いくつかの他のモジュールがサービスを提供している
別のパスへリクエストをリダイレクトします。 
これはより複雑なモジュールをより単純な実装の上に積み重ねるための
メカニズムとして意図されています。
例えば、ユーザを編集およびリストするためのモジュールは、
ユーザーとパスワードを管理するモジュールによって拡張されています。

それはユーザーの作成と更新を処理する実際のコードを持つでしょうが、
ユーザをリストして取得する多めのリクエストをより単純なユーザーモジュールにリダイレクトするかもしれません。
ハンドラ（またはフィルタ）がリダイレクトとしてマークされている場合は、
リダイレクト先を指定するredirectPathをもっていなければなりません。

ほとんどのリクエストは、 `request-response`タイプである可能性があります。
最も強力ですが潜在的に最も非効率なタイプです。
なぜなら、コンテンツをモジュールとの間でストリーミングする必要があるからです。
もっと効率的なタイプを使うことができるのであれば、そうするべきでしょう。
たとえば、認証モジュールのアクセス許可チェックでは、リクエストのヘッダーのみが参照され、
ボディは返されません。ですので、`headers`タイプとなります。
しかし、同じモジュールの初期ログインリクエストはログインパラメータを決定するために
リクエストボディを参照します。そして、メッセージも返します。なので、
 `request-response`タイプでなければなりません。

Okapiには、モジュールが例外的にX-Okapi-Stopヘッダーを返す機能があります。
このモジュールが返す結果をもって、Okapiはパイプラインを終了させることになります。
これは慎重に使用するように作られています。
たとえば、ログイン・パイプラインの中のモジュールは、
ユーザがセキュアなオフィスのIPアドレスから来ていると、
ユーザがすでに認証されていると結論付け、
ログイン画面の表示に導くイベントのシーケンスを中断するかもしれません。

<a id="chunked"/>OkapiはHTTP 1.0、HTTP 1.1のどちらのリクエストも受け付けますが、
モジュールにコネクションを作るためにチャンク・エンコーディングにはHTTP 1.1を使います。

### ステータスコード
パイプラインの継続または終了は、
実行されたモジュールによって返されたステータスコードによって制御されます。
標準[HTTPステータスコード](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) 
の範囲がOkapiで受け入れられます：

 * 2xx レンジ: OKリターン・コード。 このレンジのコードが
モジュールによって返された場合、Okapiはパイプラインの実行を継続し、
上記の規則に従って、連続したモジュールに情報を転送します。
チェーンの最後には、呼び出された最後のモジュールによって返されるステータスが、
呼び出し元に返されます。

 * 3xx レンジ: リダイレクト・コード。 パイプラインが終了され、
レスポンス（任意の `Location`ヘッダを含む）が呼び出し元にすぐに返されます。

 * 4xx-5xx レンジ: ユーザー・リクエスト・エラー、または内部システムエラー。
この範囲のコードがモジュールによって返された場合、Okapiは直ちにチェーン全体を終了し、
コードを呼び出し元に返します。

### ヘッダー・マージ・ルール

Okapiは、前のモジュールからのレスポンスを、パイプライン内の次のモジュールにフォワードします
（例えば、追加のフィルタリングや処理のため）。
ですので、初期リクエストのヘッダによっては、無効となるものがでてきます。 
例えば、モジュールがエンティティを異なるコンテンツタイプに変換したり、
そのサイズを変更する時などです。
リクエストが次のモジュールにフォワードされる前に、モジュールのレスポンス・ヘッダーの値に基づいて、
無効なヘッダーを更新する必要があります。 
同時に、Okapiは、処理パイプラインが完了したときに、
元のクライアントに返される最終応答を生成するため、
一連のレスポンス・ヘッダーも収集します。

両方のヘッダー・セットは、次の規則に従って変更されます。:

 * リクエスト・エンティティ・ボディ（例えば、Content-Type、Content-Lengthなど）に
関するメタデータを提供するすべてのヘッダーは、
返される最後のレスポンスからリクエストにマージされます。

 * 特別なデバッグおよびモニタリング・ヘッダーの追加セットが、
 最終の応答から現在の要求にマージされます（次のモジュールにそれらをフォワードするために）。
 
 * レスポンス・エンティティ・ボディに関するメタデータを提供するヘッダーのリストは、
 最終的なレスポンス・ヘッダー・セットにマージされます。

 * 特別なヘッダー（デバッグ、監視）の追加セットまたは最終レスポンスで表示されるべきその他のヘッダーが、
 最終レスポンス・ヘッダー・セットにマージされます。

Okapiは、どのモジュールに対するリクエストであっても、常にX-Okapi-Urlヘッダーを追加します。 
これにより、必要な場合に、Okapiをさらに呼び出す方法をモジュールに伝えます。
このURLは、Okapiの起動時にコマンドラインで指定することができ、
複数のOkapiインスタンスの前にあるロードバランサを指し示すことができます。

### バージョンと依存関係

モジュールは1つ以上のインタフェースを提供し、他のモジュールが提供するインタフェースを使用することができます。 
インターフェイスにはバージョンがあり、依存関係には特定のバージョンのインターフェイスが必要な場合があります。 
Okapiは、モジュールがデプロイされるたびに、またモジュールがテナントに対して有効になっているときに、
依存関係とバージョンをチェックします。

注：同じインターフェースを提供する複数のモジュールを持つことができます。
これらは同時にOkapiにデプロイすることができますが、
特定のテナントに対して、特定の時に有効にできるのはひとつだけです。
たとえば、利用者を管理するのに、ローカルデータベースをベースとした方法、
外部システムと対話する方法の、2つの方法があります。 
インストールは両方を知ることができますが、各テナントはどちらか一方を選択する必要があります。

#### Version numbers

モジュール・ソフトウェアのバージョンは、3-part versioning schemeを利用します。
（例えば、3.1.41）これは、[セマンティック・バージョニング](http://semver.org/)
に非常によく似ています。インターフェースのバージョンは最初の2つを利用します。
これは実装のバージョンがないためです。

最初の数字がメジャーバージョンです。 
例えば機能の削除やセマンティクスの変更など、
厳密にバックワードの互換性があるとはいえない変更を加える際はいつも、
この値を増やす必要があります。

2番目の数字がはマイナーバージョンです。
新しい機能やオプションのフィールドの追加など、
バックワードの互換性がある変更を行う際はいつも、
この値を増やす必要があります。

3番目の数字がソフトウェアのバージョンです。 
バグの修正や効率の改善など、インタフェースに影響を与えない変更をした場合に増やす必要があります。

すべてのモジュールでこのバージョン管理スキーマを使用することを強くお勧めしますが、
Okapiはモジュール用にこれを強制はしません。 
その理由は、Okapiはモジュールのバージョンについて何も知る必要がないからです
 -- 互換性のあるインターフェースについてのみを心配しています。

Okapiは、同じIDを持つすべてのモジュールが実際には同じソフトウェアになることを要求しています。 
私たちは、短い名前とそれに続くソフトウェアバージョンで構成されるIDの使用規約を採用しました。 
たとえば、 "test-basic-1.2.0"とします。

インタフェースバージョンをチェックするとき、Okapiは、メジャーバージョン番号が必要なものと正確に一致し、
マイナーバージョンが少なくとも必要なだけ高いことを要求します。

モジュールがインタフェース3.2を必要とする場合、Okapiは以下を受け入れます：
* 3.2 -- 同じバージョン
* 3.4 -- より高いマイナーバージョン、互換性のあるインターフェース

しかし、Okapiは次を拒否します:
* 2.2 -- より低いメジャーバージョン
* 4.7 -- より高いメジャーバージョン
* 3.1 -- より少ないマイナーバージョン

詳しい説明を見る
[Version numbers](http://dev.folio.org/community/contrib-code#version-numbers).

### セキュリティ

セキュリティに関する議論のほとんどは、独自の文書に移行されており、
[Okapi Security Model](security.md).
このOkapiガイドのこの章では、簡単な概要を説明します。

セキュリティ・モデルに重要なのは、以下の3点です:
* Authentication（認証） -- ユーザーが誰であるかを知っています。
* Authorization（認可） -- ユーザーがこのリクエストを行うことが許可されています。
* Permissions（許可） -- ユーザー・ロールから詳細パーミッションまでのマッピング

この作業のほとんどはモジュールに委託されているので、Okapi自体はそれほど多くの作業をする必要はありません。 
しかし、それはまだ全体の操作を調整する必要があります。

すべての面倒な細部を無視して、これがどのように動作するか：
クライアント（たぶんWebブラウザ上にあるが、本当に何でもよいです）は `/ authn / login`サービスを呼び出して自分自身を識別します。
テナントに応じて、 `/ authn / login`リクエストを扱う異なる認証モジュールを持つことができ、
それらは異なるパラメータ（ユーザ名とパスワードが最も可能姓が高いですが、
単純なIP認証からLDAP 、OAuth、または他のシステム）を取ることがあります。

authorization（認証）サービスはトークンをクライアントに戻し、
クライアントはOkapiに対して行うすべてのリクエストの中で,
このトークンを特別なヘッダーに渡します。
Okapiは、リクエストを満たすために呼び出されるモジュールの情報と、それらのモジュールが必要としているアクセス権と、
特別なモジュールレベルのアクセス権を持っているかどうかの情報とともに、認可モジュールに渡します。 
authorization（認証）サービスはpermission（許可）を検査します。 必要な権限がない場合、リクエスト全体が拒否されます。 
すべてがうまくいけば、モジュールは必要なpermission（許可）に関する情報と、
場合によってはいくつかのモジュールに渡される特別なトークンを返します。

Okapiはリクエストをパイプラインの各モジュールに順番に渡します。 
それぞれが必要なpermission（許可）の情報を取得するので、
必要に応じて動作を変更できるようになります。
また、後続の呼び出のために使用できるトークンを取得します。

Okapiソースツリーに含まれる簡単なokapi-test-auth-moduleモジュールは、このスキームの多くを実装していません。 
それはOkapiが処理する必要がある部分をテストするのに役立てるためにあります。

### Open Issues

#### Caching

Okapiは、特にビジー状態の読み取りが重いマルチモジュールパイプラインで、
モジュール間に追加のキャッシングレイヤーを提供できます。 
この点で標準のHTTPメカニズムとセマンティクスに従うことを計画しており、
実装の詳細は今後数か月以内に確立される予定です。

#### Instrumentation and Analytics

マイクロ・サービス・アーキテクチャでは、システム全体の堅牢性と健全性を確保するために、監視が重要です。 
有用な監視を提供する方法は、リクエスト処理パイプラインの各実行ステップの前後に、
明確に定義された計測ポイント（「フック」）を含めることです。
監視以外にも、実行中のシステムで問題をすばやく診断（「ホット」デバッグ）し、
パフォーマンスのボトルネック（プロファイリング）を発見するためには、計測は非常に重要です。 
我々は、これに関して確立された解決策を検討していています。 
例えば、JMX、Dropwizard Metrics、Graphiteなどです。

マルチモジュール・システムは、多種多様なメトリクスおよび膨大な量の測定データを提供することができます。 
このデータのほんの一部は実行時に分析することができ、
そのほとんどは分析のために後の段階で取り込まなければなりません。
簡単なポストファクトム分析に適した形式でデータをキャプチャして保存することは分析にとって不可欠であり、
オープンソリューションと一般的なソリューションとOkapiの統合を検討しています。

#### Response Aggregation

Okapiはパイプラインの順次実行を想定しており、
各レスポンスをパイプラインの次のモジュールに転送するため、
現在のところ、Okapiではレスポンスの集約は直接サポートしていません。 
このモードでは、複数のモジュールと通信し（Okapiを介して、提供された認証とサービスの検出を保持するために）、
レスポンスを結合する集約モジュールを実装することは完全に可能です。 
さらなるリリースでは、レスポンス集約のより包括的なアプローチが評価されるでしょう。

#### Asynchronous messaging

現在、Okapiは、モジュール間の転送プロトコルとして、フロントエンドとシステム内の両方でHTTPを想定し実装しています。 
HTTPはリクエスト - レスポンスパラダイムに基づいており、非同期メッセージング機能を直接には含みません。
しかしながら、HTTPの上で非同期動作モードをモデル化することは完全に可能です。 
例えば、ポーリング手法やWebSocketのようなHTTP拡張機能を使用します。 
今後のOkapiのリリースでは、非同期アプローチを深く検討し、いくつかのオープンメッセージングプロトコル（STOMPなど）をサポートする予定です。

## 実装

私たちは、Okapiの基本的な実装を行っています。 以下の例は、現在の実装で動作するはずです。

### Missing features

 この時点では、大きなものはありません。

## コンパイルと実行

最新のソフトウェアのソースは、次の場所にあります。
[GitHub](https://github.com/folio-org/okapi).

ビルド要件は次のとおりです。

 * Apache Maven 3.3.1 or later.
 * Java 8 JDK
 * [Git](https://git-scm.com)

いつものように、rootではなく、すべての開発と実行を通常のユーザーとして実行してください。
したがって、これらの要件下であれば、ビルドが可能となりました：
```
git clone --recursive https://github.com/folio-org/okapi.git
cd okapi
mvn install
```
インストールルールは、いくつかのテストも実行します。 テストは失敗しないべきです。
もし失敗したら、それを報告してください、そして、その間にフォールバックをしてください：

```
mvn install -DskipTests
```

成功した場合、  `mvn install` の終わり近くにこの行が出力されるはずです：

```
[INFO] BUILD SUCCESS
```

okapiディレクトリにはいくつかのサブモジュールが含まれています。 これらは：

 * `okapi-core` -- ゲートウェイサーバー自身。
 * `okapi-common` -- ゲートウェイとモジュールの両方で使用されるユーティリティ。
 * `doc` -- このガイドを含むドキュメント。
 * `okapi-test-auth-module` -- 認証情報をテストするための単純なモジュール。
 * `okapi-test-module` -- テスト目的でHTTPコンテンツをマングルするモジュール。
 * `okapi-test-header-module` -- ヘッダーのみのモードをテストするモジュール。

（ `pom.xml`で指定されたビルド順に注意してください：
テストは以前のテストに依存しているため、okapi-coreは最後にしなければなりません。）

The result for each module and okapi-core is a combined jar file
with all necessary components combined, including Vert.x. The listening
port is adjusted with property `port`.

各モジュールとokapi-coreの結果は、Vert.xを含むすべての必要なコンポーネントが組み合わされたjarファイルです。 
リスニングポートはプロパティ `port`で調整されます。

たとえば、okapi-test-auth-moduleモジュールを実行してポート8600でリスンするには、次のようにします:

```
cd okapi-test-auth-module
java -Dport=8600 -jar target/okapi-test-auth-module-fat.jar
```
同様に、okapi-coreを実行するには、そのjarファイルを指定します。
okapi-coreにどのモードを実行するかを指示するコマンドで、コマンドライン引数を追加する必要もあります。
単一のノードでokapiを再生する場合、 `dev`モードを使用します。

```
cd okapi-core
java -Dport=8600 -jar target/okapi-core-fat.jar dev
```

利用可能な他のコマンドがあります。 これらの詳細は `help`で補足してください。

ゲートウェイを実行するMavenルールは、メイン・ディレクトリの `pom.xml`の一部として提供されます。

```
mvn exec:exec
```
これでokapi-coreが起動し、デフォルトのポート：9130でリスンします。

```
mvn exec:exec@debug
```
このコマンドはMaven >= 3.3.1が必要です。
ポート5005でデバッグ・クライアントを待機します。

クラスタで実行する場合、下記の [Cluster](#running-in-cluster-mode)の例を参照してください。

## Okapiの利用

これらの例は、 `curl` HTTPクライアントを使用して、コマンドラインからOkapiを使用する方法を示しています。 
このドキュメントのコマンドラインにコマンドをコピーして貼り付けることができるはずです。

サービスの正確な定義は、[Reference]（＃web-service）セクションに記載されているRAMLファイルにあります。

### ストレージ
Okapiは内部のイン・メモリ・モックストレージをデフォルトにしているため、その下にデータベースレイヤーがなくても実行できます。
これは開発やテストでは問題ありませんが、実際には、あるデータをある呼び出しから次の呼び出しまで持続させることが必要です。 
現在、MongoDBとPostgreSQLストレージは、Okapiを起動するコマンドラインにそれぞれ `-Dstorage = mongo`と
` -Dstorage = postgres`オプションで有効にすることができます。

私たちはMongoのバックエンドから遠ざかっています。 
コマンドラインオプションについては、MongoHandle.javaのコードを参照する必要があります。

PostgreSQLデータベースの初期化は、2段階の操作です。 
まず、PostgreSQLでユーザとデータベースを作成する必要があります。 これは、任意のマシンで1回だけ必要です。 
Debianボックス上では次のようになるでしょう：

```
   sudo -u postgres -i
   createuser -P okapi   # When it asks for a password, enter okapi25
   createdb -O okapi okapi
```
値「okapi」、「okapi25」、および「okapi」は、開発用にのみ使用されるデフォルト値です。 
実際のプロダクションでは、適切なデータベースとそのパラメータを設定する必要があるDBAもいるでしょう。
パラメータは、コマンドラインでOkapiに渡す必要があります。

2番目のステップは、必要なテーブルとインデックスを作成することです。
Okapiはこのように呼び出されると、あなたのためにそれを行うことができます：
```
java -Dport=8600 -Dstorage=postgres -jar target/okapi-core-fat.jar initdatabase
```

このコマンドは、既存のテーブルとデータがあれば削除し、必要なものを作成して、Okapiを終了します。
既存のテーブルのみを削除する場合は、 `purgedatabase`コマンドを使用できます。

OkapiのPostgreSQLデータベースを掘り起こす必要がある場合は、次のようなコマンドで実行できます:

```
psql -U okapi postgresql://localhost:5432/okapi
```


### Curl例

次のセクションの例は、コマンドライン・コンソールに貼り付けることができます。

ソースツリーの場合のように、このガイドのこのMarkDownソースを_guide.md_としてカレントディレクトリに持っていれば、
perlの1行ですべてのサンプルレコードを抽出することもできます。
これは `/ tmp`の中に` okapi-tenant.json`のようなファイルとして保存されます。
```
perl -n -e 'print if /^cat /../^END/;' guide.md | sh
```

その後、やや複雑なコマンドですべてのサンプルを実行することもできます:

```
perl -n -e 'print if /^curl /../http/; ' guide.md |
  grep -v 8080 | grep -v DELETE |
  sh -x
```

（上記の2つのコマンドを実行する `doc / okapi-examples.sh`スクリプトを参照してください。）

これにより、DELETEコマンドのクリーンアップが明示的に省略されているため、
Okapiはいくつかの既知のテナントに対して少数のモジュールが有効になっている明確な状態になります。

### モジュール例

Okapiはすべてモジュールを呼び出すことに関するものなので、いくつか試してみる必要があります。
別のものを示す3つのダミーモジュールが付属しています。

これらはデモンストレーションとテストの目的にのみ使用されています。
実際のモジュールをこれらの上に置かないでください。

別個のリポジトリに追加のモジュールがあります
[folio-sample-modules](https://github.com/folio-org/folio-sample-modules)

#### Okapi-test-module

これは非常に単純なモジュールです。 あなたがそれにGETリクエストをすると、
それは "It works"と答えるでしょう。 
あなたが何かをPOSTすると、 "Hello"に続けてあなたがポストしたものが返されます。
リクエスト・ヘッダーをエコーするなど、いくつかのやり方も可能です。
これらはokapi-coreのテストに使用されます。

通常、Okapiがこれらのモジュールの起動と停止を行いますが、今はこれを直接実行します
-- テストの際に便利なコマンドラインHTTPクライアントであるcurlの使い方をよく見ていくために。

コンソールウィンドウを開き、okapiプロジェクトのルートに移動し、次のコマンドを実行します。

```
java -jar okapi-test-module/target/okapi-test-module-fat.jar
```

これにより、ポート8080で待機しているokapi-test-moduleが起動します。

別のコンソールウィンドウを開いて、以下のようにしてテストモジュールにアクセスしてみてください:

```
curl -w '\n' http://localhost:8080/testb
```

動作することを示すでしょう。

オプション "` -w '\ n "`はcurl出力を改行するだけです。
レスポンスが必ずしも改行で終了するわけではないからです。

ここで、テストモジュールに何かPOSTを試みます。 
実際にはこれはJSONの構造になると思われますが、今のところ単純な文字列で処理します。

```
echo "Testing Okapi" > okapi.txt
curl -w '\n' -X POST -d @okapi.txt http://localhost:8080/testb
```

再び、出力に改行を付ける-wオプションがあります。今回は POSTリクエストにする`-X POST`と
POSTしたいデータを含むファイル名を特定する` -d @ okapi.txt`を追加します。

テストモジュールは以下のレスポンスを返すはずです。

    Hello Testing Okapi

私たちのテストデータであり、 "Hello"が付加されています。

okapi-test-moduleについてはこれで十分です。 実行していたウィンドウに戻り、 `Ctrl-C`コマンドで終了します。
最初のメッセージの後に出力が生成されるべきではありません。

#### Okapi-test-header-module

`test-header`モジュールは、type = headersモジュールの使用法を示します。 
これはHTTPヘッダーを検査し、新しいHTTPヘッダーセットを生成するモジュールです。
レスポンス・ボディーは無視され、空でなければなりません。

始めに:

```
java -jar okapi-test-header-module/target/okapi-test-header-module-fat.jar
```

モジュールは先頭のパス `/ testb`から` X-my-header`を読み込みます。 
そのヘッダーが存在すれば、その値をとり、 `、foo`を付加します。 
そのようなヘッダーが存在しない場合、値 `foo`を使用します。

これらの2つのケースは次のように説明できます。

```
curl -w '\n' -D- http://localhost:8080/testb
```
そして
```
curl -w '\n' -H "X-my-header:hey" -D- http://localhost:8080/testb
```

上記のように、今すぐ簡単な検証を中止してください。

#### Okapi-test-auth-module

Okapi自体は認証を行いません。モジュールに委譲します。 
私たちは完全に機能する認証モジュールをまだ持っていませんが、
どのように動作するかを示すために使用できるダミーモジュールがあります。 
また、これは主にOkapi自体の認証メカニズムをテストするために使用されます。

ダミーモジュールは2つの機能をサポートしています： 
`/ authn / login`は、その名前が暗示するように、ユーザ名とパスワードをとるログイン機能で、
受け入れ可能な場合はHTTPヘッダ内にトークンを返します。 
他のパスは、HTTPリクエストヘッダー内に有効なトークンがあることを確認するcheck機能を経由します。

私たちは、Okapi自身をいじる際の事例を見ていきます。 
必要に応じて、okapi-test-moduleと同様にモジュールを直接検証できます。

### Okapi自身を実行する

今、私たちはOkapiを開始する準備が整いました。
注：この例を実行するには、Okapiのカレント・ディレクトリが
最上位のディレクトリ `... / okapi`であることが重要です。

```
java -jar okapi-core/target/okapi-core-fat.jar dev
```

`dev`コマンドは、開発モードで実行するよう指示します。
開発モードでは、モジュールやテナントが何も定義されていない、既知のクリーンな状態で起動します。

OkapiはPID（プロセスID）を列挙し、「API Gateway started」と述べている。 
これは、実行中であること、デフォルトポート9130でリスンしていること、イン・メモリの記憶域を使用していることを意味します。 
（代わりにPostgreSQLストレージを使用するには、 `-Dstorage = postgres`を
[command line]（＃java -d-options）に追加してください。）

Okapiが初めて起動すると、この例で使用するすべてのエンドポイントを実装する
内部モジュールのModuleDescriptorがあるかどうかを確認します。 
なければ、それは私たちのために作成されるので、我々はOkapi自体を使用することができます。 
Okapiに既知のモジュールをリストアップするように頼むことができます：

```
curl -w '\n' -D -  http://localhost:9130/_/proxy/modules

HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-1.7.1-SNAPSHOT /_/proxy/modules : 200 12710us
Content-Length: 74

[ {
  "id" : "okapi-1.7.1-SNAPSHOT",
  "name" : "okapi-1.7.1-SNAPSHOT"
} ]
```
バージョン番号は時間とともに変化します。 
この例は開発ブランチ上で実行されたので、バージョンには `-SNAPSHOT`という接尾辞があります。

すべてのOkapiの運営はテナントのために行われているため、
立ち上げ時に少なくとも1つは定義されていることをOkapiは確認します。

再び、あなたはそれを見ることができます：

```
curl -w '\n' -D - http://localhost:9130/_/proxy/tenants

HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-1.7.1-SNAPSHOT /_/proxy/tenants : 200 7080us
Content-Length: 117

[ {
  "id" : "okapi.supertenant",
  "name" : "okapi.supertenant",
  "description" : "Okapi built-in super tenant"
} ]
```

### 例1: 簡単なモジュールをデプロイして利用する

だから私たちはいくつかのモジュールで作業したいとOkapiに伝える必要があります。 
実際には、これらの操作は適切に許可された管理者によって実行されます。

前述のとおり、プロセスは3つの部分から構成されています。デプロイ、ディスカバリー、およびプロキシの構成です。

#### Deploying the test-basic module

Okapiに `okapi-test-module`を使用することを伝えるために、
moduleDescriptorのJSON構造を作成し、それをOkapiにPOSTします：

```
cat > /tmp/okapi-proxy-test-basic.1.json <<END
{
  "id": "test-basic-1.0.0",
  "name": "Okapi test module",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```
idは後でこのモジュールを参照するために使用するものです。 
バージョン番号はidに含まれているので、私たちが話しているモジュールをidが正確に特定するようになっています。 
（Okapiはこれを強制するのではなく、本当にユニークである限り、UUIDなどを使用することもできますが、
すべてのモジュールでこの命名規則を使用することに決めました。）

このモジュールは `test-basic`と呼ばれるただ1つのインタフェースを提供します。
それには、そのインターフェイスが/ testbパスに対するGETとPOSTリクエストだけに関心があることを示すハンドラが1つあります。

launchDescriptorは、このモジュールをどのように起動および停止するかをOkapiに伝えます。 
このバージョンでは、単純な `exec`コマンドラインを使用します。 
Okapiはプロセスを開始し、PIDを覚えて、完了したらただkillします。

moduleDescriptorにはもっと多くのものが含まれていますが、後の例で詳しく説明します。

それでは、ポストしましょう：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-test-basic.1.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/modules/test-basic-1.0.0
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/modules : 201 12074us
Content-Length: 350

{
  "id" : "test-basic-1.0.0",
  "name" : "Okapi test module",
  "provides" : [ {
    "id" : "test-basic",
    "version" : "2.2",
    "handlers" : [ {
      "methods" : [ "GET", "POST" ],
      "pathPattern" : "/testb"
    } ]
  } ],
  "launchDescriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
```

Okapiは「201 Created」と応答し、同じJSONを返します。
このモジュールのアドレスを示すLocationヘッダもあります。
変更または削除する場合、もしくは参照だけする場合は、次のようにします：


```
curl -w '\n' -D - http://localhost:9130/_/proxy/modules/test-basic-1.0.0
```

私たちはまた、最初にやったように、Okapiにすべての既知のモジュールをリストアップするよう依頼することもできます：

```
curl -w '\n' http://localhost:9130/_/proxy/modules
```

これは、2つのモジュールの短いリストを示しています。
内部のモジュールと、今ポストしたモジュールのリストです。

実際にはかなり長いリストになる可能性があるので、Okapiはモジュールについての詳細を提供しません。

#### Deploying the module

このようなモジュールが存在することは、Okapiが知っているだけでは不十分です。 
モジュールをデプロイする必要もあります。
ここでは、Okapiは多数のノードを持つクラスタ上で実行されることに注意する必要があります。
そのため、どのノードにデプロイするかを決定する必要があります。 
まず、作業するクラスタを確認する必要があります。

```
curl -w '\n' http://localhost:9130/_/discovery/nodes
```

Okapiはただ1つのノードの短いリストで応答します：

```
[ {
  "nodeId" : "localhost",
  "url" : "http://localhost:9130"
} ]
```

これは驚くべきことではないが、我々はすべてのものを 'dev'モードで1台のマシン上で実行しているので、
クラスタには1つのノードしかなく、デフォルトでは 'localhost'と呼ばれるます。 
これが実際のクラスタであった場合、クラスタ・マネージャは起動時にすべてのノードに対して美しくないUUIDを与えていたでしょう。 
では、そこに展開しましょう。 最初にDeploymentDescriptorを作成します。

```
cat > /tmp/okapi-deploy-test-basic.1.json <<END
{
  "srvcId": "test-basic-1.0.0",
  "nodeId": "localhost"
}
END
```
そして、それを `/ _ / discovery`にPOSTします。 そうすることができるでしょうが、
 `/ _ / deployment`にはポストしないことに注意してください。
違いは、`deployment`の場合、実際のノードにポストする必要があります。
ディスカバリはどのノードで実行されているかを知る役割を持ち、
クラスタ上の任意のオカピで利用可能であるのにです。 
プロダクションのシステムでは、ノードへの直接アクセスを防止するファイアウォールが存在する可能性があります。

```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-basic.1.json \
  http://localhost:9130/_/discovery/modules
```

Okapiは以下を応答します。

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/discovery/modules/test-basic-1.0.0/localhost-9131
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/discovery/modules : 201
Content-Length: 237

{
  "instId" : "localhost-9131",
  "srvcId" : "test-basic-1.0.0",
  "nodeId" : "localhost",
  "url" : "http://localhost:9131",
  "descriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
```

私たちがポストしたものよりも少し詳細になっています。 サービスID "test-basic-1.0.0"だけを与えて、
これを先に投稿したModuleDescriptorからLaunchDescriptorをこのIDで検索しました。

Okapiはこのモジュール用のポート9131も割り当てて、インスタンスID「localhost-9131」を与えました。 
これは、同じモジュールの複数のインスタンスを異なるノードや同じのーででさえもで実行することができるため、必要です。

最後に、OkapiはモジュールがリッスンしているURLも返します。 
現実のクラスターでは、モジュールへの直接アクセスを防ぐファイアウォールがあります。
すべてのトラフィックは認証チェック、ログなどのためにOkapiを経由しなければならないためです。
しかし、簡単なテストの例では、モジュールが実際に そのURLで実行されていることをチェックできます。 
まあ、正確にそのURLではなく、ハンドラーからのパスを上記のベースURLと組み合わせたときに得られるURLです：

```
curl -w '\n' http://localhost:9131/testb

It works
```

#### Creating a tenant

上記のように、すべてのトラフィックはモジュールに直接送信されるのではなく、
Okapiを経由する必要があります。 しかし、Okapi独自のベースURLを試してみると以下を得られます:

```
curl -D - -w '\n' http://localhost:9130/testb

HTTP/1.1 403 Forbidden
Content-Type: text/plain
Content-Length: 14

Missing Tenant
```

Okapiはマルチテナント・システムなので、各テナントに代わってそれぞれのリクエストを行う必要があります。 
スーパー・テナントを使うこともできますが、それは悪い習慣です。 
この例のテストテナントを作成しましょう。 それほど難しいことではありません：

```
cat > /tmp/okapi-tenant.json <<END
{
  "id": "testlib",
  "name": "Test Library",
  "description": "Our Own Test Library"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-tenant.json \
  http://localhost:9130/_/proxy/tenants

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/tenants : 201 1704us
Content-Length: 91

{
  "id" : "testlib",
  "name" : "Test Library",
  "description" : "Our Own Test Library"
}
```

#### Enabling the module for our tenant
次に、テナント用のモジュールを有効にする必要があります。 これはさらに簡単な操作です:

```
cat > /tmp/okapi-enable-basic-1.json <<END
{
  "id": "test-basic-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-basic-1.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-basic-1.0.0
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/tenants/testlib/modules : 201 16025us
Content-Length: 31

{
  "id" : "test-basic-1.0.0"
}
```


#### Calling the module

ですから、現在はテナントがあり、モジュールが有効になっています。 
私たちが最後にモジュールを呼び出そうとしたとき、Okapiは "Missing tenant"と答えました。
エクストラなヘッダーとして、テナントを呼び出しに追加する必要があります：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  http://localhost:9130/testb

HTTP/1.1 200 OK
Content-Type: text/plain
X-Okapi-Trace: GET test-basic-1.0.0 http://localhost:9131/testb : 200 5632us
Transfer-Encoding: chunked

It works
```

#### Another way

次のように、特定のテナントのモジュールを呼び出す別の方法があります：

```
curl -w '\n' -D - \
  http://localhost:9130/_/invoke/tenant/testlib/testb

It works!
```

これは、SSOサービスからのコールバックなど、リクエストのヘッダーを制御できない特殊なケースでは、ちょっとしたハックです。
これは非常に制限されており、認証トークンを必要とするコールでは失敗します（下記参照）。
このような場合には、後で `/ _ / invoke / token / xxxxxxx / .... ` にパスを追加することができます。

invokeエンドポイントがOkapi 1.7.0で追加されました。


### 例2:Authモジュールを追加する

前の例は、テナントIDを推測することができまする人には誰にでも有効です。 
これは小さなテストモジュールでは問題ありませんが、実際のモジュールが実際に動作する場合、
特権ユーザーに限定する必要があります。 
実際には、すべての種類の認証と認可を管理する複雑なモジュールセットが必要になりますが、
この例では、Okapi独自のtest-authモジュールのみを使用しています。 
それはどんな深刻な認証もしませんが、それを使用する方法を実証するのにちょうど十分でしょう。

前と同じように、我々が作成する最初のものはModuleDescriptorです：

```
cat > /tmp/okapi-module-auth.json <<END
{
  "id": "test-auth-3.4.1",
  "name": "Okapi test auth module",
  "provides": [
    {
      "id": "test-auth",
      "version": "3.4",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/authn/login"
        }
      ]
    }
  ],
  "filters": [
    {
      "methods": [ "*" ],
      "pathPattern": "/*",
      "phase": "auth",
      "type": "headers"
    }
  ]
}
END
```

モジュールには、 `/ authn / login`パス用のハンドラが1つあります。 
また、すべてのやってくるリクエストに接続するフィルターもあります。 
それは、ユーザーがリクエストを許可されるかどうかを決定するところです。 
これには「ヘッダー」というタイプがあります。
これは、Okapuがリクエストのすべてではなく、ヘッダーだけを渡すことを意味します。

フィルタのpathPatternはワイルドカード文字（ `*`）を使用して任意のパスにマッチさせます。 
pathPatternには、パス・コンポーネントに一致させるための中括弧のペアも含まれます。
例えば `/users/{id}`は  `/users/abc`には一致しますが、`/users/abc/d`には一致しません。

フェーズは、フィルタが適用されるステージを指定します。 
この時点では、ハンドラの前に呼び出される1つのフェーズ「auth」しかありません。 
必要に応じて、ハンドラの前後に、さまざまなフェーズが登場する可能性があります。

以前と同じようにlaunchDescriptorを含めることができましたが、別の方法を示すためにここでは省略しました。 
このようにすると、各モジュールインスタンスが余分にコマンドライン引数や環境変数を必要とするかもしれない、
クラスタ化された環境では、より意味をなすようになるかもしれません。

私たちはOkapiにそれをPOSTします：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-module-auth.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/modules/test-auth-3.4.1
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/modules : 201 5634us
Content-Length: 357

{
  "id" : "test-auth-3.4.1",
  "name" : "Okapi test auth module",
  "provides" : [ {
    "id" : "test-auth",
    "version" : "3.4",
    "handlers" : [ {
      "methods" : [ "POST" ],
      "pathPattern" : "/authn/login"
    } ]
  } ],
  "filters" : [ {
    "methods" : [ "*" ],
    "pathPattern" : "/*",
    "phase" : "auth",
    "type" : "headers"
  } ]
}
```

次に、モジュールをデプロイする必要があります。 
私たちはlaunchDescriptorをmoduleDescriptorに入れなかったので、ここでひとつ提供する必要があります。

```
cat > /tmp/okapi-deploy-test-auth.json <<END
{
  "srvcId": "test-auth-3.4.1",
  "nodeId": "localhost",
  "descriptor": {
    "exec": "java -Dport=%p -jar okapi-test-auth-module/target/okapi-test-auth-module-fat.jar"
  }
}
END

curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-auth.json \
  http://localhost:9130/_/discovery/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/discovery/modules/test-auth-3.4.1/localhost-9132
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/discovery/modules : 201
Content-Length: 246

{
  "instId" : "localhost-9132",
  "srvcId" : "test-auth-3.4.1",
  "nodeId" : "localhost",
  "url" : "http://localhost:9132",
  "descriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-auth-module/target/okapi-test-auth-module-fat.jar"
  }
}
```

そして私たちはテナントのためにモジュールを有効にします：

```
cat > /tmp/okapi-enable-auth.json <<END
{
  "id": "test-auth-3.4.1"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-auth.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-auth-3.4.1
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/tenants/testlib/modules : 201 1727us
Content-Length: 30

{
  "id" : "test-auth-3.4.1"
}
```

ですから、認証モジュールはOkapiに行ったすべての呼び出しを傍受し、
それが許可されているかどうかを確認する必要があります。
前と同様の基本モジュールへの同じ呼び出しを試してみましょう：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  http://localhost:9130/testb

HTTP/1.1 401 Unauthorized
Content-Type: text/plain
X-Okapi-Trace: GET test-auth-3.4.1 http://localhost:9132/testb : 401 84118us
Transfer-Encoding: chunked

Auth.check called without X-Okapi-Token
```

実際、私たちはもはやテストモジュールを呼び出すことはできません。 
そうなると、どのように許可を得るのでしょう？ 
エラーメッセージは、 `X-Okapi-Token`が必要であることを示しています。
それはログインサービスから得ることができるものです。
ダミーの認証モジュールは、それほど賢くはパスワードを確認できません、
ユーザー名 "peter"にはパスワード "peter-password"があると仮定しています。 
非常に安全というわけではありませんが、この例では十分です。


```
cat > /tmp/okapi-login.json <<END
{
  "tenant": "testlib",
  "username": "peter",
  "password": "peter-password"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -d @/tmp/okapi-login.json \
  http://localhost:9130/authn/login

HTTP/1.1 200 OK
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/authn/login : 202 7235us
Content-Type: application/json
X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/authn/login : 200 232721us
Transfer-Encoding: chunked

{  "tenant": "testlib",  "username": "peter",  "password": "peter-password"}
```

レスポンスはただそのパラメータをエコーするだけですが、
ヘッダー `X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig`
を返すことに注意してください。
ヘッダーの内容については本来心配するものではありませんが、
JWTに期待するものとほぼ同じフォーマットになっています：
ドットで区切られた3つの部分、最初はヘッダー、次にbase-64にエンコードされたペイロード、最後に署名。
ヘッダーと署名も通常base-64でエンコードされますが、
単純なtest-authモジュールはその部分をスキップして、本当のJWTと誤解されない別個のトークンを作成します。
ペイロードは実際にbase-64でエンコードされています。
デコードすると、ユーザーIDとテナントIDを持つJSON構造が含まれており、
他に含まれているものは大してない、ということが分かります。
実際の認証モジュールはもちろん、JWTにもっと多くのものを入れ、強力な暗号で署名します。
しかし、それはOkapiのユーザーには何の差もないはずです。
彼らが知る必要があるのは、どのようにトークンを取得するのか、またどのように各リクエストにトークンを渡すのかです。
このように：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  http://localhost:9130/testb

HTTP/1.1 200 OK
X-Okapi-Trace: GET test-auth-3.4.1 http://localhost:9132/testb : 202 18179us
Content-Type: text/plain
X-Okapi-Trace: GET test-basic-1.0.0 http://localhost:9131/testb : 200 3172us
Transfer-Encoding: chunked

It works
```

私たちはテストモジュールにポストすることもできます：
```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -d '{ "foo":"bar"}' \
  http://localhost:9130/testb
```
モジュールは同じJSONで応答しますが、文字列には "Hello"が付加されます。

### 例3:アップグレード、バージョン、環境、`_tenant`インタフェース

アップグレードはしばしば問題になることがあります。 
Okapiではなおさらです。
私たちは多くのテナントにサービスを提供しており、
テナントはいつ、何をアップグレードするのかについて異なる考えを持っているからです。
この例では、アップグレード・プロセスを説明し、バージョンと環境変数について議論し、
特別な `_tenant`システム・インタフェースを見ていきます。

例えば、新しい改良されたサンプルモジュールがあるとしましょう：
```
cat > /tmp/okapi-proxy-test-basic.2.json <<END
{
  "id": "test-basic-1.2.0",
  "name": "Okapi test module, improved",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.4",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    },
    {
      "id": "_tenant",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenant"
        }
      ]
    }
  ],
  "requires": [
    {
      "id": "test-auth",
      "version": "3.1"
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar",
    "env": [
      {
        "name": "helloGreeting",
        "value": "Hi there"
      }
    ]
  }
}
END
```

同じ名前で、別のIDを与えますが、より高いバージョン番号を付けることに注意してください。 
更に、この例では、同じokapi-test-moduleプログラムを使用していることに注意してください。
というのも、これは他に大していじれるものがないからです。
ここにあるように、もしmodule descriptorのみに変更を加えた場合、これは実際に起こる可能性があります。

モジュールがサポートする新しいインタフェースを追加しました： `_tenant`。 
これは、モジュールがテナント用に有効になったときにOkapiが自動的に呼び出すシステムインターフェイスです。 
その目的は、データベースのテーブルの作成など、モジュールが必要とする初期化を行うことです。

また、少なくともバージョン3.1ではこのモジュールにtest-authインターフェイスが必要であることを指定しました。 
前述の例でインストールしたauthモジュールは3.4であるため、十分です。 
（3.5または4.0、または2.0を要求することはできません。
[_Versioning and Dependencies_](#versioning-and-dependencies) を参照するか、
ディスクリプタを編集してポストしてください）。

最後に、異なるグリーティングを指定する起動ディスクリプタに環境変数を追加しました。 
モジュールは、POSTリクエストを処理するときにそれを報告する必要があります。

アップグレード・プロセスは、モジュールの場合と同様に、新しいmodule descriptorをポストすることから始まります。
一部のテナントがそれを使用している可能性があるため、古いものに触れることはできません。

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-test-basic.2.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created   ...
```
次に、前と同じようにモジュールをデプロイします。

```
cat > /tmp/okapi-deploy-test-basic.2.json <<END
{
  "srvcId": "test-basic-1.2.0",
  "nodeId": "localhost"
}
END

curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-basic.2.json  \
  http://localhost:9130/_/discovery/modules
```

Now we have both modules installed and running. Our tenant is still using the
old one. Let's change that, by enabling the new one instead of the old one.
This is done with a POST request to the URL of the current module.

これで、モジュールがインストールされ実行されています。 私たちのテナントはまだ古いものを使用しています。 
古いものの代わりに新しいものを有効にすることで、それを変更してみましょう。
これは、現在のモジュールのURLに対するPOST要求で行われます。

```
cat > /tmp/okapi-enable-basic-2.json <<END
{
  "id": "test-basic-1.2.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-basic-2.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules/test-basic-1.0.0

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-basic-1.2.0
X-Okapi-Trace: POST okapi-1.7.1-SNAPSHOT /_/proxy/tenants/testlib/modules/test-basic-1.0.0 : 201
Content-Length: 31

{
  "id" : "test-basic-1.2.0"
}
```

新しいモジュールがテナントに使用可能になり、古いモジュール使用不可能になります、次のように：
```
curl -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules
```

Okapiのログを見ると、次のような行があることがわかります：
```
15:32:40 INFO  MainVerticle         POST request to okapi-test-module tenant service for tenant testlib
```

It shows that our test module did get a request to the tenant interface.

In order to verify that we really are using the new module, let's post a thing
to it:

私たちのテストモジュールがテナント・インターフェースへのリクエストを得たことを示しています。

新しいモジュールを本当に使用しているかを確認するために、それにポストしてみましょう：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -d '{ "foo":"bar"}' \
  http://localhost:9130/testb

HTTP/1.1 200 OK
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/testb : 202 4325us
Content-Type: text/plain
X-Okapi-Trace: POST test-basic-1.2.0 http://localhost:9133/testb : 200 3141us
Transfer-Encoding: chunked

Hi there { "foo":"bar"}
```

確かに、 "Hello"ではなく "Hi there"と表示され、X-Okapi-Traceは要求がモジュールの改良版に送られたことを示しています。

### 例4:ModuleDescriptorを完成させる

この例では、完全なModuleDescriptorを色々とオプションをつけて表示します。 
今までに、これを使う方法を知っているはずですので、 `curl`コマンドをすべて繰り返す必要はありません。

```javascript
{
  "id": "test-basic-1.3.0",
  "name": "Bells and Whistles",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.4",
      "handlers": [
        {
          "methods": [ "GET" ],
          "pathPattern": "/testb",
          "permissionsRequired": [ "test-basic.get.list" ]
        },
        {
          "methods": [ "GET" ],
          "pathPattern": "/testb/{id}",
          "permissionsRequired": [ "test-basic.get.details" ],
          "permissionsDesired": [ "test-basic.get.sensitive.details" ],
          "modulePermissions": [ "config.lookup" ]
        },
        {
          "methods": [ "POST", "PUT" ],
          "pathPattern": "/testb",
          "permissionsRequired": [ "test-basic.update" ],
          "modulePermissions": [ "config.lookup" ]
        }
      ]
    },
    {
      "id": "_tenant",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenant"
        }
      ]
    },
    {
      "id": "_tenantPermissions",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenantpermissions"
        }
      ]
    }
  ],
  "requires": [
    {
      "id": "test-auth",
      "version": "3.1"
    }
  ],
  "permissionSets": [
    {
      "permissionName": "test-basic.get.list",
      "displayName": "test-basic list records",
      "description": "Get a list of records"
    },
    {
      "permissionName": "test-basic.get.details",
      "displayName": "test-basic get record",
      "description": "Get a record, except sensitive stuff"
    },
    {
      "permissionName": "test-basic.get.sensitive.details",
      "displayName": "test-basic get whole record",
      "description": "Get a record, including all sensitive stuff"
    },
    {
      "permissionName": "test-basic.update",
      "displayName": "test-basic update record",
      "description": "Update or create a record, including all sensitive stuff"
    },
    {
      "permissionName": "test-basic.view",
      "displayName": "test-basic list and view records",
      "description": "See everything, except the sensitive stuff",
      "subPermissions": [
        "test-basic.get.list",
        "test-basic.get.details"
      ]
    },
    {
      "permissionName": "test-basic.modify",
      "displayName": "test-basic modify data",
      "description": "See, Update or create a record, including sensitive stuff",
      "subPermissions": [
        "test-basic.view",
        "test-basic.update",
        " test-basic.get.sensitive.details"
      ]
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar",
    "env": [
      {
        "name": "helloGreeting",
        "value": "Hi there"
      }
    ]
  }
}
```

この時点でほとんどのディスクリプタはかなりよく分かるようになったはずです。
大きな新しいことはpermissionについてです。
完全な [permission system](security.md) は別のドキュメントで説明され、
複合認証モジュールシステムによって管理されます。
そのすべてはOkapi自身とこのガイドの範囲外です。

このdescriptorで最も目に見える新しいことは、permissionSetsと呼ばれる新しいセクションのすべてです。 
これは、このモジュールが気にするpermissionとpermissionsセットを定義します。
用語のポイント： "Permissions"または "Permission Bits"は "test-basic.get.list"のような単純な文字列です。
Okapiと認証モジュールは、この粒度で動作します。
「パーミッションセット」は、複数のパーミッションビット、さらには他のセットを含むことができる名前の付いたセットです。 
これらはauthモジュールによって実際のビットに縮小されます。 
これらは、管理ユーザーが通常見ている最も低いレベルであり、
より複雑なロールを構築するためのビルディングブロックを形成します。

permission bitsは、ハンドラ・エントリで使用されます。 
最初のものにはpermission "test-basic.get.list"を含むpermissionsRequiredフィールドがあります。 
つまり、ユーザーにそのようなpermissionがない場合、authモジュールはOkapiに通知し、リクエストを拒否します。

次のエントリにもpermissionsRequiredがありますが、
その中に "test-basic.get.sensitive.details"があるpermissionsDesiredフィールドもあります。 
これは、モジュールがユーザーにそのようなpermissionがあるかどうかを知りたいということを示します。
それは難しい要件ではなく、リクエストはいずれにせよモジュールに渡されますが、
X-Okapi-Permissionsヘッダにはそのパーミッション名が含まれていたり含まれていなかったりします。 
それで何をすべきかを決めるのはモジュールに任されています。
この場合、非常にセンシティブなフィールドを表示したり非表示にしたりすることができます。

また、 "config.lookup"という値を持つ第3のフィールド "modulePermissions"もあります。 
これは私たちのモジュールにこのpermissionが与えられたことを示します。 
ユーザーがそのようなpermissionを持っていなくても、モジュールは実行します。
したがって、構成設定を調べるようなことをすることができます。

上記のように、Okapiは未加工のpermission bitsで動作します。 
それは認証モジュールに必要なpermissionを渡しました。
認証モジュールは、ユーザーはそのアクセス権を持っているかどうかを何とか推測します。 
詳細はここでは重要ではありませんが、プロセスがpermissionSetsと関係があることは明らかです。
authモジュールはmoduleDescriptionのパーミッション・セットにどのようにアクセスするのでしょうか？
それは魔法によって起こるのではありませんが、そのようなものです。 
モジュールがテナントに対して有効になると、Okapiはモジュール自身の `_tenant`インタフェースを呼び出すだけでなく、
モジュールがtenantPermissionsインタフェースを提供しているかどうかを確認し、そこにpermissionSetsを渡します。
permissionモジュールはそれを行うべきもので、そのようにpermissionSetを受け取ります。

この例でも、ModuleDescriptorのすべての可能性をカバーしているわけではありません。 
まだサポートされている古いバージョン（Okapi 1.2.0より古い）には残っている機能がありますが、
将来のバージョンでは廃止され、削除されます。
たとえば、descriptorの最上位にあるModulePermissionsおよびRoutingEntriesです。 
完全な最新の定義については、[Reference](#web-service)）セクションのRAMLとJSONのスキーマを常に参照してください。


### 複数のインタフェース

通常、Okapiプロキシは所定のインタフェースを提供するのにただ一つのモジュールを許可します。
`provide`セクションで`interfaceType` `multiple`を使うことで、任意の数のモジュールが同じインターフェースを実装することができます。
しかし、結局のところ、インタフェースのユーザは、
HTTPヘッダー`X-Okapi-Module-Id`を指定し、呼び出すモジュールを選択する必要があります。
Okapiは、テナント( `_/proxy/tenants/{tenant}/interfaces/{interface}` )に対して
与えられたインタフェースを実装するモジュールのリストを返すファシリティを提供します。
通常、テナントは「カレント」テナント（ヘッダー `X-Okapi-Tenant`）と同じです。

この例を見てみましょう。
同じインターフェースを実装する2つのモジュールを定義し、そのうちの1つを呼び出します。
前の例のテナントtestlibとauthモジュールがまだ存在すると仮定します。
前に使用したテストモジュールのModule Descriptorを定義しようとしましょう。
以下のModuleDescriptorは `interfaceType`を` multiple`に設定しているので、
Okapiは`test-multi`インターフェースの複数のモジュールを共存させることができます。

```
cat > /tmp/okapi-proxy-foo.json <<END
{
  "id": "test-foo-1.0.0",
  "name": "Okapi module foo",
  "provides": [
    {
      "id": "test-multi",
      "interfaceType": "multiple",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```
`foo`を登録してデプロイする：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-foo.json \
  http://localhost:9130/_/proxy/modules
```

```
cat > /tmp/okapi-deploy-foo.json <<END
{
  "srvcId": "test-foo-1.0.0",
  "nodeId": "localhost"
}
END
```

```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-foo.json \
  http://localhost:9130/_/discovery/modules
```


ここで別のモジュール `bar`を定義します:

```
cat > /tmp/okapi-proxy-bar.json <<END
{
  "id": "test-bar-1.0.0",
  "name": "Okapi module bar",
  "provides": [
    {
      "id": "test-multi",
      "interfaceType": "multiple",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```
`bar`を登録してデプロイする：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-bar.json \
  http://localhost:9130/_/proxy/modules

```

```
cat > /tmp/okapi-deploy-bar.json <<END
{
  "srvcId": "test-bar-1.0.0",
  "nodeId": "localhost"
}
END
```
```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-bar.json \
  http://localhost:9130/_/discovery/modules
```

そして今、テナント `testlib`の` foo`と `bar`の両方のモジュールを有効にします：

```
cat > /tmp/okapi-enable-foo.json <<END
{
  "id": "test-foo-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-foo.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules
```

```
cat > /tmp/okapi-enable-bar.json <<END
{
  "id": "test-bar-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-bar.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules
```

私たちは、どのモジュールが`test-multi`インタフェースを実装しているのかをOkapiに尋ねることができます：


```
curl -w '\n' -D - \
  http://localhost:9130/_/proxy/tenants/testlib/interfaces/test-multi

HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-1.7.1-SNAPSHOT /_/proxy/tenants/testlib/interfaces/test-multi : 200 1496us
Content-Length: 64

[ {
  "id" : "test-foo-1.0.0"
}, {
  "id" : "test-bar-1.0.0"
} ]
```

モジュール `bar`を呼び出してみましょう：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -H "X-Okapi-Module-Id: test-bar-1.0.0" \
  http://localhost:9130/testb

It works
```


### クリーンアップ

例は終わりです。 うまくいくように、インストールしたものはすべて削除します：

```
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-basic-1.2.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-auth-3.4.1
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-foo-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-bar-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-auth-3.4.1/localhost-9132
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-basic-1.0.0/localhost-9131
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-basic-1.2.0/localhost-9133
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-foo-1.0.0/localhost-9134
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-bar-1.0.0/localhost-9135
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-basic-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-basic-1.2.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-foo-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-bar-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-auth-3.4.1
```

Okapiはこれらのそれぞれにシンプルに応答します：

```
HTTP/1.1 204 No Content
Content-Type: text/plain
Content-Length: 0
```
<!-- STOP-EXAMPLE-RUN -->
最後に、実行していたOkapiインスタンスを単純な `Ctrl-C`コマンドで停止することができます。

### クラスタモードで実行

今のところすべての例は、単一のマシン上で `dev`モードで実行されています。 
これはのデモンストレーション、開発などには適していますが、
本物のプロダクション設定では、マシンのクラスタ上で実行する必要があります。

#### 単一のマシン上で

最も簡単なクラスタ設定は、同じマシン上でOkapiの複数のインスタンスを実行することです。 
これは行われるべきやり方ではありませんが、デモをするのに最も簡単な方法です。

コンソールを開き、最初のOkapiを起動する
```
java -jar okapi-core/target/okapi-core-fat.jar cluster
```

Okapiは `dev`モードよりも多くの起動メッセージを表示します。 
面白いメッセージ行には次のようなものがあります

```
Hazelcast 3.6.3 (20160527 - 08b28c3) starting at Address[172.17.42.1]:5701
```

これは、vert.xがクラスタリングに使用するHazelcastを使用しており、
アドレス172.17.42.1上でポート5701を使用していることを示しています。 
ポートはデフォルトですが、Hazelcastは空きポートを見つけようとするため、
別のポートで終了する可能性があります。
アドレスは、そのインターフェースの1つにあるマシンのアドレスです。 
それについては後で詳しく説明します。

別のコンソールを開き、Okapiの別のインスタンスを起動します。
同じマシン上にあるため、両方のインスタンスが同じポートでリスンすることはできません。
デフォルトでは、Okapiはモジュール用に20個のポートを割り当てていますので、次のOkapiをポート9150から始めましょう：
```
java -Dport=9150 -jar okapi-core/target/okapi-core-fat.jar cluster
```

再びOkapiはいくつかの起動メッセージを表示しますが、最初のOkapiもいくつか表示しています。
それらの2つは接続してお互いに話しています。

これで、Okapiに既知のノードのリストを要求することができます。
3番目のコンソールウィンドウでこれを試してください：

```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 186

[ {
  "nodeId" : "0d3f8e19-84e3-43e7-8552-fc151cf5abfc",
  "url" : "http://localhost:9150"
}, {
  "nodeId" : "6f8053e1-bc55-48b4-87ef-932ad370081b",
  "url" : "http://localhost:9130"
} ]
```

実際には、2つのノードがリストされます。 
彼らはそれぞれ、到達可能なURLと、ランダムなUUID文字列であるnodeIdを持っています。

あなたのURLのポートを9150に変更することによって、
他のノードにすべてのノードをリストするように頼むことができ、
おそらく異なる順序で同じリストを取得するでしょう。


#### 複数のマシン上で

もちろん、クラスタを複数のマシンで実行したいでしょう。つまりクラスタリングの核心です。

*警告* Okapiは、クラスタ設定を管理するためにHazelcastライブラリを使用し、
クラスタ内のノードを検出するためにマルチキャスト・パケットを使用します。 
マルチキャストはほとんどの有線イーサネット上で正常に動作しますが、
無線ではあまりうまく動作しません。 プロダクションのクラスターでは、
ワイヤレスネットワークを使用するべきではありませんが、
開発者はワイヤレスネットワーク上のラップトップで試してみたくなるでしょう。 
そ　れ　は　う　ま　く　い　き　ま　せ　ん　！
あなたのクラスタに参加するすべてのIPアドレスを一覧表示するhazelcast-config-fileを含む回避策がありますが、
それは面倒です。ここでは詳細には触れません。

2つの細かい点を除いて、手順はほぼ同じです。  最初は、別々のマシン上にあり、衝突しないため、
異なるポートを指定する必要はありません。 
代わりに、マシンが同じネットワーク上にあることを確認する必要があります。
最新のLinuxマシンには、複数のネットワークインタフェース、
通常は少なくともイーサネットケーブルとループバックインタフェースがあります。 
かなり多くの場合にWi-Fiポートでもあり、Dockerを使用する場合は独自の内部ネットワークを設定します。 
`sudo ifconfig`でリストアップされたすべてのインタフェースを見ることができます。 
Okapiは、どのインタフェースを使用する必要があるかを推測するほど賢くはないないので、
しばしばそれを伝えなければなりません。 
それには、次のような方法があります：

```
java -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```
cluster-hostオプションは、コマンドラインの最後になければならないことに注意してください。 パラメータ名は少し誤解を招きます。ホスト名ではなく、IPアドレスが必要です。

同じネットワーク上にある2台目のマシンでOkapiを起動します。 
適切なIPアドレスをコマンドラインで使用するように注意してください。 
すべてがうまくいけば、マシンはお互いに見えるはずです。
両方のマシンのログに表示されます。
これで、Okapi（クラスタ内の任意のOkapi）にすべてのノードをリストするように問い合わせることができます：

```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 186

[ {
  "nodeId" : "81d1d7ca-8ff1-47a0-84af-78cfe1d05ec2",
  "url" : "http://localhost:9130"
}, {
  "nodeId" : "ec08b65d-f7b1-4e78-925b-0da18af49029",
  "url" : "http://localhost:9130"
} ]
```

ノードのUUIDは異なるがURLが同じであることに注意してください。 
どちらも `http：// localhost：9130`で到達可能であると主張しています。 
非常に技術的な意味で、ノード自体にcurlを使用すると、localhost：9130がOkapiを指しています。 
しかし、あなた（または他のOkapi）がネットワーク上の他の場所からノードに向かって話したがっていると、
これはあまり実用的ではありません。 
解決策は、コマンドラインに別のパラメータを追加して、Okapiがそれ自身のために返すべきホスト名を伝えることです。

Okapisを停止し、次のようなコマンドラインでそれらを再起動してください：
```
java -Dhost=tapas -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```
「タパス」の代わりに、あなたが始めているマシンの名前、あるいはIPアドレスを使ってください。 
再度、ノードをリストします。

```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 178

[ {
  "nodeId" : "40be7787-657a-47e4-bdbf-2582a83b172a",
  "url" : "http://jamon:9130"
}, {
  "nodeId" : "953b7a2a-94e9-4770-bdc0-d0b163861e6a",
  "url" : "http://tapas:9130"
} ]
```

URLが正常に動作していることを確認できます:

```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 178

[ {
  "nodeId" : "40be7787-657a-47e4-bdbf-2582a83b172a",
  "url" : "http://jamon:9130"
}, {
  "nodeId" : "953b7a2a-94e9-4770-bdc0-d0b163861e6a",
  "url" : "http://tapas:9130"
} ]
```
#### ノードの命名

前述したように、HazelcastシステムはnodeIdsにUUIDを割り当てます。 
それはすべて問題ありませんが、使用するのは面倒です。
実行するたびに変更されるので、スクリプトなどのノードを参照するのは簡単ではありません。
コマンドにノード名を付ける機能が追加されました このように：
```
java -Dhost=tapas -Dnodename=MyFirstNode \
  -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```

ノードをリストすると、次のように表示されます：
```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-1.11.1-SNAPSHOT /_/discovery/nodes : 200
Content-Length: 120

[ {
  "nodeId" : "7d6dc0e7-c163-4bbd-ab48-a5d7fa6c4ce4",
  "url" : "http://tapas:9130",
  "nodeName" : "MyFirstNode"
} ]
```

nodeIdの代わりに、多くの場所でその名前を使うことができます。例えば
```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes/myFirstNode```



#### So, you have a cluster
Okapiクラスタは、あなたが以前に見た単一のOkapiとして機能します。 
ほとんどの目的のために、どのノードに話をしても関係なく、すべての情報を共有します。
あるノードを介してモジュールを作成し、別のノードを介してテナントを作成し、
最初のノードを介してそのテナントのモジュールを再度有効にすることができます。
実行するノードを制御する必要があるので、モジュールを配備する際には少し注意する必要があります。
上記の単一ノードの例では、展開するnodeIdとして 'localhost'だけを使用していましたが、
これまで何度も見たUUIDを渡す必要があります。
モジュールをデプロイした後は、あなたが好きなOkapiを使ってトラフィックをプロキシすることができます。

あなたが知っておくべきいくつかの小さな違いがあります：
  *インメモリーバックエンドは、クラスター内のすべてのノード間で共有されます。 
つまり、1つのノードを停止してもう一度起動すると、他のノードと同期され、共有データが認識されます。 
実行中のすべてのOkapiが終了した場合にのみ、データはメモリから消えます。 
もちろん、Postgresのバックエンドを使用するとデータが保持されます。
  * `/ _ / deployment`エンドポイントを使用してモジュールをデプロイできます。 
これは、実行するノード上で実行する必要があります。 Okapiはそれについて他のノードに通知します。 
通常、 `/ _ / discovery`エンドポイントを通して展開し、nodeIdを指定する必要があります。
  * Okapiの起動には少し時間がかかることがあります。

使用できるクラスタリングモードはさらに2つあります。 
`配備`は、Okapiをクラスタモードで起動しますが、プロキシを実行することはありません。 
1つのノードだけが外部から見え、残りのノードはデプロイメントモードで実行できるクラスタでは便利です。

もう一方のモード、 `proxy`はそれと逆です。 
すべての要求を受け取り、それらを 'デプロイメント'ノードに渡すOkapiノードのためのものです。 
これは通常、外部から見えるノードです。 この種の分割は、プロキシだけでノードが完全に
占有されるようなトラフィックが多い場合には意味をなさないようになります。

### Okapiのセキュリティ

上記の例では、Okapiにコマンドを実行しただけで、何のチェックもせずにモジュールを喜んで配備して有効にしました。
本番システムでは、これは受け入れられません。 
Okapiは簡単にセキュアになるよう設計されています。 
実際、チェックなしでOkapiを使用できるようにするには、いくつかの小さなハッキングがあります。 
たとえば、何も指定されていない場合、Okapiはデフォルトで`okapi.supertenant`に設定され、
このテナントはデフォルトで内部モジュールが有効になっています。

原則として、Okapi自体のセキュリティ確保は、モジュールへのアクセスを確保するのと同じ方法で行われます。
`okapi.supertenant`の認証チェックフィルタをインストールします。 それは認証されていない人を通しません。
authサンプルモジュールは少し単純ですが、実際には、さまざまなユーザーなどに異なるアクセス権を処理できるシステムが必要です。

Okapiのセキュリティ確保の詳細は、使用する認証モジュールの性質によって異なります。
いずれにしても、私たちはすべてをセットアップする前に自分自身をロックしないように注意する必要があります。
そうすればまたまた入ることができます。 次の行に沿った何か：

 * Okapiが起動すると、internal module、superpertenantが作成され、
 テナント用のモジュールが有効になります。 すべての操作はチェックなしで可能です。
 * 管理者は、必要な認証モジュールをインストールしてデプロイします。
 * 管理者は、認証モジュールの保存先を有効にします。
 * 管理者は、適切な認証情報とアクセス許可を認証モジュールに送信します。
 * 管理者がauth-checkフィルタを有効にします。 今は何も許されません。
 * 管理者は以前に読み込まれた資格情報でログインし、トークンを取得します。
 * このトークンは、Okapiでのさらなる操作を行う管理者権限を与えます。


ModuleDescriptorは、その関数へのきめ細かなアクセス制御に適した権限を定義します。
現時点では、ほとんどの読み取り専用操作は誰にでも開かれていますが、
変更を加える操作には許可が必要です。


通常のクライアントがOkapiの管理機能にアクセスする必要がある場合
（たとえば、使用可能なモジュールの一覧を表示する場合など）、
内部モジュールを利用できるようにする必要があります。
もし必要であれば、アドミンユーザーにパーミッションを与えます。

### モジュール記述子（ディスクリプタ）の共有
Okapiのインストールは、モジュール記述子（ディスクリプタ）を共有することができます。 
「プル」操作では、Okapiプロキシのモジュールを別のOkapiプロキシインスタンスからフェッチできます。
「プル」という名前は、Git SCMのプル操作に似ているため、ここで使用されています。 
Okapiがプルするリモートプロキシインスタンス（またはピア）には、モジュールのデプロイを必要としません。
必要なのは、 `/ _ / proxy / modules`操作が利用可能であることだけです。
「プル」は、まだ使用できないすべてのモジュール記述子をリモートからインストールします。 
これは、モジュールの一意の実装を示すはずのモジュール記述子IDに基づいています。

プル操作の場合、Okapiはプル・ディスクリプタを取ります。
この段階では、リモートインスタンスのURLが含まれます。 
Okapiの将来のバージョンでは、認証などのために、プル・ディスクリプタにさらに情報を含めることができます。 
ローカルのOkapiインスタンスのために呼び出されるパスは `/ _ / proxy / pull / modules`です。

プル操作は、フェッチされてローカルにインストールされたモジュールの配列を返します。

#### プル操作例

この例では、2回「プル」します。 2番目のプルは、すべての/ほとんどのモジュールがすでにフェッチされているため、
1回目のプルよりもはるかに高速でなければなりません
```
cat > /tmp/pull.json <<END
{"urls" : [ "http://folio-registry.aws.indexdata.com:9130" ]}
END

curl -w '\n' -X POST -d@/tmp/pull.json http://localhost:9130/_/proxy/pull/modules
curl -w '\n' -X POST -d@/tmp/pull.json http://localhost:9130/_/proxy/pull/modules
```

### テナントごとにモジュールをインストールする

これまでは （このガイドでは）、一度に少数のモジュールしかインストールしていなかったので、
依存関係を追跡して順番どおりに実行できるようになりました。 
例えば、'test-basic'が要求した 'test-auth'インターフェースは、私たちが 'test-auth'モジュールによって提供されたと分かっていました。
それらの名前がこのように一致するのは偶然です。 
モジュールが多くのインタフェースを必要とすることは言うまでもありません。


Okapi 1.10以降では、状況を改善するために `/ _ / proxy / tenant / id / install`呼び出しを提供しています。 
この呼び出しは、1つまたは複数のモジュールを有効/アップグレード/無効にし、依存関係を考慮した同様のリストで応答します。 
詳細は、JSONスキーマTenantModuleDescriptorListとRAMLの一般的な定義を参照してください。

#### インストール操作の例

Suppose we have pulled module descriptors from the remote repo
(e.g. using [プル操作例](#プル操作例) above)
and now would like to enable `mod-users-bl-2.0.1`
for our tenant.

リモートrepoからモジュール記述子を取得したと仮定して（e.g. 上記の[プル操作例](#プル操作例)を使用）、
テナントに `mod-users-bl-2.0.1`を有効にしたいとします。
```
cat > /tmp/okapi-tenant.json <<END
{
  "id": "testlib",
  "name": "Test Library",
  "description": "Our Own Test Library"
}
END
curl -w '\n' -X POST -D - \
  -d @/tmp/okapi-tenant.json \
  http://localhost:9130/_/proxy/tenants

cat >/tmp/tmdl.json <<END
[ { "id" : "mod-users-bl-2.0.1" , "action" : "enable" } ]
END
curl -w '\n' -X POST -d@/tmp/tmdl.json \
 http://localhost:9130/_/proxy/tenants/testlib/install?simulate=true

[ {
  "id" : "mod-users-14.2.1-SNAPSHOT.299",
  "action" : "enable"
}, {
  "id" : "permissions-module-4.0.4",
  "action" : "enable"
}, {
  "id" : "mod-login-3.1.1-SNAPSHOT.42",
  "action" : "enable"
}, {
  "id" : "mod-users-bl-2.0.1",
  "action" : "enable"
} ]
```

4モジュールのセットが要求されました。 
もちろん、このリストは、リモートリポジトリのモジュールの現在のセットに応じて変更されることがあります。

Okapiバージョン1.11.0以降では、モジュールはバージョンなしで参照されることがあります。 
上記の例では、 `mod-users-bl`を使うことができました。
この場合、最新の利用可能なモジュールが選択され、action = enableになり、
インストールされたモジュールが選択されてaction = disableになります。 
Okapiは、結果として得られる完全なIDで常に応答します。

デフォルトでは、プレリリース版であろうとなかろうと、すべてのモジュールがインストール対象とみなされます。 
Okapi 1.11.0では、ブール値をとる `preRelease`フィルタを追加することができます。 
falseの場合、インストールはプレリリース情報のないモジュールのみを考慮します。

### テナントごとにモジュールをアップデートする

アップグレード機能は、無視されたボディー（空である必要があります）とインストール機能に似た応答を含むPOSTリクエストで構成されます。
呼び出しは `/ _ / proxy / tenant / id / upgrade`というパスを持ちます。
インストール機能と同様に、シミュレートのオプション・パラメータがあり、trueの場合はアップグレードをシミュレートします。
また、 `preRelease`パラメータも認識され、プレリリース情報を持つモジュールIDを考慮すべきかどうかを制御します。

アップグレード機能はOkapiバージョン1.11.0以降の一部です

## 参考

### Okapiプログラム

Okapiプログラムはバンドルされたjar（okapi-core-fat.jar）として出荷されます。
一般的な呼び出しは次のとおりです。

  `java` [*java-options*] `-jar path/okapi-core-fat.jar` *command* [*options*]

これは標準のJavaコマンドラインです。 特に興味深いのは、java-option `-D`です。
プログラムのプロパティを設定することができます。関連するプロパティについては下記を参照してください。
Okapi自体は*command*と続く*options*を解析します。

#### Java -D オプション

`-D`オプションは、Okapiでさまざまな実行時パラメータを指定するために使用できます。 
これらは `-jar`の前のコマンドラインの先頭になければなりません。

* `port`: Okapiがリスンするポート。 デフォルトは9130
* `port_start` and `port_end`: モジュールのポート範囲。 デフォルトは `port` + 1〜` port` + 10、通常は9131〜9141
* `host`: デプロイメントサービスから返されたURLで使用されるホスト名。デフォルトは `localhost`
* `nodename`: このインスタンスのノード名。 システム生成のUUIDの代わりに（クラスタモードで）、または `localhost`（devモードで）の代わりに使うことができます。
* `storage`: ストレージバックエンド、 `postgres`、` mongo`または`inmemory`（デフォルト）を定義します。
* `loglevel`: ロギングレベル。 デフォルトは `INFO`です。 他の有用な値は `DEBUG`、` TRACE`、 `WARN`と` ERROR`です。
* `okapiurl`: Okapiに独自の公式URLを伝えます。 これは、X-Okapi-Urlヘッダーとしてモジュールに渡され、モジュールはこれを使用してOkapiにさらにリクエストします。 デフォルトは `http：// localhost：9130`または指定されたポートです。
後ろにスラッシュはないはずですが、もしあったとしても、オカピはそれを取り除きます。
* `dockerUrl`: Docker DaemonがどこにあるかをOkapiデプロイメントに伝えます。 デフォルトは 
`http://localhost:4243`.
* `postgres_host` : PostgreSQLホスト。 デフォルトは `localhost`.
* `postgres_port` : PostgreSQLポート。デフォルトは 5432.
* `postgres_username` : PostgreSQLユーザー名。 デフォルトは `okapi`.
* `postgres_password`: PostgreSQLパスワード。 デフォルトは `okapi25`.
* `postgres_database`: PostgreSQLデータベース。 Dデフォルトは `okapi`.
* `postgres_db_init`: 値が`1`の場合、Okapiは既存のPostgreSQLデータベースを削除し、新しいデータベースを作成します。 値が`0`（null）の場合は変更されないままになります（デフォルト）。

#### コマンド

Okapiはコマンドをひとつだけ与えられる必要があります。 それらは：
* `cluster` クラスターモード/プロダクションで実行するため
* `dev` 開発環境で実行するため。シングルノード・モード
* `deployment` デプロイメント用のみ。 クラスター・モード
* `proxy` プロキシ + ディスカバリ用。 クラスター・モード
* `help` コマンドラインオプションとコマンドのリストを表示
* `initdatabase` 既存のデータがあればドロップし、データベースを初期化する。
* `purgedatabase` 既存のデータとテーブルをドロップする。

#### コマンドライン・オプション

これらのオプションは、コマンドラインの最後にあります:

* `-hazelcast-config-cp` _file_ -- クラスパスからコンフィグを読む
* `-hazelcast-config-file` _file_ -- ローカルファイルからコンフィグを読む
* `-hazelcast-config-url` _url_ -- URLからコンフィグを読む
* `-enable-metrics` -- さまざまなメトリックをCarbonバックエンドに送信できるようにします。
* `-cluster-host` _ip_ -- Vertx クラスタ　ホスト
* `-cluster-port` _port_ -- Vertx クラスタ　ポート


### 環境変数

Okapiは「環境変数」という概念を提供しています。 
これらはシステム全体のプロパティで、デプロイ中に情報をモジュールに渡す手段を提供します。 
たとえば、データベースにアクセスするモジュールは接続の詳細を知る必要があります。

At deployment the environment variables are defined for the
process to be deployed. Note that those can only be passed to modules
that Okapi manages, e.g. Docker instances and processes. But not
remote services defined by a URL (which are not deployed anyway).

デプロイメント時には、プロセスをデプロイするための環境変数が定義されています。 
これらは、Okapiが管理するモジュールにのみ渡すことができます。 
e.g. ドッカーのインスタンスとプロセス。URLによって定義されたリモートサービスは違います（いずれにしても展開されていない）。

For everything but the deployment-mode, Okapi provides CRU service under
`/_/env`. The identifier for the service is the name of the variable. It
must not include a slash and may not begin with underscore.
An environment entity is defined by
[`EnvEntry.json`](../okapi-core/src/main/raml/EnvEntry.json).

デプロイメント・モードを除くすべての場合、Okapiは `/ _ / env`の下にCRUサービスを提供します。 
サービスの識別子は変数の名前です。 スラッシュを含めることはできず、アンダースコアで始めることはできません。
環境エンティティは[`EnvEntry.json`]（../ okapi-core / src / main / raml / EnvEntry.json）によって定義されます。

### ウェブサービス

Okapiのサービス要求（ `/ _ /`で始まるものすべて）は、[RAML]（http://raml.org/）の構文で指定されています

  * トップレベル・ファイル [okapi.raml](../okapi-core/src/main/raml/okapi.raml)
  * [Directory of RAML and included JSON Schema files](../okapi-core/src/main/raml)
  * [API reference documentation](http://dev.folio.org/doc/api/)はこれらのファイルから生成されます

### 内部モジュール

Okapiが起動すると、内部モジュールが1つ定義されます。 
これは `okapi`と` okapi-proxy`の2つのインターフェースを提供します。 
'okapi'インターフェースは、RAMLで定義されているように（上記を参照）、すべての管理機能をカバーします。
`okapi-proxy`インターフェースはプロキシ機能を参照します。
RAMLで定義することはできません。なぜなら、モジュールが提供するものに依存するからです。 
その主な用途はモジュールがそれに依存することができ、特に新しいプロキシ機能が必要な場合です。 
このインタフェースは、時間の経過と共にかなり安定していると予想されます。

内部モジュールは、オカピバージョン1.9.0で、そして完全に詳細なModuleDescriptorはバージョン1.10.0で導入されました。

### デプロイメント

デプロイメントはスキーマによって指定されています
[DeploymentDescriptor.json](../okapi-core/src/main/raml/DeploymentDescriptor.json)
and [LaunchDescriptor.json](../okapi-core/src/main/raml/LaunchDescriptor.json). 
LaunchDescriptorはModuleDescriptorの一部でも、DeploymentDescriptorで指定することもできます。

モジュールをローンチするには、次の方法があります。

* プロセス: `exec` プロパティはaliveで、Okapi自身によって（シグナルによって）killされるプロセスを指定します。
* コマンド: `cmdlineStart` と `cmdlineStop`プロパティの存在によってトリガーされます。
`cmdlineStart`はシェルスクリプトで、バックグラウンドでサービスを生成して配置します。
`cmdlineStop` はシェルスクリプトで、対応するサービスを終了します。

* ドッカー: `dockerImage`プロパティは、既存のイメージを指定します。
Okapiはこのイメージに基づいてコンテナを管理します。 
このオプションは、 `dockerUrl`がHTTP経由でアクセス可能なDocker Daemonを指すことを要求します。 
デフォルトでは、Okapiはイメージを開始する前にイメージを取得しようとします。 
これはブール値のプロパティ `dockerPull`で変更することができます。これは、プルの発生を防ぐためにfalseに設定することができます。 Dockerfileの `CMD`ディレクティブは` dockerCMD`プロパティで変更することができます。 
これは、 `ENTRYPOINT`がモジュールの完全な呼び出しであり、` CMD`がデフォルト設定であるか、好ましくは空であると見なします。
最後に、プロパティ `dockerArgs`を使用してDocker SDKのcreate-container引数を渡すことができます。 
これは、 `Hostname`、` DomainName`、 `User`、` AttachStdin`などのキーを持つオブジェクトです。例えば、[v1.26 API]（https://docs.docker.com/engine /api/v1.26）をご覧ください。

すべてのデプロイメントタイプに対して、環境変数は `env`プロパティを介して渡されるでしょう。
これは、各環境変数を指定するオブジェクトの配列をとります。 
各オブジェクトは、それぞれ環境変数nameとvalueのプロパティ `name`と` value`を持っています。

モジュールをローンチすると、TCPリスニングポートが割り当てられます。
モジュールは、デプロイが成功した後（HTTPリクエストをサーブする）に、そのポートでリスンするはずです。
ポートはプロパティ `exec`と` cmdlineStart`の値の中に `％p`として渡されます。 
Dockerのデプロイメントの場合、Okapiはエクスポーズされたポート（ `EXPOSE`）を動的に割り当てられたポートにマップします。
It is also possible to refer to an already-launched process (maybe running in your
development IDE), by POSTing a DeploymentDescriptor to `/_/discovery`, with no nodeId
and no LaunchDescriptor, but with the URL where the module is running.

nodeIdとLaunchDescriptorなしで、かつモジュールが実行されているURLとともに
DeploymentDescriptorを `/ _ / discovery`にPOSTすることで、
既に起動されているプロセス（開発IDEで実行中かもしれません）を参照することもできます。

### Docker

Okapiはモジュールをローンチするために [Docker Engine API](https://docs.docker.com/engine/api/) を使用します。
Docker daemonを動かすにはTCP portをリスンしていなくてはなりません。
なぜなら、OkapiはUnixローカルソケットではHTTPを処理しないからです。
Dockerデーモンの有効化は、ホストシステムによって異なります。
システムベースのシステムの場合、 `/lib/systemd/system/docker.service`は調整されなければならず、`ExecStart`行はtcp リスニングホスト+ポートを持つ `-H`オプションを含むべきです。 例） `-H tcp：//127.0.0.1：4243`

```
vi /lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
```

### システムインタフェース

モジュールはシステムインタフェースを提供することができ、Okapiは明確な状況で要求を行うことができます。 
慣例により、これらのインタフェースはアンダースコアで始まる名前を持っています。

現時点では、2つのシステムインタフェースが定義されていますが、
時間がたつにつれてさらにいくつかのインタフェースが増える可能性があります。

#### テナントインタフェース

モジュールが `_tenant`というシステムインタフェースを提供する場合、
Okapiはモジュールがテナントに対して有効になるたびにそのインタフェースを呼び出します。
このリクエストには、新しく有効になったモジュールに関する情報と、
モジュールがアップグレード中の場合などには、同じ時に無効になっているモジュールのオプションの情報が含まれています。
モジュールは、この情報を使用してデータベースをアップグレードまたは初期化し、必要なハウスキーピングを行うことができます。

[詳細](#ウェブサービス)については、`.../okapi/okapi-core/src/main/raml/raml-util`配下のファイル
`ramls/tenant.raml` と `schemas/moduleInfo.schema`を見てください。
okapi-test-moduleはこれを非常に簡単に実装しており、
moduleTestはこのインタフェースを定義するDescriptorモジュールを示しています。

テナント・インタフェースはバージョン1.0で導入されました

#### TenantPermissions Interface

モジュールがテナントに対して有効になると、Okapiは `_tenantPermissions`インターフェースを提供するモジュールを探して呼び出そうとします。
通常、これはパーミッション・モジュールによって提供されるでしょう。 
これは、有効にするモジュールとmoduleDescriptorのすべてのpermissionSetsを含む構造体を取得します。 
これは、パーミッションとパーミッション・セットをパーミッション・モジュールにロードしてユーザに割り当てたり、
他のパーミッション・セットで使用することを目的としています。

パーミッションモジュールを作成している場合を除き、このインタフェースを提供する必要はありません。

このサービスはべき等であるべきです。というのも、モジュールを有効にする際に何か問題が生じた場合、サービスは再び呼び出される可能性があるためです。
まず、指定されたモジュールのすべてのアクセス許可とアクセス許可セットを削除し、
リクエストで受け取ったアクセス許可とアクセス許可セットを挿入する必要があります。 
そうすれば、古いバージョンのモジュールで導入された可能性のあるパーミッションがクリーンアップされ、使用されなくなります。

[詳細](#ウェブサービス)については、 `.../okapi/okapi-core/src/main/raml/raml-util`配下のファイル
`ramls/tenant.raml` and `schemas/moduleInfo.schema`を見てください。
okapi-test-header-moduleはこれを非常に簡単に実装しており、
moduleTestはこのインタフェースを定義するDescriptorモジュールを示しています。

TenantPermissionsインタフェースは、バージョン1.1で導入されました。


### 計装

Okapiは計装データをCarbon / Graphiteバックエンドにプッシュし、
Grafanaのようなもので表示することができます。 
Vert.xはいくつかの数値を自動的にプッシュしますが、Okapiのさまざまな部分が明示的に独自の数値をプッシュするので、
テナントまたはモジュールで分類できます。 個々のモジュールは、必要に応じて独自の番号をプッシュすることもできます。
私たちがOkapiで行っていることに近いキー命名スキームを使用することが期待されています。

`-enable-metrics`を介してメトリックを有効にすると、` localhost：2003`にメトリックを送信し始めます。

 `graphiteHost`をパラメータとしてJavaコマンドに追加した場合、
e.g.
`java -DgraphiteHost=graphite.yourdomain.io -jar okapi-core/target/okapi-core-fat.jar dev -enable-metrics`
メトリクスは `graphite.yourdomain.io`に送られます。

  * `folio.okapi.`_\$HOST_`.proxy.`_\$TENANT_`.`_\$HTTPMETHOD_`.`_\$PATH`_ -- 呼び出しが終了したすべてのモジュールを含む、要求全体の時間
  * `folio.okapi.`_\$HOST_`.proxy.`_\$TENANT_`.module.`_\$SRVCID`_ -- 1回のモジュール呼び出しの時間
  * `folio.okapi.`_\$HOST_`.tenants.count` -- システムに認識されているテナントの数
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.create` -- テナントの作成に関するタイマー
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.update` -- テナントの更新に関するタイマー
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.delete` -- テナントの削除に関するタイマー
  * `folio.okapi.`_\$HOST_`.modules.count` -- システムに認識されているモジュールの数
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.deploy` -- モジュールをデプロイするためのタイマー
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.undeploy` -- モジュールをアンデプロイするためのタイマー
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.update` -- モジュールを更新するためのタイマー

もちろん、変数「 `$` _NAME_」は実際の値を取得します。　

`doc`ディレクトリにGrafanaダッシュボード定義のいくつかの例があります：

* [`grafana-main-dashboard.json`](grafana-main-dashboard.json)
* [`grafana-module-dashboard.json`](grafana-module-dashboard.json)
* [`grafana-node-dashboard.json`](grafana-node-dashboard.json)
* [`grafana-tenant-dashboard.json`](grafana-tenant-dashboard.json)

Grafanaの便利なグラフの例をいくつか紹介します。 
これらは、編集モード（行末のツールメニュー）をテキストモードに変更すると、メトリックの下に直接貼り付けることができます。

  * テナントごとのアクティビティ:

      `aliasByNode(sumSeriesWithWildcards(stacked(folio.okapi.localhost.proxy.*.*.*.m1_rate, 'stacked'), 5, 6), 4)`
  * 毎分のHTTPリクエスト (PUT, POST, DELETE,等もここから)

      `alias(folio.okapi.*.vertx.http.servers.*.*.*.*.get-requests.m1_rate, 'GET')`
  * HTTPリターンコード (4XXと5XXコードもここから)

      `alias(folio.okapi.*.vertx.http.servers.*.*.*.*.responses-2xx.m1_rate, '2XX OK')`
  * 任意のテナントによって呼び出されたモジュール

      `aliasByNode(sumSeriesWithWildcards(folio.okapi.localhost.SOMETENANT.other.*.*.m1_rate, 5),5)`
