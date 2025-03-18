# CDK Chatting #5 2025/03/18

## cdk cliとaws-cdk-libが分離したことで困りどころはありますか

- ユーザー目線ではまだない、CLIに新しい機能が出てlibでそれに依存する機能が出てきたら、とりあえずCLIを最新に。ただCLIの変更によってはlibのバージョンを上げないといけないケースも稀に出てきそう(今ちょうど自分がCLIでやってるPRで下手したら)
- コントリビュータ目線では、どっちにも変更が必要なケースでは提出先リポジトリが増えて面倒・cli-integ用もできた。ただそういうケースはあまりなく、むしろ分かれたことでビルドやCIが少しだけシンプルになった・衝突しなくなった部分もある(今までもlernaコマンドでスコープは絞れたけど)
- cliでprojen使われるようになった、そういう意味でも今までと開発の感じが変わりそう

## CDKと他のIaCツールを比べたときに、CDKを選択する決め手となるメリットは何だと思いますか？ "インフラを宣言的に定義したい(≒あまり抽象化したくない" と考えている人も少なくない印象を持っているのですが、そんな中でCDKを選択するをお二人目線のメリット伺いたいです

1. 抽象化
    - grantメソッドで最小のIAM権限付与、allowToでセキュリティグループ開けたり
    - ログやバケットポリシーなどの自動生成
    - 推奨デフォルト値による、最低限の指定でも作れる
2. 開発者体験
    - いい感じに入力値を指定しやすくしてくれる表現豊かな型
    - 入力補完
    - Construct + ループで同じリソースの複製とか再利用も簡単
3. バリデーションによる安全性
    - CFnデプロイが走る前に組み込みのバリデーション走ってエラーにしてくれる
    - デプロイ済みのリソースに触れない、ロールバック失敗とかもない
4. AWSサポートに頼れる、日本のCDKコミュニティ層も厚い

### 「宣言的がいい・抽象化したくない」

- CDKも宣言的。「手続的にも書ける」ってだけ。
- 抽象化したくない=L2+エスケープハッチやL1 Constructで、最低限の型・入力補完・バリデーション入れたりなどCDKの恩恵に預かれる
- →これで否定するのももったいない

### CDKのデメリットもある（便利な反面仕方ない、トレードオフ）

- nodejsランタイムがいる
- 開発環境のための環境構築でつまづく
- プログラミング言語で好き放題書ける
- 多少のプログラミング知識がいる(本当に基本的な記法しか使わないし今は生成AIがある)

## CDK のデプロイはどのようにして行なっていますか？CI/CDパイプライン組んでいますか？

- →デプロイはGitLab CI経由、なのでCI/CDパイプライン組んでる。最近はCodePipelineもアツイ。

## L1に設定した引数がCfnに反映されないケースがありました(設定したが無視される)。 エスケープハッチ経由だと正常に設定ができます。どんな理由が考えられるでしょうか？

- →パラメータの大文字小文字が違うなどのtypo。
- CFn: IPv4Xxx → CDK: iPv4Xxx のようになることがある。TypeScriptのsatisfies(記法)とかでうまくやると良いかも？

## stack 分割や construct のグルーピングはどのように設計していますか？どんなリソースをどこまで 1 つの construct や stack にまとめられているか

### stack 分割

- DBなどライフサイクル違うものはスタック分けるんですが、そもそもCDKコードの管理リポジトリを別にしていて、基本リポジトリが違うとスタックが違うようになるので、結果的に分かれる
- 稀に個人ごとにスタックデプロイするような開発環境でも、KMSキーなどは個人スタック間で使い回すしてアプリは個人ごとに作りたいとかだと、KMSとかの共有リソースだけスタック分けておくみたいなのはやる
- リージョン分ける必要あるもの（CloudFront・WAF的な）みたいな仕方がないもの以外はできるだけシングルスタック
- アプリ系じゃなくて組織管理系のもので、まとめて1つの管理リポジトリとして管理しているけど、スタックわけたりはある
　- →ACM、Route53ホストゾーン、スタックセット系、コスト管理系など

### construct のグルーピング

- →ユースケース/AWSサービスがあると思うが、判断のフローがある。まずはユースケース単位。
  - xx-api (API GW+Lambda, ELB+ECS)
  - spa-hosting
