# CDK Chatting #6 2025/04/16

## 同じスタックを環境毎に使いまわしたい場合、bin/app.ts で静的/動的に複数作ったり分岐したりするよく見るパターンか、CDK の Stage を活用するパターンがありそうですが、どのようなケースで使い分けると良いのでしょうか？また、CDK の Stage を活用した実装で困った事はありますか？逆もあれば是非聞きたいです。

### スタック作成パターン

同じスタックを環境ごとに静的に作るパターン

```ts
new SampleStack(app, 'DevSampleStack', devProps);
new SampleStack(app, 'StgSampleStack', stgProps);
new SampleStack(app, 'PrdSampleStack', prdProps);
```

同じスタックを環境ごとに動的に作るパターン

```ts
const env: string = app.node.tryGetContext('env') ?? 'dev';
const props = getStackProps(env);
new SampleStack(app, 'SampleStack', props);

// 別ファイル(config.tsやparameter.ts)などで管理
const getStackProps = (env: string): SampleStackProps => {
  switch (env) {
    case 'dev':
      return {
        key: 'dev-key',
        env: {
          account: '111111111111',
          region: 'us-east-1',
        },
      };
    case 'stg':
      return {
        key: 'stg-key',
        env: {
          account: '222222222222',
          region: 'us-east-1',
        },
      };
    case 'prd':
      return {
        key: 'prd-key',
        env: {
          account: '333333333333',
          region: 'us-east-1',
        },
      };
    default:
      throw new Error(`Invalid environment: ${env}`);
  }
};
```

CDK の Stage を活用するパターン

```ts
export class MyStage extends Stage {
  constructor(scope: Construct, id: string, props?: MyStageProps) {
    super(scope, id, props);

    new SampleStack(this, 'SampleStack', props); // 実際はpropsをStackProps用に整形して渡す
  }
}

new MyStage(app, 'DevStage', devProps);
new MyStage(app, 'StgStage', stgProps);
new MyStage(app, 'PrdStage', prdProps);
```

### Stage が向いているケース

個人的には、 **「マルチスタック構成を環境ごとに、かつ静的に作りたい(呼びたい)場合」** にステージは便利です。

もしこのケースでステージがない場合、app.ts ではスタック数 × ステージ数の呼び出しが出てくるので、コードが複雑になって認知負荷が高まります。

```ts
// my-stage.ts
export class MyStage extends Stage {
  constructor(scope: Construct, id: string, props?: MyStageProps) {
    super(scope, id, props);

    new SampleStack1(this, 'SampleStack1', props); // 実際はpropsをStackProps用に整形して渡す
    new SampleStack2(this, 'SampleStack2', props); // 実際はpropsをStackProps用に整形して渡す
    new SampleStack3(this, 'SampleStack3', props); // 実際はpropsをStackProps用に整形して渡す
  }
}

// app.ts
new MyStage(app, 'DevStage', devProps);
new MyStage(app, 'StgStage', stgProps);
new MyStage(app, 'PrdStage', prdProps);
```

一方でシングルスタックの場合、もしくは動的にスタックを作る場合は、自分はステージは使っていません。

ただし静的に作る場合で、今はシングルスタックだけど今後マルチスタックになるかもしれないという予測があったり拡張性を担保しておきたい場合は、あらかじめステージを作っておくと良いと思います。

「ステージを作って損する」ようなケースはあまり思い浮かばない(ファイルが増えるくらい？)ので、であれば最初からステージを作っておくというのはアリかなとも思います。

### 後からステージ導入も OK

ただし、**後からステージを導入することも可能**です。

ステージを使った場合、スタック名は`${stageName}-${スタックID}`(e.g. MyStage-SampleStack)というルールで作られるため、**ただステージを導入するだけだとスタック名が変わってしまいます。** つまり、新規スタック作成処理が走ります(元のスタックに対する削除処理は走りません)。

しかし、StackProps には`stackName`というプロパティがあるため、これを明示的に指定する（今までのスタック名を指定する）と、スタック名に`${stageName}-`というプレフィックスが付くのを防ぐことができ、スタックの新規作成にならず今までのスタックを維持することができます。