- →ただしユースケースを切り分けていく中で再利用したいもの（Lambda）とかは別constructに切り出す
- →つまり最初はユースケースベースで考える
  - →再利用（Lambda）や、これはパラメータ数多くてファイル分けて管理したいってなったらサービスベースで分ける(WAFとか)
  - →「これはサービスベースかなあ」みたいな最初から思っているのは最初から分ける(Athena.DatabaseとかSecretsとか)

## ディレクトリ構造で工夫していること。stack, construct, aspect, … 等、ファイルが増えてきた時の整理の仕方

→全プロジェクトでそうしているわけじゃなくて、複雑なリソースのPJではこうしている。

```txt
- lib
  - aspect
    - xx.ts
  - construct
    - fargate.ts
    - waf.ts
    - ...
  - stack
    - stack-a
      - index.ts (stack-a.tsだけexport)
      - stack-a.ts
      - usecase
        - static.ts
        - app.ts
        - monitoring.ts
    - stack-b
      - ...
```

- Aspect(もしくはutil)、Stack、Constructで分けて、stack内でusecaseディレクトリ作る。
- Usecaseはスタック固有の再利用性を考えないけどファイルは分けたいConstruct。Constructディレクトリはstack, usecase跨ぎで使う再利用のためのやつ。
- 補足
  - Stack内だけで使いまわしたいConstructはStackディレクトリ内にconstructディレクトリ作ってもいいが、「スタック内だけで」というprefixがつくということはユースケース特有な情報が入っていると思うので、それはusecaseになるか、よりConstructを汎用的な作りにしてスタック跨ぎで使えるようにしてConstructディレクトリに移してもいいかも。
  - うちでは使いまわしたいConstruct系は別の共通リポジトリとして切り出している(※結局リポジトリ間で再利用ケースが多くなり、リポジトリ分けてプライベートパッケージとして公開/npm配布するようにした)

## Construct ID に使用する単語はどのように決めていますか？ルール等を定めていたら知りたいです

- 基本Construct名と同じ(そのユースケースの名前とかサービス名とか)
  - Lambdaを使い回すためのLambda Constructみたいなときはユースケースの名前(ImageRegister、MovieRegister)
  - XxxApiみたいなConstructであれば同じように「XxxApi」とかにする
- xxStack, xxConstructというsuffixはしない（論理IDが長くなる）
- L2っぽい単一リソースのConstructを作る場合は、そのメインのリソースは”Resource”または”Default”にしたり(論理IDが短くなる)

## CDK コード内で使用したい「CDK コード外で発生したシークレット情報」の管理・使用方法にオススメはありますか？現在は AWS CLI を使用して Parameter Store に格納し、CDK 側で from メソッドで import して使用しています

- リソース自体はCDKスタック内で空バリューで作って初回デプロイ後に手動格納 (リソース作成は全部IaC=CDKでやりたいから)
- 「CDKコード外で作成」というのが前提にあるのであれば、from〜がいいかなと

## CDKのインフラ構成図を作成するツールで何か使っていたりしますか？自分はCDK-DiaがしっくりこずAWS PDKのプラグインを使っていますが細かい設定ができないので自作しようか悩んでいます。最近ではAmazon Q CLIで生成できる噂を聞いたのでいらないかもと思いつつ

- Draw.ioでCDK関係ない感じで書いている。全然ここら辺のエコシステム知らない。
- AWS Infrastructure Composer (以前は [Application Composer] と呼ばれていた) でCloudFormationテンプレートから図を出してくれるやつもあり？
  - が、細かいリソース(IAMとか)は図に入れたくなかったりするので、手動で図を書いて抽象化もありかなと思っている（でもメンテしなくなるのだけ注意）
- Amazon Q CLI気になる

## 環境変数など値のバリデーションはスタックで行いますか？アプリで行いますか？ 自分はアプリ側でバリエーションライブラリを入れていますが、CDKのテストの記事などスタック側で設定してる例をたまに見るので何か明確な理由があれば知りたいです

→スタック(props)でもアプリ(Lambda・ECSコードとか)でも行っている。

- スタックに渡すpropsはCDKコードで、アプリで使う環境変数はアプリ側でやっている。
- アプリとしての環境変数のバリデーションは必須だと思うが、どっちでも行った方が安全。

スタック側でもやる理由としては、スタックとアプリで入力群は違う（スタックのpropsの中にアプリではいらないものもあれば逆もある）中で、スタックの入力(props)はスタック単位でまとめてバリデーションしたいから。