```ts
export class MyStage extends Stage {
  constructor(scope: Construct, id: string, props?: MyStageProps) {
    super(scope, id, props);

    new SampleStack(this, 'SampleStack', {
      ...props,
      stackName: 'SampleStack',
    });
  }
}
```

### 後からステージ導入したら既存のリソースが再構築されるんじゃないの？

一般的に、CDK でコンストラクトのパスを変えたとき（あるコンストラクトを別のコンストラクト配下に移したり）、リソースの論理 ID が変わって再構築が走ってしまうことを知っている方もいるかもしれません。

今回のようにスタックをステージの配下にした場合、コンストラクトのパスが変わってリソースが再構築されるんじゃないの？と思われるかもしれません。

しかし、リソースの論理 ID はスタックから下のパスから計算されるため、**論理 ID が変わることはありません**。

ただし CloudFormation テンプレートでの 各リソースの `Metadata` (`"aws:cdk:path": "MyStage/SampleStack/MyTopic/Resource"`)は変わるため、CloudFormation でリソースに対する UPDATE はかかりますが、再構築などはされません。

### CDK の Stage を活用した実装で困った事

実は私は動的にスタックを作る派なので、普段 Stage を使いません。そのため、困ったことはありません。。。なので、あくまで頭の中でのイメージの上でお話します。

まず、ステージを導入していないスタックをステージ配下に移すケースで、もともと何かのリソースで construct のパス(`this.node.path`)やアドレス(`this.node.addr`)(パスから計算されるユニークな値)を使って自前で物理名などを生成している場合などには影響が出てしまうので、思わぬリソース変更が起きてしまうでしょう。

これが起きないようにするためには、そもそもそのような機能を使って自前で(物理名などの)文字列を生成しない、もしくは CDK 内部の論理名生成ロジックと同じようにステージを含めないでパスから計算するロジックを書く、などになるかなと。

### CDK の Stage を活用しなくて困った事

### CDK の Stage を活用して良かった事

### 静的と動的はどちらがいい？

ステージの有無に関わらず、静的と動的どちらが良いかに関しては様々な意見があると思うので、良い悪いというよりあくまでフラットに見たメリットを述べます。（他にもきっとメリットはあるかと思いますが、一旦自分が思いついた限りにて。）

まず、静的が向いているケースです。

- 明確に app の構成/ツリーに環境ごとのスタック情報を含めたいとき
  - = クラウドアセンブリ(`cdk.out`)を活用しているケース
  - 片方のステージだけの synth や deploy をした時でも、全環境の情報がクラウドアセンブリとして生成される
- addValidation を app (`cdk.App`) に適用することで、1 環境の synth や deploy 時でも他の環境にもバリデーションが走るようにしたいとき
  - addValidation でなく Aspects による検査は app には適用されないので注意
  - addValidation や Aspects に関しては、[AWS CDK におけるバリデーションの使い分け方を学ぶ](https://aws.amazon.com/jp/builders-flash/202406/cdk-validation/)を参照
- app ファイルから視覚的にスタック一覧を把握したい、もしくは `cdk list` などでスタック一覧を出したいとき
- `cdk deploy -c ENV=dev`などによる context を使いたくないとき

一方で、動的が向いているケースです。

- Stage を使わずに、app ファイルをシンプルに作りたい
- 単純に dev, stg, prd の 3 種だけでなく、不特定多数のスタックを作成したいとき
  - 例: dev では複数の個人環境などを作りたいとき
    - dev-goto1-stack
    - dev-goto2-stack
    - dev-main-stack
  - ただし静的に作った場合でも、個人 or main の情報だけ context(-c)から渡すことで同じことは可能だが、もはやそれは動的と言える
- `cdk deploy` コマンドでスタック名指定をせずに環境ごとにデプロイしたい場合
  - スタック名が長くて辛い(`-c ENV=dev`の方が短く済む)ケース