スタックの入力(props)を加工してアプリ側で使うケースもあるので、どちらで行ったからどちらで行わなくていいというわけでもない。

※ただし「LambdaやECSの環境変数」自体をスタック側でバリデーションはあまりしないかも。あくまでCDKで行うのは「propsのバリデーション」。ただLambda・ECSという粒度でConstructを作っている場合は、Constructの再利用性を考えてConstructの中でバリデーションすることもあるかも。

## 以前CDKのBootstrap時に作成されるデフォルトバケットが命名がわかりやすくて脆弱性として取り上げられていたことがあったかと思うのですがその問題は解決していますか？また、CDKで使うS3バケットはデフォルトを使わず別途作成したものを明示的に指定したほうがよいでしょうか？

→2.149.0（bootstrapバージョン21）で、解決しています。

IAMポリシー(FilePublishingRoleDefaultPolicy)の制限(自分のアカウントのみからのアップロードに絞ることで、攻撃手法である「攻撃者がすでに該当アカウントのバケット名前で作ったS3バケットへのアップロード」を防止。)

デフォルトで問題はないが、ここは組織のポリシーに合わせて。（バケット名はわからないに越したことはないので。CDK操作の手間は増えるが。）

ref: https://github.com/aws/aws-cdk/pull/30823/files

## 初心者質問ですが、cliとcdk-libが分離しましたが、cdk-libを最新で使いたい場合はどうするのが良いのでしょうか。cdk initでpjを生成した際にcdk-libが最新にならないことがあり、initしてから一旦バージョンを上げる手順を挟んでいます。インストールしたのは分離前なのでそれのせいかもですが

→その時点のcdk-cliに紐づいているlibバージョンがインストールされるので、新規PJをcdk initしてもlibは最新にならないことがあるので、init後に手動でバージョンを上げるしかなさそう。

## CDKのパイプラインとアプリのパイプラインは分けてますか？ライフサイクルの違いにより分けたいと考えてますが、分離にもテクニックがいると感じてます

→自分は分けていない。CDKのdeployでビルドも行っている、ライフサイクルを一緒にできるのはCDKの良さでもある。

アプリ・インフラでチームが違うとかリポジトリが分かれているとかでない限り、無理にわけなくてもいいかも？

明確な分けたい理由に遭遇したら分けるのを考えるとかもあり。

### parameter.tsのようなファイルにパラメータまとめると思いますが、設定したいシークレットがたくさんある場合はどうしてますか？

Secrets用Constructを作って、そこでまとめて管理しています。

parameter.tsのようなファイルでは、それ用のinterfaceを切ったり。

### aws_s3_notificationsでaspectがデフォルトで500になってるようなんですがaspectのタグの上書きってできるのでしょうか？

(「aws_s3_notificationsで」というのに対して回答として沿っているかわからないのですが)、Aspect/Tagsのメソッドとしてはpriorityの上書きはできます！

```ts
export class Aspects {
  // ...
  // ...
  public add(aspect: IAspect, options?: AspectOptions)
}

/**
 * Options when Applying an Aspect.
 */
export interface AspectOptions {
  /**
   * The priority value to apply on an Aspect.
   * Priority must be a non-negative integer.
   *
   * Aspects that have same priority value are not guaranteed to be
   * executed in a consistent order.
   *
   * @default AspectPriority.DEFAULT
   */
  readonly priority?: number;
}
```

```ts
/**
 * Properties for a tag
 */
export interface TagProps {
  // ...
  // ...
  /**
   * Priority of the tag operation
   *
   * Higher or equal priority tags will take precedence.
   *
   * Setting priority will enable the user to control tags when they need to not
   * follow the default precedence pattern of last applied and closest to the
   * construct in the tree.
   *
   * @default
   *
   * Default priorities:
   *
   * - 100 for `SetTag`
   * - 200 for `RemoveTag`
   * - 50 for tags added directly to CloudFormation resources
   *
   */
  readonly priority?: number;
}
```

## CDKを使うPJでは、インフラのパラメータシートは作成しない（必要がない）ですよね？パラメータシートが必要な文化・お客様だと、CDK（IaC）がマッチしにくい印象です。資料とコードの二重管理という点もですが、CDKがいい感じに設定してくれた設定値をわざわざ確認して資料に落とし込む…？

